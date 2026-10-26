# LAB 01 — Read Operation Counter

**Difficulty:** ★★  
**Target:** ARMv7 / Linux Kernel 6.1.90  
**Base Driver:** LAB-02 Character Device Driver

## 1. Objective

Extend the existing `lab2_driver.c` to count read operations and expose the counter through a read-only sysfs attribute.

## 2. Implementation

The driver introduces:

```c
static unsigned int read_count;
```

The counter is incremented inside `lab2_read()` after successfully acquiring `lab2_mutex`:

```c
read_count++;
```

The counter includes successful read operations and reads returning EOF.

A new read-only sysfs attribute is registered:

```text
/sys/class/lab2_class/lab2/read_count
```

The attribute uses `DEVICE_ATTR_RO(read_count)` and `sysfs_emit()` to expose the current counter value.

## 3. Build

```bash
make clean
make
```

Verify the generated module:

```bash
file lab2_driver.ko
modinfo lab2_driver.ko | grep vermagic
```

## 4. Testing

The driver was tested using QEMU with ARM Cortex-A9 and Linux 6.1.90.

Load the module:

```sh
mknod /dev/lab2 c 240 0
insmod /lib/modules/6.1.90/lab2_driver.ko
```

Check the initial counter:

```sh
cat /sys/class/lab2_class/lab2/read_count
```

Expected output: `0`

Write and read data:

```sh
echo -n "HELLO" > /dev/lab2
cat /dev/lab2
cat /sys/class/lab2_class/lab2/read_count
```

Observed output:

```text
HELLO
2
```

A second `cat /dev/lab2` increased the counter to `4`.

Writing `FPGA` to the device did not change the counter.

Writing to the read-only sysfs attribute returned `Permission denied`.

After unloading and reloading the module, the counter returned to `0`.

Reading an empty device increased the counter from `0` to `1`.

## 5. Test Results

| Test | Description | Result |
|---|---|---|
| T01 | Module loading | PASS |
| T02 | Initial counter value | PASS |
| T03 | Read operation counting | PASS |
| T04 | EOF counting | PASS |
| T05 | Read-only sysfs protection | PASS |
| T06 | Write operation independence | PASS |
| T07 | Empty-buffer read counting | PASS |

**Result: PASS — 7/7 functional tests.**