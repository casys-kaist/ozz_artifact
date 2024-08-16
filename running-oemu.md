# Running User-program PoC with OEMU

In this section, we provide instructions for running a user-program PoC (Proof of Concept) in the OEMU-instrumented kernel to reproduce a bug without using Ozz. 

Before running any commands, please ensure that you have set up the environment variables:
```sh
# In project home directory
source ./scripts/envsetup.sh
```

We provide some pre-built binaries for convenience: the kernel image ([link](https://github.com/casys-kaist/ozz_kernel/tree/3e62e1011b0ae011ef8787d65ce313f169189c22)) and a PoC ([link](https://github.com/casys-kaist/ozz/blob/master/kernels/guest/tests/tests/fs.c)).

```sh
# Check provided files
→ ls $PROJECT_HOME/reproduce -l
bzImage # A pre-built instrumented kernel ()
fs      # A pre-built userprogram poc
```
The instructions below assume that the location and names of each file are as shown.

#### **Step 1: Start the Guest Kernel (Host)**

```sh
cd $PROJECT_HOME
KERNEL="$PROJECT_HOME/reproduce/bzImage" ./scripts/qemu/run_qemu.sh
```

#### **Step 2: Login as Root (Guest)**

```sh
syzkaller login: root

root@syzkaller:~#
```

#### **Step 3: Upload the PoC Binary to the Guest Kernel (Host)**

```sh
./scripts/qemu/upload.sh $PROJECT_HOME/reproduce/fs
```

#### **Step 4: Run the PoC (Guest)**

Before running the poc program, we should enable versioned load operation. 

```sh
echo 1 > /sys/kernel/debug/kssb/load_prefetch_enabled
./fs
```

If a bug is reproduced, you can see the below kernel output:

```sh
[  159.860366][ T8280] BUG: kernel NULL pointer dereference, address: 0000000000000800
[  159.862780][ T8280] kssb_pso: Store buffer entries:
[  159.864502][ T8280] kssb_pso: 0 entries
[  159.864704][ T8280] #PF: supervisor read access in kernel mode
[  159.868127][ T8280] #PF: error_code(0x0000) - not-present page
[  159.870179][ T8280] PGD 302b2067 P4D 302b2067 PUD 32dce067 PMD 0 
[  159.872645][ T8280] Oops: 0000 [#1] PREEMPT SMP KASAN
[  159.874900][ T8280] CPU: 2 PID: 8280 Comm: fs Not tainted 6.8.0-08307-gf793f93007bd #1
[  159.877837][ T8280] Hardware name: QEMU Standard PC (i440FX + PIIX, 1996), BIOS rel-1.14.0-0-g155821a1990b-prebuilt.qemu.4
[  159.881025][ T8280] RIP: 0010:__fdget_raw+0x1f7/0x270
[  159.883322][ T8280] Code: 48 8b 44 24 18 44 0f b7 30 80 3d ee 28 d1 16 00 0f 85 88 00 00 00 e9 a1 fd ff ff 48 83 f8 08 0f5
[  159.888021][ T8280] RSP: 0018:ffffc9001311fdf0 EFLAGS: 00010046
[  159.890281][ T8280] RAX: 0000000000000800 RBX: 0000000000000a24 RCX: 0000000000000800
[  159.892877][ T8280] RDX: 0000000000000110 RSI: 0000000000000000 RDI: 0000000000000000
[  159.895471][ T8280] RBP: 0000000000000a24 R08: ffffffff94ac66af R09: 1ffffffff2958cd5
[  159.898058][ T8280] R10: dffffc0000000000 R11: fffffbfff2958cd6 R12: dffffc0000000000
[  159.900649][ T8280] R13: ffffffff98960a50 R14: 0000000000000800 R15: 0000000000000286
[  159.903235][ T8280] FS:  00000000007e93c0(0000) GS:ffff888097f00000(0000) knlGS:0000000000000000
[  159.906213][ T8280] CS:  0010 DS: 0000 ES: 0000 CR0: 0000000080050033
[  159.908630][ T8280] CR2: 0000000000000800 CR3: 000000002c3ba000 CR4: 0000000000750ef0
[  159.911214][ T8280] DR0: 0000000000000000 DR1: 0000000000000000 DR2: 0000000000000000
[  159.913797][ T8280] DR3: 0000000000000000 DR6: 00000000fffe0ff0 DR7: 0000000000000400
[  159.916385][ T8280] PKRU: 55555554
[  159.917888][ T8280] Call Trace:
[  159.919290][ T8280]  <TASK>
[  159.920682][ T8280]  ? __die_body+0x8c/0xe0
[  159.922781][ T8280]  ? page_fault_oops+0x751/0xa40
[  159.925003][ T8280]  ? do_user_addr_fault+0x8cc/0x1440
[  159.927315][ T8280]  ? exc_page_fault+0xba/0x240
[  159.929500][ T8280]  ? asm_exc_page_fault+0x22/0x30
[  159.931725][ T8280]  ? __load_callback_pso+0x474/0x5d0
[  159.934043][ T8280]  ? __fdget_raw+0x1f7/0x270
[  159.936147][ T8280]  __fdget_raw+0x1f7/0x270
[  159.938220][ T8280]  __se_sys_fchdir+0x2b/0x2e0
[  159.940413][ T8280]  __x64_sys_fchdir+0x49/0x80
[  159.942611][ T8280]  do_syscall_64+0xfd/0x240
[  159.944720][ T8280]  entry_SYSCALL_64_after_hwframe+0x62/0x6a
[  159.947148][ T8280] RIP: 0033:0x453aeb
[  159.948973][ T8280] Code: 73 01 c3 48 c7 c1 b0 ff ff ff f7 d8 64 89 01 48 83 c8 ff c3 66 2e 0f 1f 84 00 00 00 00 00 90 f38
[  159.953677][ T8280] RSP: 002b:00007fffafb8c7d8 EFLAGS: 00000217 ORIG_RAX: 0000000000000051
[  159.956506][ T8280] RAX: ffffffffffffffda RBX: 00007fffafb8ca58 RCX: 0000000000453aeb
[  159.959094][ T8280] RDX: 0000000000000000 RSI: 0000000000000000 RDI: 0000000000000100
[  159.961677][ T8280] RBP: 00007fffafb8c800 R08: 00007fffafb8c6ef R09: 00007fffafb8c820
[  159.964266][ T8280] R10: 00007f3faee00640 R11: 0000000000000217 R12: 0000000000000001
[  159.966843][ T8280] R13: 00007fffafb8ca48 R14: 00000000004dd790 R15: 0000000000000001
[  159.969451][ T8280]  </TASK>
[  159.970861][ T8280] Modules linked in:
[  159.972433][ T8280] CR2: 0000000000000800
[  159.974081][ T8280] ---[ end trace 0000000000000000 ]---
[  159.975953][ T8280] RIP: 0010:__fdget_raw+0x1f7/0x270
[  159.978213][ T8280] Code: 48 8b 44 24 18 44 0f b7 30 80 3d ee 28 d1 16 00 0f 85 88 00 00 00 e9 a1 fd ff ff 48 83 f8 08 0f5
[  159.982905][ T8280] RSP: 0018:ffffc9001311fdf0 EFLAGS: 00010046
[  159.985145][ T8280] RAX: 0000000000000800 RBX: 0000000000000a24 RCX: 0000000000000800
[  159.987722][ T8280] RDX: 0000000000000110 RSI: 0000000000000000 RDI: 0000000000000000
[  159.990324][ T8280] RBP: 0000000000000a24 R08: ffffffff94ac66af R09: 1ffffffff2958cd5
[  159.992910][ T8280] R10: dffffc0000000000 R11: fffffbfff2958cd6 R12: dffffc0000000000
[  159.995489][ T8280] R13: ffffffff98960a50 R14: 0000000000000800 R15: 0000000000000286
[  159.998071][ T8280] FS:  00000000007e93c0(0000) GS:ffff888097f00000(0000) knlGS:0000000000000000
[  160.001037][ T8280] CS:  0010 DS: 0000 ES: 0000 CR0: 0000000080050033
[  160.003457][ T8280] CR2: 0000000000000800 CR3: 000000002c3ba000 CR4: 0000000000750ef0
[  160.006023][ T8280] DR0: 0000000000000000 DR1: 0000000000000000 DR2: 0000000000000000
[  160.008624][ T8280] DR3: 0000000000000000 DR6: 00000000fffe0ff0 DR7: 0000000000000400
[  160.011212][ T8280] PKRU: 55555554
[  160.012790][ T8280] Kernel panic - not syncing: Fatal exception
```
