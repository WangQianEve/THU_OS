王倩 2014011319
## 练习0：填写已有实验

本实验依赖实验1。请把你做的实验1的代码填入本实验中代码中有“LAB1”的注释相应部分。提示：可采用diff和patch工具进行半自动的合并（merge），也可用一些图形化的比较/merge工具来手动合并，比如meld，eclipse中的diff/merge工具，understand中的diff/merge工具等。

#### 我的解答
使用diff -B file1 file2找出差异，并将上次的成果复制过来

## 练习1：实现 first-fit 连续物理内存分配算法（需要编程）

在实现first fit 内存分配算法的回收函数时，要考虑地址连续的空闲块之间的合并操作。提示:在建立空闲页块链表时，需要按照空闲页块起始地址来排序，形成一个有序的链表。可能会修改default_pmm.c中的default_init，default_init_memmap，default_alloc_pages， default_free_pages等相关函数。请仔细查看和理解default_pmm.c中的注释。

#### 我的解答
1. 依据注释default_init复用了默认的函数
2. 修改default_init_memmap：依据注释将SetPageProperty(p);和list_add_before(&free_list, &(p->page_link));移到循环中，并改为对每一页都进行操作，其他不变
3. 修改default_alloc_pages：依据注释，只保留一个while循环，如果没有在循环内返回，则返回NULL。循环内代码如下：
```c
list_entry_t *le_next;
struct Page *p = le2page(le, page_link);
if(p->property >= n){
  int i;
  for(i=0;i<n;i++){
    le_next = list_next(le);
    struct Page *pp = le2page(le, page_link);
    SetPageReserved(pp);
    ClearPageProperty(pp);
    list_del(le);
    le = le_next;
  }
  if(p->property>n){
    (le2page(le,page_link))->property = p->property - n;
  }
  ClearPageProperty(p);
  SetPageReserved(p);
  nr_free -= n;
  return p;
}
```
4. 修改default_free_pages：  
  4.1 住需保证base是reserved即可，故加入assert(PageReserved(base));去掉其他代码。  

  4.2 通过以下循环找到段的末尾
  ```c
  while((le=list_next(le)) != &free_list) {
    p = le2page(le, page_link);
    if(p>base){
      break;
    }
  }
  ```
  4.3 通过以下循环插入page到list中
  ```c
  for(p=base;p<base+n;p++){
    list_add_before(le, &(p->page_link));
  }
  ```
  4.3 设置相关属性
  ```c
  base->flags = 0;
  set_page_ref(base, 0);
  ClearPageProperty(base);
  SetPageProperty(base);
  base->property = n;
  ```
  4.4 和后面的部分merge
  ```c
  p = le2page(le,page_link) ;
  if( base+n == p ){
    base->property += p->property;
    p->property = 0;
  }  
  ```
  4.5 和前面的部分merge
  ```c
  le = list_prev(&(base->page_link));
  p = le2page(le, page_link);
  if(le!=&free_list && p==base-1){
    while(le!=&free_list){
      if(p->property){
        p->property += base->property;
        base->property = 0;
        break;
      }
      le = list_prev(le);
      p = le2page(le,page_link);
    }
  }  
  ```

## 练习2：实现寻找虚拟地址对应的页表项（需要编程）

通过设置页表和对应的页表项，可建立虚拟内存地址和物理内存地址的对应关系。其中的get_pte函数是设置页表项环节中的一个重要步骤。此函数找到一个虚地址对应的二级页表项的内核虚地址，如果此二级页表项不存在，则分配一个包含此项的二级页表。本练习需要补全get_pte函数 in kern/mm/pmm.c，实现其功能。请仔细查看和理解get_pte函数中的注释。
#### 我的答案
修改get_pte函数：
1. 依据地址la找到entry
2. 如果entry存在则返回
3. 如果不存在则用alloc_page分配并进行基本的设置

#### 请描述页目录项（Pag Director Entry）和页表（Page Table Entry）中每个组成部分的含义和以及对ucore而言的潜在用处。
页目录项高20位表示页表起始物理地址，低12位表示该页表的标志位
页表项前20位代表对应物理页起始物理地址，第十二位表示该页的一些标志位

#### 如果ucore执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？
硬件会产生中断，然后调用_alltraps(trapentry.s)，然后调用trap(trap.c)，然后调用trap_dispatch(trap.c)，然后panic!!!

## 练习3：释放某虚地址所在的页并取消对应二级页表项的映射（需要编程）

当释放一个包含某虚地址的物理内存页时，需要让对应此物理内存页的管理数据结构Page做相关的清除处理，使得此物理内存页成为空闲；另外还需把表示虚地址与物理地址对应关系的二级页表项清除。请仔细查看和理解page_remove_pte函数中的注释。为此，需要补全在 kern/mm/pmm.c中的page_remove_pte函数。
#### 我的答案
如果page存在的话则减小page reference并释放page和二级页表，最后设置tlb
```c
if (*ptep & PTE_P) {
    struct Page *page = pte2page(*ptep);
    if (page_ref_dec(page) == 0) {
        free_page(page);
    }
    *ptep = 0;
    tlb_invalidate(pgdir, la);
}
```
#### 数据结构Page的全局变量（其实是一个数组）的每一项与页表中的页目录项和页表项有无对应关系？如果有，其对应关系是啥？
有，PTE每一项高二十位对应对应物理页的物理地址，PDE每一项高二十位对应对应页表的物理地址

#### 如果希望虚拟地址与物理地址相等，则需要如何修改lab2，完成此事？ 鼓励通过编程来具体完成这个问题
可修改 pmm.h 中的两个宏 PADDR(kva)和KADDR(pa),去除kernbase的加和减
