---
layout: post
title: Triggering the OOM Killer
date: 2024-05-17
---

```
dan:~ % docker run -it --rm alpine:latest sh
/ #
```

```
dan:~ % systemctl status
● ubuntu-vm
    State: running
     Jobs: 0 queued
   Failed: 0 units
    Since: Fri 2024-05-17 17:06:11 UTC; 2h 28min ago
   CGroup: /
           ├─user.slice
           │ └─user-1000.slice
           │   ├─user@1000.service
           │   │ └─init.scope
           │   │   ├─1142 /lib/systemd/systemd --user
           │   │   └─1143 (sd-pam)
           │   └─session-1.scope
           │     ├─1139 sshd: dan [priv]
           │     ├─1245 sshd: dan@pts/0
           │     ├─1246 -zsh
           │     └─3690 docker run -it --rm alpine:latest sh
           ├─init.scope
           │ └─1 /sbin/init
           └─system.slice
             ├─containerd.service
             │ ├─ 708 /usr/bin/containerd
             │ └─3725 /usr/bin/containerd-shim-runc-v2 -namespace moby -id 271ef02df83770dece079c3070209fdcacefb26140c608d584db471613343f93 -address /run/containerd/containerd.sock
             ├─docker.service
             │ └─845 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
             ├─docker-271ef02df83770dece079c3070209fdcacefb26140c608d584db471613343f93.scope
             │ └─3746 sh
           ...
```


```
dan:~ % ls -p /sys/fs/cgroup/system.slice/docker-271ef02df83770dece079c3070209fdcacefb26140c608d584db471613343f93.scope/
cgroup.controllers      hugetlb.1GB.events         io.stat
cgroup.events           hugetlb.1GB.events.local   io.weight
cgroup.freeze           hugetlb.1GB.max            memory.current
cgroup.kill             hugetlb.1GB.rsvd.current   memory.events
cgroup.max.depth        hugetlb.1GB.rsvd.max       memory.events.local
cgroup.max.descendants  hugetlb.2MB.current        memory.high
cgroup.procs            hugetlb.2MB.events         memory.low
cgroup.stat             hugetlb.2MB.events.local   memory.max
cgroup.subtree_control  hugetlb.2MB.max            memory.min
cgroup.threads          hugetlb.2MB.rsvd.current   memory.numa_stat
cgroup.type             hugetlb.2MB.rsvd.max       memory.oom.group
cpu.idle                hugetlb.32MB.current       memory.pressure
cpu.max                 hugetlb.32MB.events        memory.stat
cpu.max.burst           hugetlb.32MB.events.local  memory.swap.current
cpu.pressure            hugetlb.32MB.max           memory.swap.events
cpuset.cpus             hugetlb.32MB.rsvd.current  memory.swap.high
cpuset.cpus.effective   hugetlb.32MB.rsvd.max      memory.swap.max
cpuset.cpus.partition   hugetlb.64KB.current       misc.current
cpuset.mems             hugetlb.64KB.events        misc.max
cpuset.mems.effective   hugetlb.64KB.events.local  pids.current
cpu.stat                hugetlb.64KB.max           pids.events
cpu.uclamp.max          hugetlb.64KB.rsvd.current  pids.max
cpu.uclamp.min          hugetlb.64KB.rsvd.max      rdma.current
cpu.weight              io.max                     rdma.max
cpu.weight.nice         io.pressure
hugetlb.1GB.current     io.prio.class
```

```
dan:~ % cat /sys/fs/cgroup/system.slice/docker-*.scope/memory.current
397312
dan:~ % cat /sys/fs/cgroup/system.slice/docker-*.scope/memory.high
max
dan:~ % cat /sys/fs/cgroup/system.slice/docker-*.scope/memory.max
max
dan:~ % cat /sys/fs/cgroup/system.slice/docker-*.scope/memory.oom.group
0
```


```
dan:~ % docker run -it --rm --memory $((8 * 1024 * 1024)) alpine:latest sh
/ #
```

```
dan:~ % systemctl status
...
             ├─docker-3193584578ea07bf5ee503289e7fe7b69f20e60ad236b2b5ce2274419856f8f3.scope
             │ └─3920 sh
...
```

```
dan:~ % cat /sys/fs/cgroup/system.slice/docker-*.scope/memory.current
393216
dan:~ % cat /sys/fs/cgroup/system.slice/docker-*.scope/memory.high
max
dan:~ % cat /sys/fs/cgroup/system.slice/docker-*.scope/memory.max
8388608
dan:~ % cat /sys/fs/cgroup/system.slice/docker-*.scope/memory.oom.group
0
```

```
/ # apk update
fetch https://dl-cdn.alpinelinux.org/alpine/v3.19/main/aarch64/APKINDEX.tar.gz
fetch https://dl-cdn.alpinelinux.org/alpine/v3.19/community/aarch64/APKINDEX.tar.gz
Killed
/ #
```

```
dan:~ % journal -xe
May 17 19:54:29 ubuntu-vm kernel: CPU: 3 PID: 3974 Comm: apk Not tainted 5.15.0-107-generic #117-Ubuntu
May 17 19:54:29 ubuntu-vm kernel: Hardware name: QEMU QEMU Virtual Machine, BIOS 0.0.0 02/06/2015
May 17 19:54:29 ubuntu-vm kernel: Call trace:
May 17 19:54:29 ubuntu-vm kernel:  dump_backtrace+0x0/0x200
May 17 19:54:29 ubuntu-vm kernel:  show_stack+0x20/0x2c
May 17 19:54:29 ubuntu-vm kernel:  dump_stack_lvl+0x68/0x84
May 17 19:54:29 ubuntu-vm kernel:  dump_stack+0x18/0x34
May 17 19:54:29 ubuntu-vm kernel:  dump_header+0x54/0x218
May 17 19:54:29 ubuntu-vm kernel:  oom_kill_process+0x25c/0x260
May 17 19:54:29 ubuntu-vm kernel:  out_of_memory+0xe4/0x360
May 17 19:54:29 ubuntu-vm kernel:  mem_cgroup_out_of_memory+0x150/0x184
May 17 19:54:29 ubuntu-vm kernel:  try_charge_memcg+0x568/0x624
May 17 19:54:29 ubuntu-vm kernel:  charge_memcg+0x5c/0xb4
May 17 19:54:29 ubuntu-vm kernel:  __mem_cgroup_charge+0x40/0x8c
May 17 19:54:29 ubuntu-vm kernel:  __add_to_page_cache_locked+0x25c/0x460
May 17 19:54:29 ubuntu-vm kernel:  add_to_page_cache_lru+0x5c/0x100
May 17 19:54:29 ubuntu-vm kernel:  pagecache_get_page+0x18c/0x6e0
May 17 19:54:29 ubuntu-vm kernel:  filemap_fault+0x4ac/0x944
May 17 19:54:29 ubuntu-vm kernel:  __do_fault+0x44/0x1d0
May 17 19:54:29 ubuntu-vm kernel:  do_read_fault+0xe4/0x1ac
May 17 19:54:29 ubuntu-vm kernel:  do_fault+0xa8/0x220
May 17 19:54:29 ubuntu-vm kernel:  handle_pte_fault+0x5c/0x22c
May 17 19:54:29 ubuntu-vm kernel:  __handle_mm_fault+0x1e4/0x37c
May 17 19:54:29 ubuntu-vm kernel:  handle_mm_fault+0xd0/0x240
May 17 19:54:29 ubuntu-vm kernel:  do_page_fault+0x180/0x524
May 17 19:54:29 ubuntu-vm kernel:  do_translation_fault+0x98/0xdc
May 17 19:54:29 ubuntu-vm kernel:  do_mem_abort+0x4c/0xc0
May 17 19:54:29 ubuntu-vm kernel:  el0_ia+0x98/0x1fc
May 17 19:54:29 ubuntu-vm kernel:  el0t_64_sync_handler+0x124/0x12c
May 17 19:54:29 ubuntu-vm kernel:  el0t_64_sync+0x1a4/0x1a8
May 17 19:54:29 ubuntu-vm kernel: memory: usage 8192kB, limit 8192kB, failcnt 282377
May 17 19:54:29 ubuntu-vm kernel: swap: usage 8192kB, limit 8192kB, failcnt 1832
May 17 19:54:29 ubuntu-vm kernel: Memory cgroup stats for /system.slice/docker-3193584578ea07bf5ee503289e7fe7b69f20e60ad236b2b5ce2274419856f8f3.scope:
May 17 19:54:29 ubuntu-vm kernel: anon 7688192
                                  file 49152
                                  kernel_stack 32768
                                  pagetables 110592
                                  percpu 144
                                  sock 0
                                  shmem 0
                                  file_mapped 36864
                                  file_dirty 0
                                  file_writeback 0
                                  swapcached 290816
                                  anon_thp 0
                                  file_thp 0
                                  shmem_thp 0
                                  inactive_anon 5910528
                                  active_anon 2060288
                                  inactive_file 8192
                                  active_file 4096
                                  unevictable 0
                                  slab_reclaimable 84384
                                  slab_unreclaimable 105456
                                  slab 189840
                                  workingset_refault_anon 3651
                                  workingset_refault_file 85011
                                  workingset_activate_anon 1058
                                  workingset_activate_file 1512
                                  workingset_restore_anon 662
                                  workingset_restore_file 930
                                  workingset_nodereclaim 0
                                  pgfault 40795
                                  pgmajfault 8600
                                  pgrefill 414496
                                  pgscan 1895615
                                  pgsteal 91190
                                  pgactivate 410494
                                  pgdeactivate 412396
                                  pglazyfree 0
                                  pglazyfreed 0
                                  thp_fault_alloc 0
                                  thp_collapse_alloc 0
May 17 19:54:29 ubuntu-vm kernel: Tasks state (memory values in pages):
May 17 19:54:29 ubuntu-vm kernel: [  pid  ]   uid  tgid total_vm      rss pgtables_bytes swapents oom_score_adj name
May 17 19:54:29 ubuntu-vm kernel: [   3920]     0  3920      447      257    45056       33             0 sh
May 17 19:54:29 ubuntu-vm kernel: [   3974]     0  3974     5150     2024    81920     1930             0 apk
May 17 19:54:29 ubuntu-vm kernel: oom-kill:constraint=CONSTRAINT_MEMCG,nodemask=(null),cpuset=docker-3193584578ea07bf5ee503289e7fe7b69f20e60ad236b2b5ce2274419856f8f3.scope,mems_allowed=0,oom_memcg=/system.slice/docker-3193584578ea07bf5ee503289e7fe7b69f20e60ad236b2b5ce2274419856f8f3.scope,task_memcg=/system.slice/docker-3193584578ea07bf5ee503289e7fe7b69f20e60ad236b2b5ce2274419856f8f3.scope,task=apk,pid=3974,uid=0
May 17 19:54:29 ubuntu-vm kernel: Memory cgroup out of memory: Killed process 3974 (apk) total-vm:20600kB, anon-rss:7508kB, file-rss:588kB, shmem-rss:0kB, UID:0 pgtables:80kB oom_score_adj:0
May 17 19:54:29 ubuntu-vm systemd[1]: docker-3193584578ea07bf5ee503289e7fe7b69f20e60ad236b2b5ce2274419856f8f3.scope: A process of this unit has been killed by the OOM killer.
```


Bonus:
```
dan:~ % sudo apt-get install bpfcc-tools linux-headers-$(uname -r)
```

```
/ # apk update                        │ dan:~ % sudo /sbin/oomkill-bpfcc
fetch https://dl-cdn.alpinelinux.org/a│ [sudo] password for dan:
fetch https://dl-cdn.alpinelinux.org/a│ Tracing OOM kills... Ctrl-C to stop.
Killed                                │ 20:00:44 Triggered by PID 4241 ("apk"), OOM kill of PID 4241 ("apk"), 4096 pages, loadavg: 0.44 0.13 0.04 4/199 4242
/ #                                   │
```

