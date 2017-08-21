---
layout: post
title:  "记一次java进程被linux杀掉的排查过程"
date:   2017-05-18 15:12:12 +0800
categories: Java
author: 小米粒
keywords: java, linux, oom killer
description: 
---

> 本文聚焦于一次线上java进程被linux杀掉的排查过程

某次发现线上有个机器java进程突然挂掉了，这种情况一般都是linux 的oom killer杀掉的（当系统内存不足时oom killer会基于一套算法选出一个进程并杀掉），于是查看/var/log/messages文件，发现以下日志：

```java:n
May 17 17:00:28 localhost kernel: java invoked oom-killer: gfp_mask=0x201da, order=0, oom_adj=0, oom_score_adj=0
May 17 17:00:28 localhost kernel: java cpuset=/ mems_allowed=0
May 17 17:00:28 localhost kernel: Pid: 3794, comm: java Not tainted 2.6.32-642.11.1.el6.x86_64 #1
May 17 17:00:28 localhost kernel: Call Trace:
May 17 17:00:28 localhost kernel: [<ffffffff81131420>] ? dump_header+0x90/0x1b0
May 17 17:00:28 localhost kernel: [<ffffffff8123c04c>] ? security_real_capable_noaudit+0x3c/0x70
May 17 17:00:28 localhost kernel: [<ffffffff811318a2>] ? oom_kill_process+0x82/0x2a0
May 17 17:00:28 localhost kernel: [<ffffffff811317e1>] ? select_bad_process+0xe1/0x120
May 17 17:00:28 localhost kernel: [<ffffffff81131ce0>] ? out_of_memory+0x220/0x3c0
May 17 17:00:28 localhost kernel: [<ffffffff8113e6bc>] ? __alloc_pages_nodemask+0x93c/0x950
May 17 17:00:28 localhost kernel: [<ffffffff8117794a>] ? alloc_pages_current+0xaa/0x110
May 17 17:00:28 localhost kernel: [<ffffffff8112e817>] ? __page_cache_alloc+0x87/0x90
May 17 17:00:28 localhost kernel: [<ffffffff8112e1fe>] ? find_get_page+0x1e/0xa0
May 17 17:00:28 localhost kernel: [<ffffffff8112f7b7>] ? filemap_fault+0x1a7/0x500
May 17 17:00:28 localhost kernel: [<ffffffff811591b4>] ? __do_fault+0x54/0x530
May 17 17:00:28 localhost kernel: [<ffffffff810bb686>] ? futex_wait+0x1e6/0x310
May 17 17:00:28 localhost kernel: [<ffffffff81159787>] ? handle_pte_fault+0xf7/0xb20
May 17 17:00:28 localhost kernel: [<ffffffff810aab70>] ? hrtimer_wakeup+0x0/0x30
May 17 17:00:28 localhost kernel: [<ffffffff810abb84>] ? hrtimer_start_range_ns+0x14/0x20
May 17 17:00:28 localhost kernel: [<ffffffff8115a449>] ? handle_mm_fault+0x299/0x3d0
May 17 17:00:28 localhost kernel: [<ffffffff81052156>] ? __do_page_fault+0x146/0x500
May 17 17:00:28 localhost kernel: [<ffffffff81015866>] ? native_read_tsc+0x6/0x20
May 17 17:00:28 localhost kernel: [<ffffffff8100ba4e>] ? common_interrupt+0xe/0x13
May 17 17:00:28 localhost kernel: [<ffffffff81046f28>] ? pvclock_clocksource_read+0x58/0xd0
May 17 17:00:28 localhost kernel: [<ffffffff8154f09e>] ? do_page_fault+0x3e/0xa0
May 17 17:00:28 localhost kernel: [<ffffffff8154c3a5>] ? page_fault+0x25/0x30
May 17 17:00:28 localhost kernel: Mem-Info:
May 17 17:00:28 localhost kernel: Node 0 DMA per-cpu:
May 17 17:00:28 localhost kernel: CPU    0: hi:    0, btch:   1 usd:   0
May 17 17:00:28 localhost kernel: CPU    1: hi:    0, btch:   1 usd:   0
May 17 17:00:28 localhost kernel: CPU    2: hi:    0, btch:   1 usd:   0
May 17 17:00:28 localhost kernel: CPU    3: hi:    0, btch:   1 usd:   0
May 17 17:00:28 localhost kernel: Node 0 DMA32 per-cpu:
May 17 17:00:28 localhost kernel: CPU    0: hi:  186, btch:  31 usd:   0
May 17 17:00:28 localhost kernel: CPU    1: hi:  186, btch:  31 usd:   0
May 17 17:00:28 localhost kernel: CPU    2: hi:  186, btch:  31 usd:   0
May 17 17:00:28 localhost kernel: CPU    3: hi:  186, btch:  31 usd:   0
May 17 17:00:28 localhost kernel: Node 0 Normal per-cpu:
May 17 17:00:28 localhost kernel: CPU    0: hi:  186, btch:  31 usd:   0
May 17 17:00:28 localhost kernel: CPU    1: hi:  186, btch:  31 usd:   0
May 17 17:00:28 localhost kernel: CPU    2: hi:  186, btch:  31 usd:   0
May 17 17:00:28 localhost kernel: CPU    3: hi:  186, btch:  31 usd:   0
May 17 17:00:28 localhost kernel: active_anon:1951791 inactive_anon:128 isolated_anon:0
May 17 17:00:28 localhost kernel: active_file:222 inactive_file:168 isolated_file:0
May 17 17:00:28 localhost kernel: unevictable:0 dirty:0 writeback:0 unstable:0
May 17 17:00:28 localhost kernel: free:25773 slab_reclaimable:5075 slab_unreclaimable:8282
May 17 17:00:28 localhost kernel: mapped:320 shmem:134 pagetables:5575 bounce:0
May 17 17:00:28 localhost kernel: Node 0 DMA free:15752kB min:124kB low:152kB high:184kB active_anon:0kB inactive_anon:0kB active_file:0kB inactive_file:0kB unevictable:0kB isolated(anon):0kB isolated(file):0kB present:15364kB mlocked:0kB dirty:0kB writeback:0kB mapped:0kB shmem:0kB slab_reclaimable:0kB slab_unreclaimable:0kB kernel_stack:0kB pagetables:0kB unstable:0kB bounce:0kB writeback_tmp:0kB pages_scanned:0 all_unreclaimable? yes
May 17 17:00:28 localhost kernel: lowmem_reserve[]: 0 3000 8050 8050
May 17 17:00:28 localhost kernel: Node 0 DMA32 free:45304kB min:25140kB low:31424kB high:37708kB active_anon:2738172kB inactive_anon:76kB active_file:576kB inactive_file:168kB unevictable:0kB isolated(anon):0kB isolated(file):0kB present:3072096kB mlocked:0kB dirty:0kB writeback:0kB mapped:768kB shmem:76kB slab_reclaimable:6140kB slab_unreclaimable:2964kB kernel_stack:1088kB pagetables:6480kB unstable:0kB bounce:0kB writeback_tmp:0kB pages_scanned:832 all_unreclaimable? no
May 17 17:00:28 localhost kernel: lowmem_reserve[]: 0 0 5050 5050
May 17 17:00:28 localhost kernel: Node 0 Normal free:42036kB min:42316kB low:52892kB high:63472kB active_anon:5068992kB inactive_anon:436kB active_file:312kB inactive_file:504kB unevictable:0kB isolated(anon):0kB isolated(file):0kB present:5171200kB mlocked:0kB dirty:92kB writeback:0kB mapped:512kB shmem:460kB slab_reclaimable:14160kB slab_unreclaimable:30164kB kernel_stack:7360kB pagetables:15820kB unstable:0kB bounce:0kB writeback_tmp:0kB pages_scanned:1504 all_unreclaimable? no
May 17 17:00:28 localhost kernel: lowmem_reserve[]: 0 0 0 0
May 17 17:00:28 localhost kernel: Node 0 DMA: 2*4kB 2*8kB 1*16kB 1*32kB 1*64kB 0*128kB 1*256kB 0*512kB 1*1024kB 1*2048kB 3*4096kB = 15752kB
May 17 17:00:28 localhost kernel: Node 0 DMA32: 8571*4kB 1393*8kB 5*16kB 0*32kB 0*64kB 0*128kB 0*256kB 0*512kB 0*1024kB 0*2048kB 0*4096kB = 45508kB
May 17 17:00:28 localhost kernel: Node 0 Normal: 756*4kB 728*8kB 552*16kB 336*32kB 134*64kB 37*128kB 2*256kB 0*512kB 0*1024kB 0*2048kB 0*4096kB = 42256kB
May 17 17:00:28 localhost kernel: 612 total pagecache pages
May 17 17:00:28 localhost kernel: 0 pages in swap cache
May 17 17:00:28 localhost kernel: Swap cache stats: add 0, delete 0, find 0/0
May 17 17:00:28 localhost kernel: Free swap  = 0kB
May 17 17:00:28 localhost kernel: Total swap = 0kB
May 17 17:00:28 localhost kernel: 2097151 pages RAM
May 17 17:00:28 localhost kernel: 81869 pages reserved
May 17 17:00:28 localhost kernel: 3256 pages shared
May 17 17:00:28 localhost kernel: 1983551 pages non-shared
May 17 17:00:28 localhost kernel: [ pid ]   uid  tgid total_vm      rss cpu oom_adj oom_score_adj name
May 17 17:00:28 localhost kernel: [  549]     0   549     2662      118   1     -17         -1000 udevd
May 17 17:00:28 localhost kernel: [  929]     0   929      373       45   3       0             0 aliyun-service
May 17 17:00:28 localhost kernel: [ 1185]     0  1185    62991     1252   0       0             0 rsyslogd
May 17 17:00:28 localhost kernel: [ 1207]    28  1207   583627     1604   1       0             0 nscd
May 17 17:00:28 localhost kernel: [ 1362]     0  1362    12780      381   0       0             0 ilogtail
May 17 17:00:28 localhost kernel: [ 1368]     0  1368    79135    19704   3       0             0 ilogtail
May 17 17:00:28 localhost kernel: [ 1398]     0  1398    16559      190   2     -17         -1000 sshd
May 17 17:00:28 localhost kernel: [ 1409]    38  1409     6650      141   1       0             0 ntpd
May 17 17:00:28 localhost kernel: [ 1442]     0  1442    29218      164   2       0             0 crond
May 17 17:00:28 localhost kernel: [ 1511]     0  1511     1016       28   1       0             0 mingetty
May 17 17:00:28 localhost kernel: [ 1513]     0  1513     1016       29   0       0             0 mingetty
May 17 17:00:28 localhost kernel: [ 1515]     0  1515     1016       28   0       0             0 mingetty
May 17 17:00:28 localhost kernel: [ 1517]     0  1517     1016       29   1       0             0 mingetty
May 17 17:00:28 localhost kernel: [ 1519]     0  1519     1016       29   1       0             0 mingetty
May 17 17:00:28 localhost kernel: [ 1521]     0  1521     1016       28   2       0             0 mingetty
May 17 17:00:28 localhost kernel: [ 1562]     0  1562     7605       83   0       0             0 gseMaster
May 17 17:00:28 localhost kernel: [ 1563]     0  1563   610826     8059   0       0             0 AgentWorker
May 17 17:00:28 localhost kernel: [16999]     0 16999    29146      162   1       0             0 wrapper
May 17 17:00:28 localhost kernel: [17002]     0 17002   610039    17275   2       0             0 java
May 17 17:00:28 localhost kernel: [27003]     0 27003    11276      275   0       0             0 nginx
May 17 17:00:28 localhost kernel: [16189]     0 16189     2663      123   2     -17         -1000 udevd
May 17 17:00:28 localhost kernel: [16190]     0 16190     2661      117   1     -17         -1000 udevd
May 17 17:00:28 localhost kernel: [31842]   498 31842    68188      817   1       0             0 nxlog
May 17 17:00:28 localhost kernel: [20608]   500 20608  1142219   189539   1       0             0 java
May 17 17:00:28 localhost kernel: [15570]     0 15570    76633     1437   1       0             0 falcon-agent
May 17 17:00:28 localhost kernel: [23498]   497 23498    12343     1373   0       0             0 nginx
May 17 17:00:28 localhost kernel: [23499]   497 23499    12343     1373   3       0             0 nginx
May 17 17:00:28 localhost kernel: [23500]   497 23500    12343     1373   1       0             0 nginx
May 17 17:00:28 localhost kernel: [23501]   497 23501    12709     1684   1       0             0 nginx
May 17 17:00:28 localhost kernel: [24887]     0 24887     6152      172   2       0             0 AliYunDunUpdate
May 17 17:00:28 localhost kernel: [24955]     0 24955    34036     1231   0       0             0 AliYunDun
May 17 17:00:28 localhost kernel: [10825]   496 10825    19251      205   1       0             0 zabbix_agentd
May 17 17:00:28 localhost kernel: [10826]   496 10826    19251      294   3       0             0 zabbix_agentd
May 17 17:00:28 localhost kernel: [10828]   496 10828    19281      260   3       0             0 zabbix_agentd
May 17 17:00:28 localhost kernel: [10829]   496 10829    19281      260   3       0             0 zabbix_agentd
May 17 17:00:28 localhost kernel: [10830]   496 10830    19281      260   3       0             0 zabbix_agentd
May 17 17:00:28 localhost kernel: [10831]   496 10831    19253      244   1       0             0 zabbix_agentd
May 17 17:00:28 localhost kernel: [31995]   500 31995  1994392   889625   0       0             0 java
May 17 17:00:28 localhost kernel: [ 7521]     0  7521    25002      255   2       0             0 sshd
May 17 17:00:28 localhost kernel: [ 7523]   500  7523    25002      256   0       0             0 sshd
May 17 17:00:28 localhost kernel: [ 7524]   500  7524    27120      128   0       0             0 bash
May 17 17:00:28 localhost kernel: [ 7788]   500  7788   848302   814178   0       0             0 vim
May 17 17:00:28 localhost kernel: Out of memory: Kill process 31995 (java) score 442 or sacrifice child
May 17 17:00:28 localhost kernel: Killed process 31995, UID 500, (java) total-vm:7977568kB, anon-rss:3558396kB, file-rss:104kB
```

先看最后一行：

```java:n
 Killed process 31995, UID 500, (java) total-vm:7977568kB, anon-rss:3558396kB, file-rss:104kB
```

进程31995被杀掉了，total-vm是指所有可用内存，anon-rss + file-rss 是该进程实际使用的内存，约8G，这个进程实际使用了3.5GB左右内存，为什么会导致被oom killer杀掉呢？

应该是其他进程导致的，我注意到其中有个vim进程（7788），占用的rss内存页个数是814178，内存页大小为4KB，那么这个vim进程占用了814178*4/1024 = 3180MB，那么原因就很明显了，有人用vim打开了一个很大的文件，导致内存耗尽，最终oom killer登场杀死了java进程。

总结：vim打开文件会一次性加载所有文件内容到内存，打开一个大文件将会非常消耗内存。如果有需要，建议用less命令来查看文件。

