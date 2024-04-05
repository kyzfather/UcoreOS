## 项目介绍：
Ucore是一个实验性质的基于X86指令集架构的操作系统内核代码。<br>
Ucore是基于MIT的xv6的设计，一步一步完成从一个空空如也到五脏俱全的“麻雀”操作系统。<br>
此“麻雀”包括虚存管理、进程管理、处理器调度、同步互斥、进程间通信等主要内核功能，总的内核代码量（c+asm）不超过5K行，充分体现了小而全的思想。<br>

## 项目收获：
通过Ucore项目，能对操作系统的原理有了更深入的理解，包括从CPU上电开始，通过BootLoader程序为操作系统准备好起始的运行环境，再将UcoreOS内核加载到内存中，权限转交给操作系统，操作系统依次执行各个子系统的初始化工作，整个流程在脑海中有了清晰认识，对操作系统的理解形成了一个体系而不再是分散的知识点。并且对于以下概念有了更深入的理解：<br>
● 为什么堆上内存会发生泄露？<br>
● 用户态和内核态是什么，如何从用户态切换到内核态？<br>
● 零拷贝技术是什么？为什么传统网络内核协议栈处理速度较慢？<br>
● 怎么通过fork复制出一个进程？怎么通过exec加载一个用户程序创建一个用户进程？<br>
● 段页式映射机制是怎么工作的？<br>
● 虚存管理是如何给用户程序提供了一个磁盘大小的内存空间的假象？<br>
● 函数调用怎么创建栈帧？<br>
● ......<br>



## 个人职责和项目实现内容
Ucore本身包括操作系统内核的框架代码，个人需要阅读弄清楚Ucore的框架代码，在此基础上为每个模块实现关键部分代码，具体如下：<br>
● 创建Page结构体管理物理页，实现first-fit连续物理内存分配算法<br>
● 实现根据虚拟地址寻找对应的页表项，以及释放虚地址所在的页并取消对应二级页表项的映射<br>
● 补充完成基于FIFO的页面替换算法，实现缺页异常处理<br>
● 设计实现进程控制块，实现alloc_proc函数分配并初始化一个进程控制块<br>
● 实现do_fork函数调用，为新创建的线程分配资源，实现进程地空间的拷贝<br>
● 实现do_execv函数调用，加载并解析一个处于内存中的ELF执行文件的应用程序，创建用户进程<br>
● 补充完成Round Robin进程调度算法，补充完成进程间同步互斥的信号量机制<br>
● 完善中断初始化和处理，对中断描述符表的初始化工作<br>

## 项目结构介绍
/boot：操作系统的bootloader程序的实现，主要负责为操作系统建立好相应的环境(开启保护模式、初始化段表、设置C语言运行的栈寄存器、探测物理内存)，读取磁盘第一个扇区将ucore的elf格式文件加载到内存，将权限交给OS。<br>
/kern：包含Ucore的主要内核代码，包含各个内核子模块的实现。<br>
/kern/init: 包含Ucore的起始入口地址，调用其他内核子模块完成整个ucore操作系统的初始化。<br>
/kern/mm: 主要包含UCore的内存管理的内核代码，包括物理内存和虚存的管理，建立页式映射机制、实现物理页的换入换出策略。<br>
/kern/process: 进程管理的相关代码，包括进程控制块结构体、进程的复制fork、用户进程创建execv等实现。<br>
/kern/schedule: 进程调度的相关代码，实现round-robin调度算法。<br>
/kern/sync: 主要包含进程同步互斥的相关代码，包括信号量semphore机制的实现。<br>
/kern/trap: 中断和异常处理的相关代码，定义中断描述符表以及每个中断处理程序的地址，负责处理ucore运行过程中遇到的中断和异常。

## 物理内存管理
使用Page结构体来管理4KB的物理页
```
struct Page {
    int ref;        // 当前物理页被多少虚拟地址引用
    uint32_t flags; // 页的状态
    unsigned int property;// 用于first-fit物理页分配,标志当前空闲块包含的物理页数目
    list_entry_t page_link;//链表节点，用于将Page链接到一个双向链表中
};
```

双向链表来链接管理所有的空闲页
```
typedef struct {
            list_entry_t free_list;                        
            unsigned int nr_free; //链表上空闲页的数目                             
} free_area_t;
```

First-Fit物理页分配算法实现，物理页的分配：
```
static struct Page* default_alloc_pages(size_t n) {
    assert(n > 0);
    if (n > nr_free) {
        return NULL;
    }
    list_entry_t *le, *len;
    le = &free_list; //获取双向链表的头节点

    while ((le = list_next(le)) != &free_list) {
        struct Page *p = le2page(le, page_link);
        if (p->property >= n) { //查看当前空闲物理块是否满足需求，如果满足就分配
            int i;
            for (i = 0; i < n; i++) {
                len = list_next(le);
                struct Page *pp = le2page(le, page_link);
                SetPageReserved(pp); //设置物理页状态为已分配
                ClearPageProperty(pp);
                list_del(le); //从链表上移除
                le = len;
            }
            if (p->property > n) { //更改空闲物理块的空闲物理页数目
                (le2page(le, page_link))->property = p->property - n;
            }
            ClearPageProperty(p);
            SetPageReserved(p);
            nr_free -= n;
            return p; //返回结果
        }
    }
    return NULL;
}
```

First-Fit物理页释放，需要考虑内存碎片的问题，合并连续的空闲物理块
```
static void
default_free_pages(struct Page *base, size_t n) {
    assert(n > 0);
    assert(PageReserved(base));

    list_entry_t *le = &free_list;
    struct Page *p;
    while ((le = list_next(le)) != &free_list) {
        p = le2page(le, page_link);
        if (p > base) { // 找到第一个地址大于base的位置处
            break;
        }
    }

    for (p = base; p < base + n; p++) { //将释放的物理页插入到链表中
        list_add_before(le, &(p->page_link));
    }

    // 更改物理页状态：未分配，引用为0
    base->flags = 0;
    set_page_ref(base, 0);
    ClearPageProperty(base); 
    SetPageProperty(base);
    base->property = n;

    // 查看是否能合并后面的物理块
    p = le2page(le, page_link);
    if (base + n == p) { // base + n == p，说明后面有连续的物理块
        base->property += p->property;
        p->property = 0;
    }

    // 查看是否能合并前面的物理块
    le = list_prev(&(base->page_link));
    p = le2page(le, page_link);
    if (le != &free_list && p == base - 1) { //p == base，说明前面有连续的物理块
        while (le != &free_list) {
            if (p->property) {
                p->property += base->property;
                base->property = 0;
                break;
            }
            le = list_prev(le);
            p = le2page(le, page_link);
        }
    }

    nr_free += n;
    return;
}
```
