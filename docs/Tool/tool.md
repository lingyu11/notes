## gcc
* -c: 只编译不链接。 它告诉编译器编译源代码文件，但不进行链接操作。当使用这个选项时，GCC 会将每个指定的源代码文件编译成一个对应的目标文件（通常是 .o 文件），而不是生成可执行文件。目标文件包含了编译后的机器代码，这些代码可以稍后被链接器（通常是 ld）用来生成可执行文件或库文件。

## objdump
* -d: 反汇编。 它告诉 objdump 显示每个目标文件或可执行文件中的代码和数据的反汇编。

## example
1. hello.c
```C
#include <stdio.h>
int main()
{
    printf("Hello World\n");
}
```
```
~/ICS/practise$ gcc a.c                    # this generate a.out
~/ICS/practise$ ./a.out 
Hello World
~/ICS/practise$ hexdump a.out | less
~/ICS/practise$ gcc -c a.c                 # this generate a.o
~/ICS/practise$ objdump -d a.o             # disassemble

a.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <main>:
   0:   f3 0f 1e fa             endbr64 
   4:   55                      push   %rbp
   5:   48 89 e5                mov    %rsp,%rbp
   8:   48 8d 05 00 00 00 00    lea    0x0(%rip),%rax        # f <main+0xf>
   f:   48 89 c7                mov    %rax,%rdi
  12:   e8 00 00 00 00          call   17 <main+0x17>
  17:   b8 00 00 00 00          mov    $0x0,%eax
  1c:   5d                      pop    %rbp
  1d:   c3                      ret    
```