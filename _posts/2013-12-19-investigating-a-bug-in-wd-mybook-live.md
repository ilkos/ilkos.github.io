---
layout: post
title: "Debugging WD MyBook Live network and DLNA issues"
description: ""
category: ""
tags: ["bug", "programming", "NAS", "smb", "dlna"]
---
{% include JB/setup %}

Adventures with a Western Digital MyBook Live device -
a 6TB network attached storage device that is packed with nice features and functionality such as RAID (0 / 1) and DLNA streaming.

## The issue
Despite having performed multiple firmware upgrades and getting a replacement device the SMB connectivity was unstable and the DLNA streaming server
would crash a few minutes after rebooting the device - two seemingly unrelated problems.

## Debugging the DLNA daemon
Thankfully the device has a hidden option to provide root access via ssh - so it was possible to take a look under the hood.

After enabling ssh, first stop was to figure out what was going on with the DLNA streaming.

{% highlight bash %}
MyBookLiveDuo:~# netstat -anp | grep tcp
...
tcp        0      0 0.0.0.0:2869            0.0.0.0:*               LISTEN      27441/dms_smm
...

{% endhighlight %}

The process running on port 2869 was the one that stood out as the most likely to be part of the DLNA daemon.
Indeed, the DLNA devices connected to it and even though it was running when the device booted, it mysteriously disappeared
after a while (along with the abrupt ending of any streaming). Knowing the binary and being able to reproduce is as good as figuring the problem out:

{% highlight bash %}
MyBookLiveDuo:/var/log/samba# gdb `which dms_smm`
GNU gdb (GDB) 7.0.1-debian
Copyright (C) 2009 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "powerpc-linux-gnu".
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>...
Reading symbols from /usr/local/bin/dms_smm...done.
(gdb) r
Starting program: /usr/local/bin/dms_smm
[Thread debugging using libthread_db enabled]
DMS root directory : /usr/local/dlna-access/DMS/media
DMS config directory : /usr/local/dlna-access/xml
DMS description file name : dms_descr.xml
[New Thread 0x4884f490 (LWP 14940)]


========================================
NFLC SDK version 2.3.0
DMS_SMM  version 2.3.0.12093
========================================
Enabled features:
----------------------------------------
Media Processing Extension (MPE)
XMMRR support
User agent customizations support
Subtitle support
Embedded subtitle support (MPE)
QOS support
JPEG_SM res creation
XML runtime generation
Search support
FIFO mechanism for storage adding and removing enabled
Parental guidance
========================================

[New Thread 0x4904f490 (LWP 14941)]
[New Thread 0x4984f490 (LWP 14942)]

Program received signal SIGSEGV, Segmentation fault.
[Switching to Thread 0x4884f490 (LWP 14940)]
0x100ee488 in _jpeg_rescale_routine_whole_image ()
(gdb) bt
#0  0x100ee488 in _jpeg_rescale_routine_whole_image ()
#1  0x100c03c8 in media_create_reference ()
#2  0x10096740 in contents_index_create_reference_file ()
#3  0x10096c5c in contents_index_create_reference ()
#4  0x100973c8 in contents_index_storage_create_references ()
#5  0x1008e6fc in contents_index_storage_travel ()
#6  0x0ffc6528 in start_thread () from /lib/libpthread.so.0
#7  0x0fe0e6f0 in clone () from /lib/libc.so.6
(gdb)
{% endhighlight %}

It looks like the dms_smm binary (apart from dealing with the streaming of data) deals with the generation of the thumbnails of
the disk media content in a separate thread. But an issue (most likely with a corrupted file in the disk) crashes the jpeg rescaling code
and results in the whole daemon being taken down.

## Debugging SMB
A corner-case exception handling bug in non-generic software is one thing, but smbd issues are much more interesting -
even though there may be bugs from time to time, a systematically reproducible scenario like this was bound to be seen in the wild given the user base
of software as popular.

The first stop was naturally the samba logs in /var/log/:
{% highlight bash %}
[2013/08/14 17:46:56.536355,  0] lib/fault.c:48(fault_report)
  INTERNAL ERROR: Signal 7 in pid 31499 (3.6.5)
  Please read the Trouble-Shooting section of the Samba3-HOWTO
[2013/08/14 17:46:56.537915,  0] lib/fault.c:50(fault_report)
  
  From: http://www.samba.org/samba/docs/Samba3-HOWTO.pdf
[2013/08/14 17:46:56.538702,  0] lib/fault.c:51(fault_report)
  ===============================================================
[2013/08/14 17:46:56.539510,  0] lib/util.c:1117(smb_panic)
  PANIC (pid 31499): internal error
[2013/08/14 17:46:56.580203,  0] lib/util.c:1221(log_stack_trace)
  BACKTRACE: 31 stack frames:
   #0 /usr/sbin/smbd(log_stack_trace+0x40) [0x20ba2634]
   #1 /usr/sbin/smbd(smb_panic+0x7c) [0x20ba2780]
   #2 /usr/sbin/smbd(iface_count+0) [0x20b8fae4]
   #3 [0x100344]
   #4 [0x1]
   #5 /usr/lib/powerpc-linux-gnu/libtdb.so.1(+0x8cfc) [0x20158cfc]
   #6 /usr/lib/powerpc-linux-gnu/libtdb.so.1(+0x9470) [0x20159470]
   #7 /usr/lib/powerpc-linux-gnu/libtdb.so.1(+0xc914) [0x2015c914]
   #8 /usr/lib/powerpc-linux-gnu/libtdb.so.1(+0x6d54) [0x20156d54]
   #9 /usr/lib/powerpc-linux-gnu/libtdb.so.1(+0x6e0c) [0x20156e0c]
   #10 /usr/lib/powerpc-linux-gnu/libtdb.so.1(tdb_chainlock+0x44) [0x20157090]
   #11 /usr/sbin/smbd(+0x4b84c4) [0x20b884c4]
   #12 /usr/sbin/smbd(connections_fetch_entry+0xc4) [0x20bafe7c]
   #13 /usr/sbin/smbd(claim_connection+0x80) [0x208552c4]
   #14 /usr/sbin/smbd(+0x1f8988) [0x208c8988]
   #15 /usr/sbin/smbd(+0x1f92a8) [0x208c92a8]
   #16 /usr/sbin/smbd(make_connection+0x5c0) [0x208c98b4]
   #17 /usr/sbin/smbd(reply_tcon_and_X+0x354) [0x208821e8]
   #18 /usr/sbin/smbd(+0x1f1ec0) [0x208c1ec0]
   #19 /usr/sbin/smbd(+0x1f5894) [0x208c5894]
   #20 /usr/sbin/smbd(+0x1f5b84) [0x208c5b84]
   #21 /usr/sbin/smbd(+0x1f5bf8) [0x208c5bf8]
   #22 /usr/sbin/smbd(run_events_poll+0x484) [0x20bb3928]
   #23 /usr/sbin/smbd(smbd_process+0x970) [0x208c5580]
   #24 /usr/sbin/smbd(+0x77b0a4) [0x20e4b0a4]
   #25 /usr/sbin/smbd(run_events_poll+0x484) [0x20bb3928]
   #26 /usr/sbin/smbd(+0x4e3de8) [0x20bb3de8]
   #27 /usr/sbin/smbd(_tevent_loop_once+0xc4) [0x20bb4378]
   #28 /usr/sbin/smbd(main+0x1234) [0x20e4c864]
   #29 /lib/libc.so.6(+0x1f63c) [0x1ff5f63c]
   #30 /lib/libc.so.6(+0x1f800) [0x1ff5f800]
[2013/08/14 17:46:56.592742,  0] lib/fault.c:372(dump_core)
  dumping core in /var/log/samba/cores/smbd
{% endhighlight %}

smbd was also crashing! Not only that, but the smbd version was a stable production one, with no reports of similar stack traces.
If we eliminate the possibility of memory / hardware faults because the device had already been replaced by the WD support, then we are left either
with a case of very bad luck, or with the two issues being somehow connected.

If we take a closer look at the smbd logs before the core dump, we notice (up to several minutes before):

{% highlight bash %}
[2013/12/19 18:15:18.029862,  0] ../lib/util/tdb_wrap.c:65(tdb_wrap_log)
  tdb(/var/run/samba/connections.tdb): expand_file write of 8192 bytes failed (No space left on device)
[2013/12/19 18:15:18.030831,  0] smbd/connection.c:164(claim_connection)
  claim_connection: tdb_store failed with error NT_STATUS_UNSUCCESSFUL.
[2013/12/19 18:15:18.390960,  0] ../lib/util/tdb_wrap.c:65(tdb_wrap_log)
  tdb(/var/run/samba/connections.tdb): expand_file write of 8192 bytes failed (No space left on device)
[2013/12/19 18:15:18.391486,  0] smbd/connection.c:164(claim_connection)
  claim_connection: tdb_store failed with error NT_STATUS_UNSUCCESSFUL.
{% endhighlight %}

So apparently the problem is around libtdb (which is a lightweight database library used commonly in smbd).
We are failing to write data properly and do not handle the issue gracefully resulting in a corruption that further leads to read issues,
and this cascades to a segfault. But we have a very good hint as to why this happens: "No space left on device".

{% highlight bash %}
MyBookLiveDuo:/var# ls -l run
lrwxrwxrwx 1 root root 4 Sep 16 21:52 run -> /tmp

MyBookLiveDuo:/tmp# df
Filesystem           1K-blocks      Used Available Use% Mounted on
/dev/md1               1968336    615504   1252844  33% /
tmpfs                     5120         0      5120   0% /lib/init/rw
udev                     10240      1216      9024  12% /dev
tmpfs                     5120         0      5120   0% /dev/shm
tmpfs                   102400    102400         0 100% /tmp
ramlog-tmpfs             20480      8832     11648  44% /var/log
/dev/md3             5828537152 337732608 5490804544   6% /DataVolume
{% endhighlight %}

So samba is serialising the libtdb data in /tmp which is full. Upon further inspection, we can see that all the space in /tmp is taken by 3 files
with a JPG prefix and a seemingly random suffix:
{% highlight bash %}
-rw------- 1 root       root       31795968 Dec 19 18:49 JPG17Si53T
-rw------- 1 root       root       33685504 Dec 19 18:49 JPG2irQTXD
-rw------- 1 root       root       34979328 Dec 19 18:49 JPG3lv6tTn
{% endhighlight %}

If we delete these files, purge the /var/run/samba directory and restart smbd,
we will no longer have any issues with smbd and file sharing will work properly.

But how did these files end up being there in the first place?!
It turns out that we are not out of luck: these files are leftovers from the crashing dms_smm process which is taking down most of the system functionality with it because of the /tmp filesystem interactions.

## Conclusion
1. If you are experiencing a similar issue, you now know how to resolve it (at least partially)
2. Bug free software is hard - handling exceptional cases is difficult, and debugging them can be quite interesting :)

