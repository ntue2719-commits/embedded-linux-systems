# Embedded Linux Systems

Collection of labs, assignments, and Linux driver exercises for the **Embedded Linux Systems** course.

Main topics:

- ARM Linux with QEMU
- Linux kernel modules
- Character device drivers
- procfs / sysfs
- ioctl
- BusyBox initramfs
- MTD / JFFS2
- Driver synchronization and debugging

---

## Development Environment

| Component | Configuration |
|---|---|
| Host | Ubuntu 24.04.5 LTS x86_64 |
| Target | ARMv7 Cortex-A9 |
| Emulator | QEMU `vexpress-a9` |
| Kernel | Linux `6.1.90` |
| Cross Compiler | `arm-linux-gnueabihf-` GCC 13.3.0 |
| RootFS | BusyBox 1.35.0 + initramfs |

---

## Repository Structure

```text
embedded-linux-systems/
├── labs/
│   ├── lab01/
│   └── lab02/
│
├── assignments/
│   ├── assignment01/
│   ├── assignment02/
│   │   ├── README.md
│   │   ├── driver/
│   │   │   ├── Makefile
│   │   │   ├── sms_sensor.h
│   │   │   └── sms_sensor_driver.c
│   │   └── tests/
│   │       └── test_ioctl.c
│   │
│   └── driver-exercises/
│       ├── lab21_sysfs_device_name/
│       ├── lab22_sysfs_buffer_capacity/
│       ├── lab23_open_failure_counter/
│       ├── lab24_last_read_size/
│       ├── lab25_ascii_validation/
│       ├── lab26_info_command/
│       ├── lab27_sysfs_is_open/
│       └── lab28_open_limit/
│
└── resources/
```

---

## Progress

| Work | Status |
|---|---|
| LAB-01 | **PASS — ARM Linux direct QEMU boot** |
| LAB-02 | **PASS — Parts A–D completed** |
| Driver Exercises 21–28 | **8/8 PASS** |
| Driver Exercises 29–40 | Pending |
| Assignment-01 | In progress / coursework |
| Assignment-02 Kernel Driver | **PASS — baseline completed** |
| Assignment-02 Userspace App | Next |

---

# LAB-01

Verified direct ARM Linux boot using:

```text
QEMU
 ├── zImage
 ├── vexpress-v2p-ca9.dtb
 └── BusyBox initramfs
```

Target kernel:

```text
Linux 6.1.90 ARMv7
```

---

# LAB-02

Completed:

- character driver `/dev/lab2`
- `open/read/write/release`
- mutex protection
- `/proc/lab2_info`
- sysfs attributes
- BusyBox module auto-load
- MTD / JFFS2 testing

Baseline:

```text
Module      : lab2_driver.ko
Device      : /dev/lab2
Major/minor : 240 / 0
Buffer      : 1024 bytes
```

---

# Driver Exercises

Each exercise is an independent modification of the LAB-02 baseline.

Completed:

```text
Lab 21  device_name sysfs             PASS
Lab 22  buffer_capacity sysfs          PASS
Lab 23  exclusive open                PASS
Lab 24  last_read_size                PASS
Lab 25  ASCII validation              PASS
Lab 26  info command                  PASS
Lab 27  is_open sysfs                 PASS
Lab 28  maximum two active opens      PASS
```

Labs 29–40 are pending.

---

# Assignment-02 — Sensor Monitoring System

Current completed baseline:

```text
D1  Module init/exit                   PASS
D2  Character device                  PASS
D3  Sensor CSV read                   PASS
D4  /proc/sms_stats                   PASS
D5  SMS_SET_INTERVAL ioctl            PASS
```

Implemented driver features:

- `/dev/sms_sensor`
- dynamic major number
- random temperature `15–45 °C`
- random humidity `30–90 %`
- monotonic timestamp
- CSV sensor output
- partial-read support
- `file->private_data`
- `/proc/sms_stats`
- `read_count`
- last temperature / humidity
- driver uptime
- mutex synchronization
- default sampling interval `100 ms`
- `SMS_SET_INTERVAL`
- `-EAGAIN` when reading too early
- ARM userspace ioctl test

Example sensor output:

```text
61719,34.95,62.75
```

Example procfs output:

```text
read_count: 4
last_temperature: 42.47
last_humidity: 30.69
uptime_seconds: 89
```

---

## Assignment-02 Build

```bash
cd ~/embedded_asm2/driver
make clean
make
```

Verify:

```bash
file sms_sensor_driver.ko
modinfo sms_sensor_driver.ko | grep vermagic
```

Expected target:

```text
ARM 32-bit
Linux 6.1.90 ARMv7
```

---

## Assignment-02 QEMU Boot

```bash
qemu-system-arm \
  -M vexpress-a9 \
  -cpu cortex-a9 \
  -m 512M \
  -smp 2 \
  -nographic \
  -kernel ~/embedded_lab1/output/zImage \
  -dtb ~/embedded_lab1/output/vexpress-v2p-ca9.dtb \
  -initrd ~/embedded_asm2/output/initramfs_asgn2.cpio.gz \
  -append "console=ttyAMA0,115200 rdinit=/sbin/init"
```

Inside QEMU:

```bash
insmod /lib/modules/6.1.90/sms_sensor_driver.ko
grep sms_sensor /proc/devices
```

Create the device using the returned major:

```bash
mknod /dev/sms_sensor c <major> 0
chmod 666 /dev/sms_sensor
```

Test:

```bash
cat /dev/sms_sensor
cat /proc/sms_stats
test_ioctl
```

Cleanup:

```bash
rmmod sms_sensor_driver
rm -f /dev/sms_sensor
```

---

## Assignment-02 Next Work

Next userspace components:

```text
Circular Buffer
Sensor Thread
Processor Thread
Alert Thread
Logger Thread
POSIX message queues
CLI options
SIGINT shutdown
Valgrind / GDB / strace / perf
SCHED_FIFO and jitter analysis
```

Detailed Assignment-02 documentation is available in:

```text
assignments/assignment02/README.md
```

---

## Git Workflow

```bash
cd ~/Github_embedded_system/embedded-linux-systems

git status
git add <file>
git diff --cached --name-only
git commit -m "<type>: <message>"
git push origin main
```

Commit convention:

```text
feat     new feature
fix      bug fix
docs     documentation
test     tests
build    Makefile/build
refactor code restructuring
chore    maintenance
```

---

## Notes

- Do not load ARM `.ko` modules into the x86_64 host kernel.
- Do not commit unnecessary build artifacts such as `.ko`, `.o`, `.cmd`, `Module.symvers`, or generated initramfs files.
- Keep completed baselines unchanged.
- Test changes in QEMU before committing them.

---

## Author

**Nguyen Tri Tue**  
FPT University  
Embedded Linux Systems