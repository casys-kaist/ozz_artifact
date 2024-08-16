# Measuring the Performance Overhead of OEMU (Table 5)

In this section, we measure the performance overhead of OEMU by comparing kernels with and without OEMU instrumentation.
We used the following programs for measurement.

| Program | Description | Repository Link | Commit |
| --- | --- | --- | --- |
| Kernel 1 (OEMU) | OEMU-instrumented kernel | [link](https://github.com/casys-kaist/ozz_kernel) | [ab4e73d](https://github.com/casys-kaist/ozz_kernel/commits/master) |
| Kernel 2 (Baseline) | Non-instrumented kernel | [link](https://github.com/casys-kaist/ozz_kernel) | [480e035](https://github.com/casys-kaist/ozz_kernel/commits/480e035fc4c714fb5536e64ab9db04fedc89e910) |
| LMBench | Benchmark for overhead measurement |[link](https://github.com/intel/lmbench) | [701c6c3](https://github.com/intel/lmbench/commit/701c6c35b0270d4634fb1dc5272721340322b8ed) |

As mentioned, we prepared both versions of complied kernels and LMBench source code with patches applied on the provided server. So one can skip to [Running LMBench](#running-lmbench), or follow the instructions in [Building Baseline Kernel](#building-baseline-kernel) and [Applying Patch for LMBench](#applying-patch-for-lmbench) to build them.

Before running any commands, please ensure that you have set up the environment variables:
```sh
# In project home directory
source ./scripts/envsetup.sh
```

## Building Baseline Kernel

The OEMU-instrumented kernel is already built [here](README.md#installing-ozz).

```sh
# There will be two directories like this:
→ ls $PROJECT_HOME/kernels/guest/builds -l
x86_64 -> ozz/kernels/guest/builds/x86_64-HEAD
x86_64-HEAD     # OEMU-instrumented kernel
```

To avoid overwriting, rename this directory:
```sh
cd $PROJECT_HOME/kernels/guest/builds
mv x86_64-HEAD x86_64-oemu
```

Next, we checkout to non-instrumented version of the kernel source.
```sh 
cd $PROJECT_HOME/kernels/linux
git checkout 480e035
```

To build the non-instrumented kernel, run following commands:
```sh
cd $PROJECT_HOME
CONFIG=./kernels/guest/configs/config.x86_64 ./scripts/linux/build.sh

# For this message, type 'y'
Do you want to proceed? [yn] y

# Now, there will be three directories like this:
→ ls $PROJECT_HOME/kernels/guest/builds -l
x86_64 -> ozz/kernels/guest/builds/x86_64-HEAD
x86_64-oemu     # OEMU-instrumented kernel
x86_64-HEAD     # non-instrumented kernel
```

Restore original setting:
```sh
cd $PROJECT_HOME/kernels/guest/builds
mv x86_64-HEAD x86_64-baseline
mv x86_64-oemu x86_64-HEAD

→ ls $PROJECT_HOME/kernels/guest/builds -l
x86_64 -> /home/artifact/ozz/kernels/guest//builds/x86_64-HEAD
x86_64-baseline     # non-instrumented kernel
x86_64-HEAD         # OEMU-instrumented kernel
```

## Applying Patch for LMBench

LMBench has several bugs, so we apply some patches(`lmbench_patches/*.patch`).
Clone the repository and apply the patch by running the following commands:

```sh
cd ~
git clone git@github.com:intel/lmbench.git

cd lmbench
git checkout 701c6c3 # checkout to the version we used
git am path/to/lmbench_patches/* # apply patches
```

## Running LMBench

We need to run LMBench in the guest kernel, so it requires interactions between the host and guest kernel.
Thus, artifact committees need to run two terminals, one for running the guest kernel in a virtual machine and the other for uploading required files from the host to the guest.
All below commands are needed to be executed twice to 1) measure the performance of the OEMU-instrumented kernel, and 2) measure the performance of the baseline kernel.

If you are using the provided machine and skipped the earlier sections, the following files should already be in place:

```sh
# Check lmbench source at home directory
→ ls ~
lmbench  ozz  ozz_artifact

# Check two compiled kernels
→ ls $PROJECT_HOME/kernels/guest/builds -l
x86_64 -> /home/artifact/ozz/kernels/guest//builds/x86_64-HEAD
x86_64-baseline     # non-instrumented kernel
x86_64-HEAD         # OEMU-instrumented kernel
```
The instructions below assume that the location and names of each file are as shown.

#### **Step 1: Start the Guest Kernel (Host)**

- For the OEMU-instrumented kernel:
```sh
cd $PROJECT_HOME
KERNEL="$KERNELS_DIR/guest/builds/x86_64-HEAD/arch/x86_64/boot/bzImage" ./scripts/qemu/run_qemu.sh
```

- For the non-instrumented kernel:
```sh
cd $PROJECT_HOME
KERNEL="$KERNELS_DIR/guest/builds/x86_64-baseline/arch/x86_64/boot/bzImage" ./scripts/qemu/run_qemu.sh
```

#### **Step 2: Login as Root (Guest)**

```sh
syzkaller login: root

root@syzkaller:~#
```

#### **Step 3: Upload LMBench to the Guest Kernel (Host)**

```sh
./scripts/qemu/upload.sh ~/lmbench
```

#### **Step 4: Build and Upload `enable_kssb` (Host, OEMU-instrumented only)**

```sh
cd $PROJECT_HOME/kernels/guest/tests
make

cd $PROJECT_HOME
./scripts/qemu/upload.sh $PROJECT_HOME/kernels/guest/tests/build/tests/enable_kssb
```

#### **Step 5: Build LMBench (Guest)**

```sh
cd lmbench
make
```

#### **Step 6: Initial Run (Guest)**

1. **Run the benchmark**:

```sh
make results
```

2. **Benchmark Configuration**:
    - For `Job placement selection`, select `1`
    - For `MB [default XXX]`, select `4096`   
    - For `Mail results`, select `no`
    - (**09/11/2024 updated**) For `SUBSET (ALL|HARWARE|OS|DEVELOPMENT)`, select `OS`
    - Press `Enter` to accept all default settings

Ensure the message `Configuration done, thanks.` appears. The benchmark will take about 2 hours.

#### **Step 7: Enable OEMU Callbacks (Guest, OEMU-instrumented only)**

```sh
~/enable_kssb
```

From now on, the guest kernel will execute OEMU callback and will be slow. 

#### **Step 8: Rerun the Benchmark (Guest)**

```sh
make rerun
```

The OEMU-instrumented kernel run will take about 12 hours or more (dependent on machine), while the non-instrumented kernel run will take about 2 - 3 hours. For more accurate results, one can rerun the benchmark multiple times.

#### **Step 9: View Results (Guest)**

Please note that the numbers in the paper were measured on a different machine, so the results may vary depending on the machine.

```sh
# You can see results like this format:
→ make see
cd results && make summary percent 2>/dev/null | more
make[1]: Entering directory '/root/lmbench/results'

                 L M B E N C H  3 . 0   S U M M A R Y
                 ------------------------------------
                 (Alpha software, do not distribute)


Processor, Processes - times in microseconds - smaller is better
------------------------------------------------------------------------------
Host                 OS  Mhz null null      open slct sig  sig  fork exec sh
                             call  I/O stat clos TCP  inst hndl proc proc proc
--------- ------------- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ----
syzkaller Linux 6.8.0-g 1494 1.88 2.43 101. 212. 46.6 3.15 45.0 11.K 28.K 60.K
syzkaller Linux 6.8.0-g 1494 49.2 33.9 701. 1335 1531 49.9 574. 144K 326K 612K
# In instrumented kernel, first line doesn't show actual overhead
# since OEMU callback is not enabled in the first run.

syzkaller Linux 6.8.0-g 2964 1.33 1.72 72.0 134. 32.6 2.15 32.0 14.K 33.K 62.K

Context switching - times in microseconds - smaller is better
-------------------------------------------------------------------------
Host                 OS  2p/0K 2p/16K 2p/64K 8p/16K 8p/64K 16p/16K 16p/64K
                         ctxsw  ctxsw  ctxsw ctxsw  ctxsw   ctxsw   ctxsw
--------- ------------- ------ ------ ------ ------ ------ ------- -------
syzkaller Linux 6.8.0-g  101.3   98.5   94.4  210.3  277.9   231.7   224.8
syzkaller Linux 6.8.0-g   12.6   17.6   73.0  426.1  385.1   306.0   612.7

*Local* Communication latencies in microseconds - smaller is better
---------------------------------------------------------------------
Host                 OS 2p/0K  Pipe AF     UDP  RPC/   TCP  RPC/ TCP
                        ctxsw       UNIX         UDP         TCP conn
--------- ------------- ----- ----- ---- ----- ----- ----- ----- ----
syzkaller Linux 6.8.0-g 101.3 150.8 275. 374.0       494.1       1328
syzkaller Linux 6.8.0-g  12.6 569.9 3074 4316.       5220.       14.1

File & VM system latencies in microseconds - smaller is better
-------------------------------------------------------------------------------
Host                 OS   0K File      10K File     Mmap    Prot   Page   100fd
                        Create Delete Create Delete Latency Fault  Fault  selct
--------- ------------- ------ ------ ------ ------ ------- ----- ------- -----
syzkaller Linux 6.8.0-g                              753.4K 9.776    16.4  19.2
syzkaller Linux 6.8.0-g                             8014.2K 146.0   332.6 475.0

*Local* Communication bandwidths in MB/s - bigger is better
-----------------------------------------------------------------------------
Host                OS  Pipe AF    TCP  File   Mmap  Bcopy  Bcopy  Mem   Mem
                             UNIX      reread reread (libc) (hand) read write
--------- ------------- ---- ---- ---- ------ ------ ------ ------ ---- -----
syzkaller Linux 6.8.0-g 109. 349. 210. 3224.8 8988.1 9381.1 4419.1 9083 6642.
syzkaller Linux 6.8.0-g 4.61 21.4 12.7  207.8  14.1K  10.5K 7833.2 10.K 9200.
make[1]: Leaving directory '/root/lmbench/results'
```
