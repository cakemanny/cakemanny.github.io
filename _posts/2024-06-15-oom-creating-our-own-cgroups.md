---
layout: post
title: "OOM Killer: Creating our own CGroups"
date: 2024-06-25
---

Last time, when [triggering the OOM Killer][1], we were a bit silent
but we looked at how docker creates cgroups,
how the memory limits are seen on the cgroup
when we limit the memory of a container
and how it looks when the OOM killer gets involved and kills our process
when we run a command that exceeds those limits.

[1]: /2024/05/17/triggering-the-oom-killer.html

Today I want to go a bit deeper and expand our understanding.
We learn by doing, not by watching.
So, instead of watching docker create a cgroup, we're going to create our own.

First let's look at our cgroup tree in systemd

```
# Terminal 1
dan:~ % systemctl status
...
           └─user.slice
             └─user-1000.slice
               ├─session-1.scope
               │ ├─ 697 "sshd: dan [priv]"
               │ ├─ 711 "sshd: dan@pts/0"
               │ ├─ 712 -zsh
               │ ├─3533 systemctl status
               │ └─3534 pager
               ├─session-3.scope
               │ ├─1495 "sshd: dan [priv]"
               │ ├─1501 "sshd: dan@pts/1"
               │ └─1502 -zsh
               └─user@1000.service
                 └─init.scope
                   ├─700 /lib/systemd/systemd --user
                   └─701 "(sd-pam)"
```

Let's create our own cgroup called `dans-slice`, inside `user-1000.slice`

```
# Terminal 1
dan:~ % sudo mkdir /sys/fs/cgroup/user.slice/user-1000.slice/dans-slice
dan:~ % ls -p /sys/fs/cgroup/user.slice/user-1000.slice/dans-slice
cgroup.controllers      cpu.pressure         memory.pressure
cgroup.events           cpu.stat             memory.reclaim
cgroup.freeze           cpu.weight           memory.stat
cgroup.kill             cpu.weight.nice      memory.swap.current
cgroup.max.depth        io.pressure          memory.swap.events
cgroup.max.descendants  memory.current       memory.swap.high
cgroup.pressure         memory.events        memory.swap.max
cgroup.procs            memory.events.local  memory.zswap.current
cgroup.stat             memory.high          memory.zswap.max
cgroup.subtree_control  memory.low           pids.current
cgroup.threads          memory.max           pids.events
cgroup.type             memory.min           pids.max
cpu.idle                memory.numa_stat     pids.peak
cpu.max                 memory.oom.group
cpu.max.burst           memory.peak
```

If we check `systemctl status`, we would see that it doesn't appear
in the cgroup tree.

Let's add a process to it.

```
# Terminal 2
dan:~ % sleep 1000
```

```
# Terminal 1
dan:~ % pgrep sleep
3635
dan:~ % echo 3635 | sudo tee /sys/fs/cgroup/user.slice/user-1000.slice/dans-slice/cgroup.procs
3635
dan:~ % systemctl status
...
           └─user.slice
             └─user-1000.slice
               ├─dans-slice         ## <---- our cgroup
               │ └─3635 sleep 1000  ## <---- our sleep process
               ├─session-1.scope
               │ ├─ 697 "sshd: dan [priv]"
               │ ├─ 711 "sshd: dan@pts/0"
               │ ├─ 712 -zsh
               │ ├─3655 systemctl status
               │ └─3656 pager
               ├─session-3.scope
               │ ├─1495 "sshd: dan [priv]"
               │ ├─1501 "sshd: dan@pts/1"
               │ └─1502 -zsh
               └─user@1000.service
                 └─init.scope
                   ├─700 /lib/systemd/systemd --user
                   └─701 "(sd-pam)"
```

`sleep` is unlikely to run out of memory very soon,
so let's <kbd>ctrl-c</kbd> it
and run something a bit different.

Let's create a shell and put it in `dans-slice`,
then we'll be able to see that subprocesses are create within the same cgroup.
This will save us from continually needing to echo pids into `cgroup.procs`.

```
# Terminal 2
dan:~ % bash
dan@debian:~$
```

```
# Terminal 1
dan:~ % pgrep bash | sudo tee /sys/fs/cgroup/user.slice/user-1000.slice/dans-slice/cgroup.procs
3713
dan:~ % systemctl status | fgrep -A 100 user.slice | head
           └─user.slice
             └─user-1000.slice
               ├─dans-slice
               │ └─3713 bash
               ├─session-1.scope
               │ ├─ 697 "sshd: dan [priv]"
               │ ├─ 711 "sshd: dan@pts/0"
               │ ├─ 712 -zsh
               │ ├─3781 systemctl status
               │ ├─3782 grep -F -A 100 user.slice
```

```
# Terminal 2
dan@debian:~$ sleep 10
```

```
# Terminal 1
dan:~ % systemctl status | fgrep -A 100 user.slice | head
           └─user.slice
             └─user-1000.slice
               ├─dans-slice
               │ ├─3713 bash
               │ └─3787 sleep 10  ## <-- subprocess appears aside parent
               ├─session-1.scope
               │ ├─ 697 "sshd: dan [priv]"
               │ ├─ 711 "sshd: dan@pts/0"
               │ ├─ 712 -zsh
               │ ├─3788 systemctl status
```

Since we're interested in the OOM Killer,
let's set ourselves some limits.
This time, 20 MiB and let's not allow any swapping.
I leave it as an exercise to the reader
to investigate how `memory.swap.*` and `memory.zswap.*` behave.

```
# Terminal 1
dan:~ % cat /sys/fs/cgroup/user.slice/user-1000.slice/dans-slice/memory.current
73728
dan:~ % echo $((20 * 1024 * 1024)) | sudo tee /sys/fs/cgroup/user.slice/user-1000.slice/dans-slice/memory.max
20971520
dan:~ % echo 0 | sudo tee /sys/fs/cgroup/user.slice/user-1000.slice/dans-slice/memory.swap.max
0
```

Now let's try to exceed these memory limits.
First let's write a program that allocates 50MiB.

```
# Terminal 1
dan:~ % mkdir -p src/bigleak && cd src/bigleak
dan:~/src/bigleak % edit bigleak.c
...
dan:~/src/bigleak % cat bigleak.c
#include <stdlib.h>
#include <unistd.h>

int main()
{
    void* x = malloc(50 * 1024 * 1024);

    for (;;) {
        sleep(1);
    }

    free(x);
}
dan:~/src/bigleak % make bigleak
cc     bigleak.c   -o bigleak
```

```
# Terminal 2
dan@debian:~/src/bigleak$ ./bigleak

```

Why don't we get "Killed"? Let's check how much memory its using:

```
# Terminal 1
dan:~/src/bigleak % cat /sys/fs/cgroup/user.slice/user-1000.slice/dans-slice/memory.current
380928
```

Only 372 KiB ...

Perhaps it's because we haven't used the memory.
Let's write a new program that writes to the memory, MiB by MiB:

```
# Terminal 1
dan:~/src/bigleak % edit bigleak2.c
...
dan:~/src/bigleak % cat bigleak2.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main()
{
    enum { N =  50 * 1024 * 1024 };
    char* x = malloc(N);

    for (int i = 0; i < N; i += (1024 * 1024)) {
        printf("i = %d\n", i);
        memset(x + i, 0, 1024 * 1024);
    }

    free(x);
}
dan:~/src/bigleak % make bigleak2
cc     bigleak2.c   -o bigleak2
```

```
# Terminal 2
^C
dan@debian:~/src/bigleak$ ./bigleak2
i = 0
i = 1048576
i = 2097152
i = 3145728
i = 4194304
i = 5242880
i = 6291456
i = 7340032
i = 8388608
i = 9437184
i = 10485760
i = 11534336
i = 12582912
i = 13631488
i = 14680064
i = 15728640
i = 16777216
i = 17825792
i = 18874368
i = 19922944
Killed
dan@debian:~/src/bigleak$
```


We can again see the details in the kernel logs with

```
sudo journalctl --system --dmesg -xe --utc
```

Or, since I want to reduce the width of this to fit it in the blog, let's
take it straight from `dmesg`.

```
# Terminal 1
dan:~/src/bigleak % sudo dmesg | fgrep -A 1000 bigleak2 | cut -c 16-
bigleak2 invoked oom-killer: gfp_mask=0xcc0(GFP_KERNEL), order=0, oom_score_adj=0
CPU: 1 PID: 4045 Comm: bigleak2 Not tainted 6.1.0-21-arm64 #1  Debian 6.1.90-1
Hardware name: QEMU QEMU Virtual Machine, BIOS 0.0.0 02/06/2015
Call trace:
 dump_backtrace+0xe4/0x140
 show_stack+0x20/0x30
 dump_stack_lvl+0x64/0x80
 dump_stack+0x18/0x34
 dump_header+0x4c/0x200
 oom_kill_process+0x2ec/0x2f0
 out_of_memory+0xec/0x590
 mem_cgroup_out_of_memory+0x134/0x14c
 try_charge_memcg+0x584/0x66c
 charge_memcg+0x54/0xc0
 __mem_cgroup_charge+0x40/0x8c
 __handle_mm_fault+0x620/0x1100
 handle_mm_fault+0xe4/0x260
 do_page_fault+0x174/0x3c0
 do_translation_fault+0x54/0x70
 do_mem_abort+0x4c/0xa0
 el0_da+0x48/0xf0
 el0t_64_sync_handler+0xac/0x120
 el0t_64_sync+0x18c/0x190
memory: usage 20480kB, limit 20480kB, failcnt 148
swap: usage 0kB, limit 0kB, failcnt 0
Memory cgroup stats for /user.slice/user-1000.slice/dans-slice:
anon 20705280
file 0
kernel 188416
kernel_stack 16384
pagetables 81920
sec_pagetables 0
percpu 0
sock 0
vmalloc 0
shmem 0
zswap 0
zswapped 0
file_mapped 0
file_dirty 0
file_writeback 0
swapcached 0
anon_thp 18874368
file_thp 0
shmem_thp 0
inactive_anon 20652032
active_anon 4096
inactive_file 0
active_file 0
unevictable 0
slab_reclaimable 0
slab_unreclaimable 18752
slab 18752
workingset_refault_anon 0
workingset_refault_file 0
workingset_activate_anon 0
workingset_activate_file 0
workingset_restore_anon 0
workingset_restore_file 0
workingset_nodereclaim 0
pgscan 0
pgsteal 0
pgscan_kswapd 0
pgscan_direct 0
pgsteal_kswapd 0
pgsteal_direct 0
pgfault 2461
pgmajfault 0
pgrefill 0
pgactivate 0
pgdeactivate 0
pglazyfree 0
pglazyfreed 0
zswpin 0
zswpout 0
thp_fault_alloc 19
thp_collapse_alloc 0
Tasks state (memory values in pages):
[  pid  ]   uid  tgid total_vm      rss pgtables_bytes swapents oom_score_adj name
[   3713]  1000  3713     2115     1257    57344        0             0 bash
[   4045]  1000  4045    13348     5298    90112        0             0 bigleak2
oom-kill:constraint=CONSTRAINT_MEMCG,nodemask=(null),cpuset=user.slice,mems_allowed=0,oom_memcg=/user.slice/user-1000.slice/dans-slice,task_memcg=/user.slice/user-1000.slice/dans-slice,task=bigleak2,pid=4045,uid=1000
Memory cgroup out of memory: Killed process 4045 (bigleak2) total-vm:53392kB, anon-rss:20020kB, file-rss:1172kB, shmem-rss:0kB, UID:1000 pgtables:88kB oom_score_adj:0
```

So, what happened there?
As we first start use the memory, we take a page fault and the kernel tries
to find us some physical memory.
We reached the limit of the physical memory available to our cgroup and so
the kernel chose to kill a process from our cgroup to free some memory.
It went for `bigleak2`.

So to wrap up. We learned how to create our own cgroup, set memory limits for
it and to exhaust those limits by writing to memory we've been allocated.
