# Ozz SOSP24 Artifact Evaluation

## Open Source Software

1. OEMU's compiler pass (Section 3. in the paper)

	- https://github.com/casys-kaist/ozz/tree/master/tools/SoftStoreBufferPass
	- When compiling the kernel, it replaces load/store operations with callback functions (see below) that emulate memory access reordering.

2. OEMU's callbacks (Section 3. in the paper)

	- https://github.com/casys-kaist/ozz_kernel/tree/master/kernel/kssb
	- When compiling the kernel, callbacks are transplanted into the kernel. During kernel runtime, callbacks manipulate the order of memory accesses instead of reading from/writing to memory directly.

3. A custom scheduler implemented in QEMU (Section 4.4.1. in the paper)

	- https://github.com/casys-kaist/ozz/tree/master/tools/qemu/qcsched
	- The custom scheduler exposes hypercall interfaces to a guest userspace program, granting the userspace program (i.e., Ozz) the ability to control the kernel's thread scheduling.

4. Ozz (Section 4. in the paper)

	- https://github.com/casys-kaist/ozz
	- A kernel fuzzer specifically tailored to detect out-of-order concurrency bugs in the kernel

## System Setup

The Ozz's userspace executor needs to invoke hypercalls from the guest
userspace, but the deployed kvm module rejects hypercalls from
the guest userspace. Thus, the kvm module should be patched to accept
hypercalls from the guest userspace.

To enable hypercalls from the guest userspace, one needs to modify
**the host kvm module**. Since modifying (a module of) the host kernel
is a burdensome task, we provide a machine that is already equipped
with the modified kvm module to artifact committees, and encourage
them to use the provided machine.

To use the provided machine, we kindly ask artifact committees to send
us their public keys for ssh (possibly through comments in the HotCRP
system), so we can permit them to access the server.

To access the server after acquiring access permission, artifact
committees can access the server as follows:
```
ssh artifact@[REDACTED] -p 10022     # Server 1
```

Just in case, we have prepared another server where its IP address is
`[REDACTED]`. However, please note that `[REDACTED]` (Server 1) is the only address that is accessible from outside of our
institute, while `[REDACTED]` (Server 2) is *not*. Thus, to access
`[REDACTED]` (Server 2), artifact committees need to connect to
`[REDACTED]` (Server 1) first, and then connect to
`[REDACTED]` (Server 2) from Server 1.

```
ssh artifact@[REDACTED]              # Server 2, use the default 22 port
```

If using the provided machine, one can skip from [Obtaining source
code](#obtaining-source-code) to [Installing Ozz](#installing-ozz),
and can directly go to [Artifact
evaluation](#artifact-evaluation). Alternatively, one can build all
artifacts by him/herself. How to build Ozz from scratch is described
from [Obtaining source code](#obtaining-source-code).

### Obtaining Source Code

There are two repositories, one for Ozz and the other for the target
kernel which incorporates OEMU's callback functions. The target kernel
repository is added to Ozz's repository as a submodule. To get the
source code, run the following commands:

```sh
git clone git@github.com:casys-kaist/ozz.git
cd ozz
git submodule update --init --recursive
```

### Modifying the Host KVM Module

In the provided machines, we have prepared and patched the kvm
module. So, if an artifact committee uses provided machines, he or
she does not need to worry about this.

If one wants to patch the kvm module by oneself, he or she can apply
patches in `$PROJECT_HOME/kernels/host/patches` into the kernel source
code, and then build and install the kernel (or the kvm
module).

Unfortunately, we couldn't write a one-button script to modify the
host kvm module, because repositories of the host kernel differ
depending on the running operating system and human intervention is
required.

In the case of Ubuntu, one can find the repository of the kernel in
[Ubuntu Wiki](https://wiki.ubuntu.com/Kernel/Dev/KernelGitGuide). Here
are some examples:

| Ubuntu Release | Kernel repository |
| --- | --- |
| focal | git://git.launchpad.net/~ubuntu-kernel/ubuntu/+source/linux/+git/focal |
| jammy | git://git.launchpad.net/~ubuntu-kernel/ubuntu/+source/linux/+git/jammy |
| noble | git://git.launchpad.net/~ubuntu-kernel/ubuntu/+source/linux/+git/noble |

Since modifying the host kernel may affect the entire system, we would
like to ask you to carefully read the webpage to sufficiently
understand the consequences before patching the host kvm module.


### Installing Toolchains

Ozz depends on various toolchains (eg, LLVM, Capstone, ...). If an
artifact committee decides to use a provided machine, please do *NOT*
install toolchains again, as they consume a large amount of the disk
space (eg, >100GB mainly due to LLVM) and a long time (eg, a few
hours). Instead, please skip to the [Artifact
evaluation](#artifact-evaluation) section.

In the case that one decides to install Ozz in their own machine, we
provide scripts to install the toolchains.

First, `cd` into the project directory. In the directory, run the
following command to setup environment variables that are used in the
artifact evaluation process.

```sh
source ./scripts/envsetup.sh
```

After sourcing the script, the `$PROJECT_HOME` environment variable
points to the root directory of the repository.

Then, run the following command to install toolchains.
```sh
./scripts/install.sh
```
It will take a few hours.

All binaries will be installed in `$PROJECT_HOME`, so to remove Ozz,
just remove the `$PROJECT_HOME` directory.


### Installing Ozz

After installing toolchains, one needs to build the Ozz's compiler
pass, the target Linux kernel, a disk image, and Ozz. Again, we have
prepared them in the provided machines.

The Ozz's compiler pass resides in
`$PROJECT_HOME/tools/SoftStoreBufferPass`. To build it, run the
following commands:

```sh
mkdir -p $PROJECT_HOME/tools/SoftStoreBufferPass/build
cd $PROJECT_HOME/tools/SoftStoreBufferPass/build
cmake ..
ninja

# Check the result
→ ls $PROJECT_HOME/tools/SoftStoreBufferPass/build/pass
...
libSSBPass.so      # The LLVM pass shared object
...
```

The target Linux kernel should be built after the Ozz's compiler pass,
as the compiler pass is required to build the kernel. To build the
target Linux kernel, run the following commands:

```sh
cd $PROJECT_HOME
CONFIG=./kernels/guest/configs/config.x86_64 ./scripts/linux/build.sh

# Check the result. vmlinux and bzImage should be there
→ ls $PROJECT_HOME/kernels/guest/builds/x86_64
arch   built-in.a  crypto   fs       init      ipc     lib       mm               modules.builtin.modinfo  Module.symvers  scripts   sound   System.map  usr   vmlinux    vmlinux.o
block  certs       drivers  include  io_uring  kernel  Makefile  modules.builtin  modules.order            net             security  source  tools       virt  vmlinux.a

→ ls $PROJECT_HOME/kernels/guest/builds/x86_64/arch/x86/boot
a20.o       cmdline.o   cpucheck.o  cpustr.h                header.o  mkcpustr  printf.o   setup.elf  tty.o         video-mode.o  video-vga.o  zoffset.h
bioscall.o  compressed  cpuflags.o  early_serial_console.o  main.o    pmjump.o  regs.o     string.o   version.o     video.o       vmlinux.bin
bzImage     copy.o      cpu.o       edd.o                   memory.o  pm.o      setup.bin  tools      video-bios.o  video-vesa.o  voffset.h
```

Ozz requires a disk image to correctly boot up a virtual machine. To
create the disk image, run the following commands.

```sh
cd $PROJECT_HOME/kernels/guest/images/x86_64
./create-image-x86_64.sh

# Check the result. bookworm.img and bookworm.id_rsa are required.
→ ls $PROJECT_HOME/kernels/guest/images/x86_64
bookworm.id_rsa  bookworm.id_rsa.pub  bookworm.img  create-image-x86_64.sh
```

If the script exits with an error message as below,
```
E: Release signed by unknown key (key id F8D2585B8783D481)
   The specified keyring /usr/share/keyrings/debian-archive-keyring.gpg may be incorrect or out of date.
   You can find the latest Debian release key at https://ftp-master.debian.org/keys.html
```
get a new keyring and inform `debootstrap` to use it as follows:
```sh
cd $PROJECT_HOME/kernels/guest/images/x86_64
wget https://ftp-master.debian.org/keys/release-12.asc -qO- | gpg --import --no-default-keyring --keyring ./debian-release-12.gpg
./create-image-x86_64.sh -k ./debian-release-12.gpg
```

Lastly, run the following commands to build Ozz:
```sh
cd $PROJECT_HOME/tools/syzkaller
make

# Check the result.
→ ls $PROJECT_HOME/tools/syzkaller/bin
linux_amd64  syz-db  syz-manager  syz-sysgen

→ ls $PROJECT_HOME/tools/syzkaller/bin/linux_amd64
syz-executor  syz-fuzzer
```

## Artifact Evaluation

After installing toolchains and Ozz (or by using the provided
machines), artifact committees can run Ozz and evaluate artifacts.

This artifact evaluation evaluates three key results, 1) [reproducing
the newly-found bugs](#reproducing-the-newly-found-bugs-table-3), 2)
[reproducing the previously-reported
bugs](#reproducing-the-previously-reported-bugs-table-4), and
[confirming the performance overhead of
OEMU](#measuring-the-performance-overhead-of-oemu-table-5).

In addition, the functionality can be checked by [running
Ozz](#running-ozz).


### Running Ozz

Ozz is implemented based on Syzkaller. Thus, a configuration of
Syzkaller should be generated:
```sh
cd $PROJECT_HOME/exp/x86_64
envsubst < templates/syzkaller.cfg.template > syzkaller.cfg
```

Then, run Ozz as follows:
```sh
cd $PROJECT_HOME/exp/x86_64
../scripts/run_fuzzer.sh

Run syzkaller
    syzkaller : ozz/gotools/src/github.com/google/syzkaller/bin/syz-manager
    kernel    : (default) ozz/kernels/guest/builds/x86_64-HEAD
    config    : ozz/exp/x86_64/syzkaller.cfg
    debug     :
    options   :  -config ozz/exp/x86_64/syzkaller.cfg
    tee       : ozz/exp/x86_64/workdir/log
    timestamp :
    workdir   : ozz/exp/x86_64/workdir
[MANAGER] 2024/08/17 19:09:12 loading corpus...
[MANAGER] 2024/08/17 19:09:13 serving http on http://:56741
```

Note that running Ozz for the first time may take a few minutes to
start up. Please wait until it starts running.

**Please be aware that the provided machines are shared with other
artifact committees. Running Ozz with the above commands consumes the
whole computing power of the machine, and we may kill the running Ozz
instance if it interferes with artifact evaluation.**

### Reproducing the Newly-found Bugs (Table 3)

We provide PoCs to reproduce newly-found bugs, as it is infeasible for
artifact committees to run Ozz for a long time (eg, a month) to
re-find the bugs that we found during our evaluation.

To reproduce a bug, artifact committee can run the following command
and simply observe a crash:

```sh
cd $PROJECT_HOME/exp/x86_64
DEBUG=1 OPTS="-seed=TEST_NAME -load-reordering=true" ../scripts/run_fuzzer.sh
```
where `TEST_NAME` is a file name of a PoC (eg, `reorderings/11-gsm`).

A full list of `TEST_NAME` is as follows (also refer to Table 3 in the paper):
- `reorderings/1-rds`
- `reorderings/2-watchqueue`
- `reorderings/3-vmci`
- `reorderings/4-xdp`
- `reorderings/5-tls`
- `reorderings/6-bpf`
- `reorderings/7-xdp2`
- `reorderings/8-smc`
- `reorderings/9-tls2`
- `reorderings/10-smc2`
- `reorderings/11-gsm`

If a bug is reproduced, you can see the kernel output like below.

```==================================================================
BUG: KASAN: null-ptr-deref in instrument_atomic_read_write kernels/linux/include/linux/instrumented.h:125 [inline]
BUG: KASAN: null-ptr-deref in atomic_long_dec_and_test kernels/linux/include/linux/atomic/atomic-instrumented.h:4686 [inline]
BUG: KASAN: null-ptr-deref in fput+0x44/0x370 kernels/linux/fs/file_table.c:427
Write of size 8 at addr 0000000000000058 by task syz-executor.0/21117

CPU: 2 PID: 21117 Comm: syz-executor.0 Not tainted 6.8.0-rc1-gab2867bfb07f-dirty #15
Hardware name: QEMU Standard PC (i440FX + PIIX, 1996), BIOS rel-1.14.0-0-g155821a1990b-prebuilt.qemu.org 04/01/2014
Call Trace:
 <TASK>
 __dump_stack kernels/linux/lib/dump_stack.c:88 [inline]
 dump_stack_lvl+0x2b1/0x410 kernels/linux/lib/dump_stack.c:106
 print_report+0xed/0x220 kernels/linux/mm/kasan/report.c:491
 kasan_report+0x131/0x160 kernels/linux/mm/kasan/report.c:601
 kasan_check_range+0x27e/0x2b0 kernels/linux/mm/kasan/generic.c:189
 instrument_atomic_read_write kernels/linux/include/linux/instrumented.h:125 [inline]
 atomic_long_dec_and_test kernels/linux/include/linux/atomic/atomic-instrumented.h:4686 [inline]
 fput+0x44/0x370 kernels/linux/fs/file_table.c:427
 fput_light kernels/linux/include/linux/file.h:33 [inline]
 __sys_setsockopt kernels/linux/net/socket.c:2336 [inline]
 __do_sys_setsockopt kernels/linux/net/socket.c:2343 [inline]
 __se_sys_setsockopt+0x210/0x2c0 kernels/linux/net/socket.c:2340
 __x64_sys_setsockopt+0x112/0x1a0 kernels/linux/net/socket.c:2340
 do_syscall_x64 kernels/linux/arch/x86/entry/common.c:53 [inline]
 do_syscall_64+0xf9/0x220 kernels/linux/arch/x86/entry/common.c:85
 entry_SYSCALL_64_after_hwframe+0x63/0x6b
RIP: 0033:0x472f6d
Code: c3 e8 17 28 00 00 0f 1f 80 00 00 00 00 f3 0f 1e fa 48 89 f8 48 89 f7 48 89 d6 48 89 ca 4d 89 c2 4d 89 c8 4c 8b 4c 24 08 0f 05 <48> 3d 01 f0 ff ff 73 01 c3 48 c7 c1 b0 ff ff ff f7 d8 64\
 89 01 48
RSP: 002b:00007f3cdd8d3028 EFLAGS: 00000246 ORIG_RAX: 0000000000000036
RAX: ffffffffffffffda RBX: 00000000005a1428 RCX: 0000000000472f6d
RDX: 000000000000001e RSI: 0000000000000006 RDI: 0000000000000003
RBP: 00000000f477909a R08: 0000000000000004 R09: 0000000000000000
R10: 0000000020000040 R11: 0000000000000246 R12: 00000000005a14e0
R13: 00000000005a1434 R14: 00000000004314a0 R15: 00007f3cdd8b3000
 </TASK>
==================================================================
```

### Reproducing the Previously-reported Bugs (Table 4)

Similarly to reproducing the newly found bugs, we provide PoCs of the
previously-reported bugs and artifact committees can reproduce them by
running the following command:

```sh
cd $PROJECT_HOME/exp/x86_64
DEBUG=1 OPTS="-seed=TEST_NAME -load-reordering=true" ../scripts/run_fuzzer.sh
```
where `TEST_NAME` is a file name of a PoC (eg, `reordering-cve/1-vlan`).

A full list of `TEST_NAME` is as follows (also refer to Table 4 in the paper):
- `reordering-cve/1-vlan`
- `reordering-cve/2-watchqueue`
- `reordering-cve/3-xsk`
- `reordering-cve/4-xsk2`
- `reordering-cve/5-fs` (**09/15/2024 Updated**: please refer to [this documentation](running-oemu.md))
- `reordering-cve/7-nbd`
- `reordering-cve/9-unix`

Note that two bugs in Table 4 are excluded as \#6 is not reproducible
and \#8 does not cause an observable symptom.

If a bug is reproduced, you will see a kernel output similar to the above.


### Measuring the Performance Overhead of OEMU (Table 5)

We measure the performance overhead of OEMU with LMBench. Because the
documentation of running LMBench is a bit long, we separate the
document. Please refer to [this documentation](lmbench.md).
