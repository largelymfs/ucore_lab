##Lab2实验报告

###1. 实现 first-fit 连续物理内存分配算法（需要编程）

1.实现过程：

* 原本的实现没有按照地址的大小进行维护，这里我们的修改就是要整个链表是按照起始地址有序的。
* 在分配算法中，原来的做法是简简单单的删除原来的block，如果还有新的空间剩下来就在链表尾部添加一个新的block。在这里我们修改之后是将剩余空间的block插入到链表中原来这段block所在的位置，具体的是将下面的代码进行替换：

```
list_del(&(page->page_link));
if (page->property > n) {
    struct Page *p = page + n;
    p->property = page->property - n;
    list_add(&free_list, &(p->page_link));
}
nr_free -= n;
ClearPageProperty(page);
```

替换成下面的一段代码：

```
/// Code By Largelymfs
list_entry_t * prev = list_prev(le);
list_del(&(page->page_link));
if (page->property > n){
    struct Page *p = page + n;
    p->property = page->property - n;
    SetPageProperty(p);
    list_add_after(prev, &(p->page_link));
};
nr_free -= n;
ClearPageProperty(page);
```

具体的执行流程是删除当前的空闲块，如果有剩余的内存，就新建一个页，将`property`设置为原来的`property`-n，然后将`property`设置为1。加入到原来的内存块所在的位置。最后更新`nr_free`和之前页的`property`。

* 在释放算法中，同样的我们不需要每次合并之后直接在链表尾部添加新的block，我们需要找出新block所在的位置，然后针对前后进行判断，决定出合并的结果，最后插入到这个位置就行。具体的代码实现是:

```
//Code By Largelymfs
//Find the correct position
while (le != &free_list){
	p = le2page(le, page_link);
	if (base + base->property <= p) break;
	le = list_next(le);
}
//Describe the position
list_entry_t * prev = list_prev(le);
list_entry_t * next = le;
struct Page* prevPage = le2page(prev, page_link);
struct Page* nextPage = le2page(next, page_link);

if ((prevPage != NULL) && (prevPage + prevPage->property == base)){
	prevPage->property += base->property;
	base->property = 0;
	ClearPageProperty(base);
	base = prevPage;
}else{
	list_add_after(prev, &(base->page_link));
}

if ((nextPage != NULL) && (base + base->property == nextPage)){
	base->property += nextPage->property;
	ClearPageProperty(nextPage);
	nextPage->property = 0;
	list_del(&(nextPage->page_link));
}
nr_free += n;
```

这样就解决了问题。具体的执行过程是首先找到需要释放块所处链表中的位置，然后判断是否能和前面一块合并，能否和后面一块合并，最后维护链表的性质即可。

2.如何改进我的First-Fit算法

我的First-Fit算法目前在查找的时候还是在利用双向列表，但是这时候查找的时间复杂度是O(n)，如果我们能够使用平衡二叉树来对空闲块进行维护，这样可以将时间复杂度降低成O(logN)。

###2：实现寻找虚拟地址对应的页表项（需要编程）

1. 实现算法过程：

* 首先从虚拟地址中获取PDE，然后根据PDT的首地址读出PDE的内容。
* 判断PDE的存在位判断是否需要创建一个页。
* 如果创建新的页我们需要创建一个物理页同时将这个页中全部清成0，同时设置PDE中得存在位和里面的内容。
* 然后按照PDE检索到二级页表的虚地址，进行返回。

2.请描述页目录项（Pag Director Entry）和页表（Page Table Entry）中每个组成部分的含义和以及对ucore而言的潜在用处。

* PDE的组成：二级页表的初始地址，是否存在，是否可读写，用户权限。
	* 二级页表的初始地址是为了让操作系统能够访问二级页表的内容。
	* 是否存在位是检测当前页是否存在，方便进行添加和删除。
	* 是否读写位是表示当前进程是否有读写的权限。
	* 用户权限位是表示当前进程是否有权限进行操作。
* PTE的组成：物理内存的初始地址，是否存在，是否可读写，用户权限。
	* 物理内存的初始地址是为了进行物理内存的访问。
	* 附加位的作用同上一个问题。

3.如果ucore执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？

这个时候MMU会触发了Page Fault异常，等待操作系统将请求的页从外存换入。


###3：释放某虚地址所在的页并取消对应二级页表项的映射（需要编程）

1. 实现算法过程：

* 首先判断PTE是否是存在的，如果不存在直接退出。
* 获得这页，将引用-1，如果引用是0，释放这个页。
* 将PTE的位置设置为不存在
* 在TLB中消除

2.数据结构Page的全局变量（其实是一个数组）的每一项与页表中的页目录项和页表项有无对应关系？如果有，其对应关系是啥？

Page表示的是所有的页，这里面的页分成两个部分：一个部分是最普通的页，另外一个部分用来存放页表。其中页表的每一项都对应一个普通的页。

3.如果希望虚拟地址与物理地址相等，则需要如何修改lab2，完成此事？ 鼓励通过编程来具体完成这个问题

我们可以在段机制的过程中使用对等映射，而不是使用现在的段映射。


###4：本实验和答案的不同

* 对于实验一，我的实现方法与标准答案有很大的不同，在标准答案中，`freelist`里面有每一个空闲的页，而我的实现中没有中间的内存块，而是对于一个连续的N连块，我们只存储有一个节点，所以在很多设置位的时候我的实现只需要对连续块中地址最小的一块进行处理，而并非像答案一样设置很多页的标志位。此外在释放的过程中，答案需要扫过所有的节点，我们只需要扫到合适的插入位置即可。

* 对于实验二和实验三，我的实现方法和标准答案一致，只有某些语句的表达方式不一致。

###5：本实验知识点

* 连续物理内存分配的First Fit 和 Best Fit 算法。
* 非连续物理内存分配的段机制和页机制算法。这里主要是建立页表，建立页表项和释放页表项的过程。