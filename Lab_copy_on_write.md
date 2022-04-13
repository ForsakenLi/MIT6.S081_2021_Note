# Lab copy on write

## target

本实验要求实现现代OS中常见的COW功能来减少不必要的物理内存拷贝，我们需要针对fork和copyout(将内核态的页拷贝到用户地址空间)两个场景来实现cow。

## solution

### uvmcopy

我们首先需要修改uvmcopy函数，这个函数会将子进程的虚拟地址空间(也就是PTE)映射到父进程，同时需要清理PTE_W位以在后面父/子进程需要修改地址空间时产生一个trap来使操作系统为父子进程真正的分配属于各自的物理内存。

### usertrap

我们需要在这个函数中处理前面因我们设置的`PTE_W=0`而出现的page fault, 根据xv6-book的4.6节我们可以知道, `scause`寄存器中的值指示trap错误的类型，`stval`寄存器包含引起page fault的地址。而此时我们需要处理的是wirte的page fault，根据下图可知，我们需要处理的错误类型号为15:

![img](img/p5.png)

我们需要在usertrap中增加对该类型的page fault处理逻辑，通过stval的值(引起page fault的va)我们可以通过walk函数找到对应的页表项，我们需要重新分配一块物理内存，复制旧的页表项对应的物理地址中的page，然后重设页表项为新的物理页并将权限页加上PTE_W。

### reference count

我们需要在管理物理内存时记录每块物理内存的引用计数，仅在计数为零时才能真正释放掉这块内存空间。我们可以创建一个长度为`PHYSTOP/PGSIZE`的char数组ref_count用于计数，初值为0，对于每个物理地址我们除以PGSIZE就可以得到计数块的下标。对于`kalloc.c`文件，在kalloc初始分配一块内存时将这块内存的引用计数标记为1，在kfree时将引用计数减一，如果计数为0了才会真正释放空间。需要注意的是，因为ref_count数组会被多个CPU上的内核进程共享，所以和空闲链表一样需要在临界区中进行读写。还有一个需要注意的点是由于xv6在启动时会调用kinit()->freerange()来清空内存，此时ref_count仍为初值0，kfree()无法释放计数初值为0的页，因此我们可以在freerange()中修改ref_count的值为1来满足kfree的逻辑。

完成对ref_count的维护后，我们需要修改uvmcopy函数，当一个子进程页绑定到父进程的页时我们需要增加该页对应的物理地址的引用计数，同样在遇到由COW引起的缺页异常时我们需要在页面拷贝完成后为原先的页减少一次引用。

### copyout

