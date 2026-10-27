# LAB 03 — Clear Command via `write()`

**Difficulty:** ★★  
**Target Architecture:** ARMv7  
**Target Kernel:** Linux 6.1.90  
**Emulator:** QEMU `vexpress-a9` / Cortex-A9  
**Base Driver:** LAB-02 Character Device Driver

---

## 1. Objective

The objective of this exercise is to extend the existing `lab2_driver.c` so that the character device can recognize a special command through its `write()` interface.

When userspace writes the exact 5-byte string:

```text
clear
```

to:

```text
/dev/lab2
```

the driver must clear its internal shared buffer instead of storing the word `clear`.

All other input must continue to behave as normal data.

---

## 2. Functional Requirements

The modified driver must satisfy the following behavior.

| Input | Expected Behavior |
|---|---|
| `HELLO` | Store normally |
| `clear` | Clear the internal buffer |
| `CLEAR` | Store normally |
| `clear\n` | Store normally |
| `clear123` | Store normally |

The command matching rule is intentionally strict.

The clear command is valid only when:

```text
length == 5
```

and the five bytes are exactly:

```text
c l e a r
```

Therefore:

```text
"clear"      -> command
"CLEAR"      -> normal data
"clear\n"    -> normal data
"clear123"   -> normal data
```

After a successful clear command:

```text
buffer_len = 0
device_buffer[0] = '\0'
```

A subsequent read from `/dev/lab2` must return EOF.

---

## 3. Baseline Driver Analysis

The original LAB-02 driver stores all data written through `/dev/lab2` into a shared character buffer.

The important global state is:

```c
static char device_buffer[BUFFER_SIZE];
static size_t buffer_len;
```

The baseline `lab2_write()` follows this basic flow:

```text
userspace write()
      |
      v
lab2_write()
      |
      v
check len == 0
      |
      v
calculate to_copy
      |
      v
lock lab2_mutex
      |
      v
copy_from_user()
      |
      v
buffer_len = to_copy
      |
      v
append '\0'
      |
      v
unlock mutex
      |
      v
return number of bytes written
```

The original implementation does not interpret the data.

For example:

```sh
echo -n "clear" > /dev/lab2
```

would normally result in:

```text
device_buffer = "clear"
buffer_len = 5
```

LAB 03 changes this behavior by interpreting one exact input value as a control command.

---

## 4. Design Reasoning

The clear command is implemented inside `lab2_write()` because the command is received through the device's normal write interface.

The driver must first copy the userspace data into kernel memory using:

```c
copy_from_user()
```

Directly dereferencing the userspace pointer is not valid kernel-driver practice.

After a successful copy, the driver checks:

```c
to_copy == 5
```

and compares the five bytes using:

```c
memcmp(device_buffer, "clear", 5)
```

The full condition is:

```c
if (to_copy == 5 &&
    memcmp(device_buffer, "clear", 5) == 0)
```

Using both the length check and `memcmp()` prevents partial or extended strings from being incorrectly interpreted as commands.

For example:

```text
clear123
```

starts with the characters `clear`, but its length is 8, so it must not trigger the clear operation.

Similarly:

```text
clear\n
```

contains six bytes and must be treated as normal data.

---

## 5. Implementation

### 5.1. String Support Header

The driver includes:

```c
#include <linux/string.h>
```

for the `memcmp()` operation.

---

### 5.2. Modified `lab2_write()`

After successfully copying data from userspace, the driver checks whether the input is the exact clear command.

```c
if (copy_from_user(device_buffer, buf, to_copy)) {
    mutex_unlock(&lab2_mutex);
    return -EFAULT;
}

/*
 * LAB03:
 * The exact 5-byte command "clear" clears the shared buffer.
 * Other input is stored normally.
 */
if (to_copy == 5 &&
    memcmp(device_buffer, "clear", 5) == 0) {

    buffer_len = 0;
    device_buffer[0] = '\0';

    mutex_unlock(&lab2_mutex);

    pr_info("lab2_driver: buffer cleared\n");
    return to_copy;
}

buffer_len = to_copy;
device_buffer[buffer_len] = '\0';

mutex_unlock(&lab2_mutex);

pr_info("lab2_driver: received %zu bytes\n", to_copy);
return to_copy;
```

---

## 6. Why the Command Still Returns 5

When the command:

```text
clear
```

is accepted, the driver returns:

```c
return to_copy;
```

where:

```text
to_copy = 5
```

This means the userspace write operation successfully consumed all five bytes.

The data is interpreted as a command rather than stored as ordinary buffer contents.

Returning 5 therefore correctly reports that all input bytes were successfully processed.

---

## 7. Synchronization

The existing driver mutex:

```c
static DEFINE_MUTEX(lab2_mutex);
```

continues to protect the shared state.

The clear operation occurs while the mutex is held:

```text
mutex_lock
     |
     v
copy_from_user
     |
     v
command comparison
     |
     v
buffer_len = 0
device_buffer[0] = '\0'
     |
     v
mutex_unlock
```

This prevents another reader or writer from observing a partially modified buffer.

---

## 8. Development Environment

The exercise was developed and tested with:

| Component | Configuration |
|---|---|
| Host OS | Ubuntu Linux |
| Target architecture | ARMv7 |
| Target machine | QEMU `vexpress-a9` |
| Target CPU | Cortex-A9 |
| Target kernel | Linux 6.1.90 |
| Cross compiler | `arm-linux-gnueabihf-` |
| Device node | `/dev/lab2` |
| Major number | 240 |
| Minor number | 0 |
| Buffer size | 1024 bytes |

Exercise workspace:

```text
~/embedded_linux_driver_exercises/exercises/lab03_clear_command/
```

---

## 9. Build Instructions

Enter the exercise directory:

```bash
cd ~/embedded_linux_driver_exercises/exercises/lab03_clear_command
```

Clean previous build artifacts:

```bash
make clean
```

Build the kernel module:

```bash
make
```

Verify the generated module:

```bash
file lab2_driver.ko
```

The module must be an ARM relocatable ELF object.

Verify kernel compatibility:

```bash
modinfo lab2_driver.ko | grep vermagic
```

Expected kernel version:

```text
6.1.90
```

---

## 10. Prepare the Initramfs

Copy the new module into the shared root filesystem:

```bash
cp ~/embedded_linux_driver_exercises/exercises/lab03_clear_command/lab2_driver.ko \
~/embedded_linux_driver_exercises/rootfs/initramfs/lib/modules/6.1.90/
```

Enter the initramfs directory:

```bash
cd ~/embedded_linux_driver_exercises/rootfs/initramfs
```

Repack the filesystem:

```bash
find . -print0 | \
cpio --null -o --format=newc | \
gzip -9 > ~/embedded_linux_driver_exercises/output/initramfs_lab03.cpio.gz
```

Verify the archive:

```bash
ls -lh ~/embedded_linux_driver_exercises/output/initramfs_lab03.cpio.gz
```

---

## 11. QEMU Boot

Boot the ARM system using:

```bash
qemu-system-arm \
    -M vexpress-a9 \
    -cpu cortex-a9 \
    -m 512M \
    -smp 2 \
    -nographic \
    -kernel ~/embedded_lab1/output/zImage \
    -dtb ~/embedded_lab1/output/vexpress-v2p-ca9.dtb \
    -initrd ~/embedded_linux_driver_exercises/output/initramfs_lab03.cpio.gz \
    -append "console=ttyAMA0,115200 rdinit=/sbin/init"
```

---

## 12. Driver Setup Inside QEMU

Check loaded modules:

```sh
lsmod
```

Load the driver:

```sh
insmod /lib/modules/6.1.90/lab2_driver.ko
```

Observed kernel messages:

```text
lab2_driver: loading out-of-tree module taints kernel.
lab2_driver: initializing module
lab2_driver: loaded, major=240
```

Verify:

```sh
lsmod
```

Observed:

```text
lab2_driver 16384 0 - Live ... (O)
```

Create the device node:

```sh
mknod /dev/lab2 c 240 0
```

---

# 13. Functional Testing

## Test 01 — Normal Data Write

Write ordinary data:

```sh
echo -n "HELLO" > /dev/lab2
```

Observed driver output:

```text
lab2_driver: device opened (count=1)
lab2_driver: received 5 bytes
lab2_driver: device closed
```

Read the device:

```sh
cat /dev/lab2
```

Observed:

```text
lab2_driver: device opened (count=2)
lab2_driver: sent 5 bytes to user
HELLO
lab2_driver: device closed
```

Inspect procfs:

```sh
cat /proc/lab2_info
```

Observed important fields:

```text
Data length   : 5 bytes
Open count    : 2
Last data     : [HELLO]
```

### Analysis

The normal write path remains unchanged.

The five input bytes were stored in `device_buffer`, and:

```text
buffer_len = 5
```

The data was read back correctly.

**Result: PASS**

---

## Test 02 — Exact `clear` Command

Execute:

```sh
echo -n "clear" > /dev/lab2
```

Observed:

```text
lab2_driver: device opened (count=3)
lab2_driver: buffer cleared
lab2_driver: device closed
```

Check procfs:

```sh
cat /proc/lab2_info
```

Observed:

```text
Data length   : 0 bytes
Open count    : 3
```

No `Last data` entry was displayed.

### Analysis

The input length was exactly five bytes and:

```c
memcmp(device_buffer, "clear", 5) == 0
```

Therefore the clear-command branch executed:

```c
buffer_len = 0;
device_buffer[0] = '\0';
```

**Result: PASS**

---

## Test 03 — Read After Clear

Execute:

```sh
cat /dev/lab2
```

Observed:

```text
lab2_driver: device opened (count=4)
lab2_driver: device closed
```

No data was returned.

Check the sysfs buffer length:

```sh
cat /sys/class/lab2_class/lab2/buffer_len
```

Observed:

```text
0
```

### Analysis

Because:

```text
buffer_len = 0
```

the read function immediately reached the EOF condition:

```c
if ((size_t)*offset >= buffer_len)
    return 0;
```

The clear command therefore removed the previously stored data successfully.

**Result: PASS**

---

## Test 04 — Uppercase `CLEAR`

Execute:

```sh
echo -n "CLEAR" > /dev/lab2
```

Observed:

```text
lab2_driver: device opened (count=5)
lab2_driver: received 5 bytes
lab2_driver: device closed
```

Read the device:

```sh
cat /dev/lab2
```

Observed:

```text
CLEAR
```

Check the buffer length:

```sh
cat /sys/class/lab2_class/lab2/buffer_len
```

Observed:

```text
5
```

### Analysis

Although the input length is five bytes, `memcmp()` is case-sensitive.

Therefore:

```text
CLEAR != clear
```

and the command branch was not executed.

The data was stored normally.

**Result: PASS**

---

## Test 05 — `clear` with Newline

Execute:

```sh
echo "clear" > /dev/lab2
```

Unlike `echo -n`, the normal `echo` command appends a newline.

The actual bytes are therefore:

```text
c l e a r \n
```

for a total length of six bytes.

Observed:

```text
lab2_driver: device opened (count=7)
lab2_driver: received 6 bytes
lab2_driver: device closed
```

Check the buffer length:

```sh
cat /sys/class/lab2_class/lab2/buffer_len
```

Observed:

```text
6
```

Read the data:

```sh
cat /dev/lab2
```

Observed:

```text
clear
```

followed by the stored newline.

### Analysis

The clear-command condition requires:

```c
to_copy == 5
```

but the received input contained six bytes.

Therefore the command was not recognized and the input was stored normally.

**Result: PASS**

---

## Test 06 — Prefix Match `clear123`

Execute:

```sh
echo -n "clear123" > /dev/lab2
```

Observed:

```text
lab2_driver: device opened (count=9)
lab2_driver: received 8 bytes
lab2_driver: device closed
```

Check buffer length:

```sh
cat /sys/class/lab2_class/lab2/buffer_len
```

Observed:

```text
8
```

Read the device:

```sh
cat /dev/lab2
```

Observed:

```text
lab2_driver: device opened (count=10)
lab2_driver: sent 8 bytes to user
clear123
lab2_driver: device closed
```

Check procfs:

```sh
cat /proc/lab2_info
```

Observed:

```text
LAB-02 Driver Statistics
======================
Driver name   : lab2
Major number  : 240
Buffer size   : 1024 bytes
Data length   : 8 bytes
Open count    : 10
Last data     : [clear123]
```

### Analysis

The string begins with `clear`, but its total length is eight bytes.

The command condition requires exactly five bytes:

```c
to_copy == 5
```

so `clear123` was correctly treated as normal data.

This test confirms that the implementation does not accidentally perform prefix-based command matching.

**Result: PASS**

---

# 14. Test Summary

| Test | Description | Result |
|---|---|---|
| T01 | Normal write/read | PASS |
| T02 | Exact `clear` command | PASS |
| T03 | EOF after clear | PASS |
| T04 | Uppercase `CLEAR` rejection | PASS |
| T05 | `clear\n` rejection | PASS |
| T06 | `clear123` rejection | PASS |

**Overall Result: PASS — 6/6 functional tests**

---

## 15. Key Learning Points

This exercise demonstrates several important Linux-driver concepts.

### Exact command parsing

Commands received through a device interface should have a clearly defined binary or textual format.

Using both length and content checks prevents ambiguous matches.

### Userspace memory handling

Userspace data must first be transferred using:

```c
copy_from_user()
```

before kernel code processes it.

### Shared state synchronization

Modifications to:

```text
device_buffer
buffer_len
```

must remain protected by the driver mutex.

### Data interface vs control interface

The same `write()` interface now accepts two logical categories:

```text
normal input -> data
"clear"      -> control command
```

This is a simple example of command parsing in kernel drivers.

Later exercises use more structured control mechanisms such as:

```text
sysfs
ioctl
```

---

## 16. Limitation of This Exercise

Only the exact 5-byte lowercase command:

```text
clear
```

is recognized.

In particular:

```text
echo "clear" > /dev/lab2
```

does not clear the buffer because `echo` appends a newline.

This behavior is intentional for LAB 03.

Handling:

```text
clear\n
```

as a valid command is addressed separately in a later exercise.

---

## 17. Files to Keep

The exercise source directory should contain:

```text
lab03_clear_command/
├── Makefile
├── README.md
└── lab2_driver.c
```

Generated kernel build artifacts should not be committed to Git.

Examples include:

```text
*.ko
*.o
*.cmd
*.mod
*.mod.c
Module.symvers
modules.order
```

---

## 18. Git Submission

After successful testing, copy the verified source files to the Git repository.

Example destination:

```text
assignments/driver-exercises/lab03_clear_command/
```

Then stage and commit the implementation separately from documentation when appropriate.

Suggested implementation commit:

```bash
git commit -m "feat(lab03): add clear command to character device"
```

Suggested documentation commit:

```bash
git commit -m "docs(lab03): document clear command and verified QEMU tests"
```

---

## 19. Final Result

LAB 03 successfully extends the baseline character driver with a textual control command while preserving normal read/write functionality.

The exact command:

```text
clear
```

clears the driver's shared buffer.

Inputs that merely resemble the command remain ordinary device data.

The implementation was verified on ARMv7 Linux 6.1.90 under QEMU.

**Final Status: PASS — 6/6**