## lab3
#### 2014011319 王倩
### 练习0：填写已有实验
本实验依赖实验1/2。请把你做的实验1/2的代码填入本实验中代码中有“LAB1”,“LAB2”的注释相应部分。

### 练习1：给未被映射的地址映射上物理页（需要编程）
完成do_pgfault（mm/vmm.c）函数，给未被映射的地址映射上物理页。设置访问权限 的时候需要参考页面所在 VMA 的权限，同时需要注意映射物理页时需要操作内存控制 结构所指定的页表，而不是内核的页表。注意：在LAB2 EXERCISE 1处填写代码。执行make　qemu后，如果通过check_pgfault函数的测试后，会有“check_pgfault() succeeded!”的输出，表示练习1基本正确。
##### 设计过程
当物理内存不存在时可能是访问权限不够，或者需要换页，或者映射关系已经被撤销（页表项为0）。因此需要做的是：
首先，寻找pte对应的pt，如果失败则进行处理。
如果该物理地址不存在，则给其分配物理空间，否则，把该页载入到内存，并进行map，如果失败则进行处理。

##### 请描述页目录项（Pag Director Entry）和页表（Page Table Entry）中组成部分对ucore实现页替换算法的潜在用处。
pgdir存储了所有物理地址与逻辑地址对应的关系，是页替换的范围，而pte则存储了被换的页的信息。
##### 如果ucore的缺页服务例程在执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？
出现异常后，CPU把异常地址存入CR2，给出错误代码。OS会把它保存在struct trapframe 的tf_err中。中断服务例程会调用do_pgfault进行具体处理。

### 练习2：补充完成基于FIFO的页面替换算法（需要编程）
完成vmm.c中的do_pgfault函数，并且在实现FIFO算法的swap_fifo.c中完成map_swappable和swap_out_vistim函数。通过对swap的测试。注意：在LAB2 EXERCISE 2处填写代码。执行make　qemu后，如果通过check_swap函数的测试后，会有“check_swap() succeeded!”的输出，表示练习2基本正确。
##### 设计过程
首先在_fifo_map_swappable函数中将最近被用的页加入队列。
然后，需要完成页面换出函数：  
1.找到最早加入的页表  
2.将换出的页表删除  
3.更新ptr_page  
##### 如果要在ucore上实现"extended clock页替换算法"请给你的设计方案，现有的swap_manager框架是否足以支持在ucore中实现此算法？如果是，请给你的设计方案。如果不是，请给出你的新的扩展和基此扩展的设计方案。
现在足以支持extended clock。PTE_A可以表明某页是否被访问过，所以只需要修改_fifo_swap_out_victim函数，增加如果未被访问，则删除并返回即可。
##### 需要被换出的页的特征是什么？
在FIFO中是最早加入的一页
##### 在ucore中如何判断具有这样特征的页？
mm->sm_priv->next指的页
##### 何时进行换入和换出操作？
当需要的页不在页表且页表满了的时候 
