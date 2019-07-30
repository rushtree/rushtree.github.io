# background

最近遇到一个问题，微服务的进程virt很高，达到10g+，实际占用内存还不到1g。

# 为什么java进程virt如此高？

因为线程过多，一个进程的线程有100+。
````
top -Hp pid
````

# 为什么线程多会导致virt如此高？

pmap查看内存映射，发现有许多64MB的内存块，查了资料，发现这是glibc 2.10版本之后，引入的新特性，会为每个线程分配一个64MB的内存池。根据pmap的输出结果，计算了一下，这些内存块的数量和线程的数量是吻合的。这部分线程占用了7G+的virt。这些内存地址空间虽然分配了，但是实际上使用的内存并不大，rss和pss很多都是0kb。这也解释了为什么virt如此高，实际占用的内存却不高。
查看环境上glibc的版本，是2.22。
`````
pmap pid

ldd --version
`````

# ref
https://www.ibm.com/developerworks/community/blogs/kevgrig/entry/linux_glibc_2_10_rhel_6_malloc_may_show_excessive_virtual_memory_usage?lang=en
(居然是12年的文章，久远不久远 ><)
