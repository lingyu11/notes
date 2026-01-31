# OSTEP

## 资料
* [OSTEP](https://pages.cs.wisc.edu/~remzi/OSTEP/)

## Chapter 6 机制：受限直接执行

xv6 的上下文切换代码：

```asm
# void swtch(struct context **old, struct context *new); 
# 
# Save current register context in old 
# and then load register context from new. 
.globl swtch
swtch: 
  # Save old registers 
  movl 4(%esp), %eax  # put old ptr into eax 
  popl 0(%eax)        # save the old IP 
  movl %esp, 4(%eax)  # and stack 
  movl %ebx, 8(%eax)  # and other registers 
  movl %ecx, 12(%eax) 
  movl %edx, 16(%eax) 
  movl %esi, 20(%eax) 
  movl %edi, 24(%eax) 
  movl %ebp, 28(%eax) 

  # Load new registers 
  movl 4(%esp), %eax  # put new ptr into eax 
  movl 28(%eax), %ebp # restore other registers 
  movl 24(%eax), %edi 
  movl 20(%eax), %esi 
  movl 16(%eax), %edx 
  movl 12(%eax), %ecx 
  movl 8(%eax), %ebx 
  movl 4(%eax), %esp  # stack is switched here 
  pushl 0(%eax)       # return addr put in place 
  ret                 # finally return into new ctxt
```

x86 函数调用时，参数通过栈传递：调用 `swtch` 时，栈中依次存放 返回地址（调用者下一条指令地址）、`old`（参数 1，栈偏移 +4）、`new`（参数 2，栈偏移 +8）。`movl 4(%esp), %eax`中的`4(%esp)`指向`old`（`struct context **` 类型），将其加载到 `eax` 寄存器，后续通过 `eax` 操作旧进程的上下文结构。

保存完旧进程状态后，代码从 `new` 指向的上下文结构中读取寄存器值，加载到硬件寄存器，完成新进程的状态恢复。第二段的`movl 4(%esp), %eax`
中 `4(%esp)` 对应的是第二个参数 `new`（`struct context *` 类型）。由于之前步骤执行了 `popl`（弹出返回地址），此时栈中 `esp` 指向的位置已变化，`4(%esp)` 恰好对应 `new` 参数的地址，将其加载到 `eax`，后续通过 `eax` 操作新进程的上下文结构。

其中`movl 4(%eax), %esp`是**切换栈指针：核心跳转**，将 `new` 结构体的 `esp` 字段（`4(%eax)`）的值加载到硬件的 `esp` 寄存器。
这是上下文切换的 “灵魂步骤”：栈是进程的私有资源，切换 `esp` 意味着后续栈操作（如 `push`、`pop`）将基于新进程的栈，而非旧进程的栈。从此刻起，CPU 已 “感知” 到栈的切换，为后续恢复指令地址做好准备。

## Chapter 15 地址转换
!!! abstract "address translation"

    利用地址转换，操作系统可以控制进程的所有内存访问，确保访问在地址空间的界限内。这个技术高效的关键是<span style="color:blue;">**硬件支持**</span>，硬件快速地将所有内存访问操作中的虚拟地址（进程自己看到的内存位置）转换为物理地址（实际位置）。

    具体来说，每个 CPU 需要两个硬件寄存器：基址（base）寄存器和界限（bound）寄存器，有时称为限制（limit）寄存器。这组基址和界限寄存器，让我们能够将地址空间放在物理内存的任何位置，同时又能确保进程只能访问自己的地址空间。

## Chapter 16 分段
!!! abstract "segmentation"

    在 MMU 中引入不止一个基址和界限寄存器对，而是给地址空间内的每个逻辑段（segment）一对。一个段只是地址空间里的一个连续定长的区域，在典型的地址空间里有 3 个逻辑不同的段：代码、栈和堆。分段的机制使得操作系统能够将不同的段放到不同的物理内存区域，从而避免了虚拟地址空间中的未使用部分占用物理内存。

## Chapter 18 分页
!!! abstract "页表"

    现代操作系统的内存管理子系统中最重要的数据结构之一就是页表（page table）。通常，页表存储虚拟—物理地址转换（virtual-to-physical address translation），从而让系统知道地址空间的每个页实际驻留在物理内存中的哪个位置。由于每个地址空间都需要这种转换，因此一般来说，系统中每个进程都有一个页表。页表的确切结构要么由硬件（旧系统）确定，要么由 OS（现代系统）更灵活地管理。

## Chapter 19 分页：快速地址转换（TLB）
!!! abstract "TLB"

    地址转换旁路缓冲存储器（translation-lookaside buffer，TLB），它就是频繁发生的虚拟到物理地址转换的硬件缓存（cache）。因此，更好的名称应该是地址转换缓存（address-translation cache）。对每次内存访问，硬件先检查 TLB，看看其中是否有期望的转换映射，如果有，就完成转换（很快），不用访问页表（其中有全部的转换映射）。TLB 带来了巨大的性能提升。

## Chapter 27 插叙：线程 API

条件变量典型用法：

```C
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;  
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;

// 线程1的代码：
Pthread_mutex_lock(&lock);  
while (ready == 0) 
    Pthread_cond_wait(&cond, &lock);  
Pthread_mutex_unlock(&lock);

// 线程2的代码：
Pthread_mutex_lock(&lock);  
ready = 1;  
Pthread_cond_signal(&cond);  
Pthread_mutex_unlock(&lock);
```

`pthread_cond_wait`传锁、`pthread_cond_signal`不传锁的底层逻辑：

wait传锁 → 为了 “原子释放锁 + 休眠”（让其他线程能操作条件），以及 “唤醒后重获锁”（保证条件判断和数据处理的锁保护）；

signal不传锁 → 因为其仅负责 “唤醒等待线程”，锁的管理由等待线程和信号线程各自控制，职责边界清晰。

这一设计是多线程同步的经典范式：通过 “条件变量处理等待 / 唤醒，互斥锁保护条件和共享数据”，既避免了轮询的性能损耗，又保证了多线程操作的安全性。