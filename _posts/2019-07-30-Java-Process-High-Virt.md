# background

最近遇到一个问题，微服务的进程virt很高，达到10g+，实际占用内存还不到1g。

# 为什么java进程virt如此高？

因为线程过多，一个进程的线程有100+。
````
top -Hp pid
top - 11:18:39 up 12 days, 12:46, 15 users,  load average: 1.44, 1.78, 1.80
Threads: 119 total,   0 running, 119 sleeping,   0 stopped,   0 zombie
%Cpu(s):  6.2 us,  1.4 sy,  0.0 ni, 92.2 id,  0.0 wa,  0.0 hi,  0.2 si,  0.0 st
KiB Mem:  65567324 total, 64011672 used,  1555652 free,  4706436 buffers
KiB Swap: 33554428 total,        0 used, 33554428 free. 24843352 cached Mem

   PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND                                                                                                                                                         
154756 ossuser   20   0 10.427g 774792  35000 S 0.332 1.182   2:39.14 java                                                                                                                                                            
154758 ossuser   20   0 10.427g 774792  35000 S 0.332 1.182   2:41.53 java                                                                                                                                                            
154759 ossuser   20   0 10.427g 774792  35000 S 0.332 1.182   2:39.81 java  
````

# 为什么线程多会导致virt如此高？

pmap查看内存映射，发现有许多64MB的内存块，查了资料，发现这是glibc 2.10版本之后，引入的新特性，会为每个线程分配一个64MB的内存池。根据pmap的输出结果，计算了一下，这些内存块的数量和线程的数量是吻合的。这部分线程占用了7G+的virt。这些内存地址空间虽然分配了，但是实际上使用的内存并不大，rss和pss很多都是0kb。这也解释了为什么virt如此高，实际占用的内存却不高。
查看环境上glibc的版本，是2.22。
`````
pmap pid
00007f6880062000  65144K      0K      0K      0K ---p 0000000000000000 00:00  [anon]
00007f6884000000    356K    356K    356K    356K rw-p 0000000000000000 00:00  [anon]
00007f6884059000  65180K      0K      0K      0K ---p 0000000000000000 00:00  [anon]
00007f6888000000    316K    316K    316K    316K rw-p 0000000000000000 00:00  [anon]
00007f688804f000  65220K      0K      0K      0K ---p 0000000000000000 00:00  [anon]
00007f688c000000    436K    436K    436K    436K rw-p 0000000000000000 00:00  [anon]
00007f688c06d000  65100K      0K      0K      0K ---p 0000000000000000 00:00  [anon]
00007f6890000000    240K    240K    240K    240K rw-p 0000000000000000 00:00  [anon]
00007f689003c000  65296K      0K      0K      0K ---p 0000000000000000 00:00  [anon]
00007f6894000000    344K    344K    344K    344K rw-p 0000000000000000 00:00  [anon]
00007f6894056000  65192K      0K      0K      0K ---p 0000000000000000 00:00  [anon]
00007f6898000000    364K    364K    364K    364K rw-p 0000000000000000 00:00  [anon]
00007f689805b000  65172K      0K      0K      0K ---p 0000000000000000 00:00  [anon]
00007f689c000000    256K    256K    256K    256K rw-p 0000000000000000 00:00  [anon]
00007f689c040000  65280K      0K      0K      0K ---p 0000000000000000 00:00  [anon]
00007f68a0000000    688K    688K    688K    688K rw-p 0000000000000000 00:00  [anon]
00007f68a00ac000  64848K      0K      0K      0K ---p 0000000000000000 00:00  [anon]
00007f68a4000000    408K    384K    384K    384K rw-p 0000000000000000 00:00  [anon]
00007f68a4066000  65128K      0K      0K      0K ---p 0000000000000000 00:00  [anon]
00007f68a8000000    360K    360K    360K    360K rw-p 0000000000000000 00:00  [anon]
00007f68a805a000  65176K      0K      0K      0K ---p 0000000000000000 00:00  [anon]
00007f68ac000000    132K    128K    128K    128K rw-p 0000000000000000 00:00  [anon]
00007f68ac021000  65404K      0K      0K      0K ---p 0000000000000000 00:00  [anon]
00007f68b0000000    132K     68K     68K     68K rw-p 0000000000000000 00:00  [anon]


ldd --version

ldd (GNU libc) 2.22
Copyright (C) 2015 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
Written by Roland McGrath and Ulrich Drepper.

`````

# ref
https://www.ibm.com/developerworks/community/blogs/kevgrig/entry/linux_glibc_2_10_rhel_6_malloc_may_show_excessive_virtual_memory_usage?lang=en
(居然是12年的文章，久远不久远 ><)
