# OS lab2 report

苏克

2014011402

计45班


## 练习0

采用meld，将lab1项目中的kdebug.c和trap.c复制到lab2项目中。在WSL下，需要开启X-server才能启动meld的GUI窗口，这里我采用的是Xming。

## 练习1

练习1要求实现first fit内存分配算法，包括块的初始化、内存的分配和回收三部分。

### 初始化：default_init_memmap

初始化时，对每个内存块p (范围是[base, base + n)]) 做SetPageProperty(p)处理。这里值得一提的是，这个函数的Property指的是p->flags的取值PG_property，取值为1代表可以被分配；为0代表不能被分配，和p->property完全不同，不能混淆。除了flags之外，还需要把p->ref清零（表示未被访问过），把p->property置为0，表示并非头结点（但是头节点的该域应该设置为块的总数n，只要特殊处理即可）。

在设置好初始值之后，还需要把page添加进链表中，调用list_add_before函数即可。

### 分配：default_alloc_pages

由于我们要实现的是first fit，所以只要遍历free_list，找到第一个长度>=n的块，然后从这个块的首页开始分配出去n个页即可。分配块p的做法是

```
SetPageReserved(p);
ClearPageProperty(p);
list_del(&(p->page_link));
```

在分配过后，需要考虑如果这一块尚有剩余，需要把剩余的部分（长度为page->property - n）留存下来。我们所采用的方式是将“下一个”块的块首（它本来不是块首）的property设置为剩余部分的值。只要有剩余长度，就一定存在这样的新块首。

### 释放：default_free_pages

释放的过程则稍显繁琐。首先，我们需要找到被释放的块们所应该插入的地方。由于所提供的插入函数是list_add_before，所以我们遍历链表，直到找到一个地址高于所释放地址的页，在其前面插入n个页。这n个页也需要经过初始化处理。

在插入释放的页之后，需要做一个向前和向后合并的工作。首先我们向后（向高地址方向）合并：将所插入页的property加上后一个块的property，再把后一个块首页的property设为0，表示后一个块被我们新释放的块合并了。之后再向前（低地址方向）合并：在链表上向前走（直到遭遇链表头为止），用所遇到的第一个块合并我们新插入的块（因为只有低地址合并高地址）。

这样便完成了练习1的要求，make qemu之后可以

值得一提的是，为了调试，我写了一个check_list函数，检查当前free_list中的块首页。这也是我和所谓答案代码很大的不同……

### 改进空间

first-fit并不关心是否能最合理地分配内存，不在乎分配完成后是否产生了不必要的碎片。可以做这样的改进：在分配内存时选择一个长度最大的块进行分配，防止小段碎片的过多产生。

## 练习2

### get_pte实现

练习2要求实现get_pte函数，该函数的作用是给出一个虚地址，返回该虚地址对应的PTE的虚地址。如果该虚地址所对应的PTE不存在，则需要新建一个（利用练习1中实现好的工具新建page即可）。

我们首先看一下对于一个虚地址（uintptr_t）la，如何将其划分为PDI-PTI-OFFSET三段。在mmu.h文件中，有注释说明了这一点：

```c
// A linear address 'la' has a three-part structure as follows:
//
// +--------10------+-------10-------+---------12----------+
// | Page Directory |   Page Table   | Offset within Page  |
// |      Index     |     Index      |                     |
// +----------------+----------------+---------------------+
//  \--- PDX(la) --/ \--- PTX(la) --/ \---- PGOFF(la) ----/
//  \----------- PPN(la) -----------/
//
// The PDX, PTX, PGOFF, and PPN macros decompose linear addresses as shown.
// To construct a linear address la from PDX(la), PTX(la), and PGOFF(la),
// use PGADDR(PDX(la), PTX(la), PGOFF(la)).
```

在lab2中virtual address和linear address是等价的。因此我们只要通过在PDT中索引PDX(la)，得到页表索引PDE，再在得到的PDE中索引PTX(la)得到PTE的地址。

在get_pte函数的注释处，值得一提的是在访问物理地址时都需要加上KADDR()。在实验书中我们已经看到了这一做法的用意：实际上只是把内核虚地址减去一个偏移量KERNBASE，其值在memlayout.h中定义为0xC0000000，与实验书的介绍一致。

知道了KADDR这一点就不会遇到太大的困难，我们可以根据注释的指引实现这个过程：

```c
pde_t *pdep = &pgdir[PDX(la)];  // (1) find page directory entry
if ((*pdep & PTE_P) == 0) {              // (2) check if entry is not present
                      // (3) check if creating is needed, then alloc page for page table
    if (!create) // check
        return NULL;
                      // CAUTION: this page is used for page table, not for common data page
    struct Page* newpg = alloc_page(); // alloc page for page table
    set_page_ref(newpg, 1); // (4) set page reference
    uintptr_t pa = page2pa(newpg); // (5) get linear address of page
    memset(KADDR(pa), 0, PGSIZE); // (6) clear page content using memset
    *pdep = pa | PTE_U | PTE_P | PTE_W; // (7) set page directory entry's permission
}

return &((pte_t*)KADDR(PDE_ADDR(*pdep)))[PTX(la)]; // (8) return page table entry
```

注意最后一步，我们首先要从表项*pdep中通过宏PDE_ADDR()读取出PDE的地址，为了访问该物理地址又要套上一层KADDR。这里我一开始忘记了KADDR的事情，爆炸了很久，输出调试才发现问题……

### PDE和PTE组成部分的含义

PDE的每一项是一个PTE。PTE的每一项是物理地址（同时也是线性地址），带有一些标志位PTE_P，PTE_W，PTE_U。

有两层页表（PDE-PTE）可以加快虚地址转换过程，可以提升ucore运行效率。

### 页访问异常的处理

发生页访问一场时，硬件应当保存相关参数，转入中断处理，进行页的换入换出。进行完之后返回断点处继续执行。

## 练习3

练习3执行的是练习2的逆过程，即释放某个虚地址，并且取消对应PTE的映射。这个过程比较简单，根据注释的指引实现代码如下：

```c
if ((*ptep & PTE_P) == 1) { // (1) if pte is present
    struct Page* page = pte2page(*ptep); // (2) find corresponding page to pte
    set_page_ref(page, page->ref - 1); // (3)decrease page reference
    if (page->ref == 0) { // (4)and free this page when page reference reachs 0
        free_page(page);
    }
    *ptep = 0; // (5)clear second page table entry
    tlb_invalidate(pgdir, la); // (6)flush tlb
}
```

其中tlb_invalidate这一步操作在上面的注释中已经进行了介绍。

### Page的全局变量和PTE/PDE的关系

数组中每个Page项代表一个物理页。在pmm.h中的pte2page函数中可以看到（一组）PDE和PTE与page的对应关系：

```c
static inline struct Page *
pa2page(uintptr_t pa) {
    if (PPN(pa) >= npage) {
        panic("pa2page called with invalid pa");
    }
    return &pages[PPN(pa)];
}

/// 中略

static inline struct Page *
pte2page(pte_t pte) {
    if (!(pte & PTE_P)) {
        panic("pte2page called with invalid pte");
    }
    return pa2page(PTE_ADDR(pte));
}
```

可以看到page[i]的值也就是第i个物理页的首地址，之后根据Page结构体的定义可以索引出首地址附近（物理页的头）附近储存的ref、property等信息。

### 如何使得虚拟地址与物理地址相等

将memlayout.h中的宏KERNBASE从0xC0000000改为0x0即可。根据实验书的介绍，还有tools/kernel.ld中有一个常量"."需要从0xC0100000修改为0x00000000。

## 总结

本实验中重要的知识点是内存管理机制和段页式管理。通过编程实践，我对基础的first-fit内存管理机制和段页式管理机制都有了较深的理解（尤其是基础的内存管理机制，由于我用了大量的时间使用check_list函数进行输出调试，对整个check过程十分熟悉，也算是有了更深的认识）。

感谢助教老师的辛勤工作。之前我的报告中使用了大量的图片，刚刚注意到报告是文字为主的，十分抱歉……之后不会乱插图片了。
