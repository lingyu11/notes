## gcc
* `-c`: 只编译不链接。 它告诉编译器编译源代码文件，但不进行链接操作。当使用这个选项时，GCC 会将每个指定的源代码文件编译成一个对应的目标文件（通常是 .o 文件），而不是生成可执行文件。目标文件包含了编译后的机器代码，这些代码可以稍后被链接器（通常是 ld）用来生成可执行文件或库文件。
* `-I.`: 指定当前目录（`.`）为头文件搜索路径，这意味着编译器会在当前目录下查找头文件。
* `-g`：生成调试信息。这告诉编译器在编译过程中生成调试信息，这些信息可以被调试器（如 GDB）使用，以便在调试程序时提供更多的帮助。只有这样gdb才能调试时看到代码。
* `--verbose`: 查看所有编译选项。这会使编译器在编译过程中输出编译过程中的每一个步骤。
* `-Wl,--verbose`: 查看所有链接选项。
* `-S`: 表示编译到汇编代码阶段，即生成 `.s` 文件，而不是二进制文件。`.S`文件：在编译之前，会经过C预处理器（cpp）的处理。这意味着`.S`文件可以包含C预处理指令，如宏定义、条件编译指令`#ifdef`、`#ifndef`、`#endif`等，以及`#include`指令来包含其他文件。预处理器会根据这些指令对文件进行处理，然后将处理后的文件交给编译器进行编译。`.s`文件：不会经过C预处理器的处理，直接被编译器编译。因此，`.s`文件中不能包含C预处理指令，只能包含汇编代码。
* `-Wall`: 启用所有的警告信息。
* `-Os`: 优化代码大小。在不显著影响程序性能的前提下，尽可能地减少代码的大小。
* `-s`: 去除符号表和调试信息，这也有助于减小最终可执行文件的大小
* `-DSTANDALONE`: `-D` 选项定义一个名为 `STANDALONE`的宏，这个宏可以在代码中被用来控制不同的编译路径或行为。
* `-fsanitize=address`: 用于启用 AddressSanitizer（ASan）。AddressSanitizer 是一个内存错误检测工具，它可以帮助开发者发现和定位内存访问错误，如缓冲区溢出、越界访问、使用未初始化的内存等。
* `-fsanitize=thread`: 用于启用 ThreadSanitizer（TSan）。ThreadSanitizer 是一个线程错误检测工具，它可以帮助开发者发现和定位线程相关的错误，如数据竞争、竞态条件等。（并非万能，有的时候可能抓不到data race）
* `-fPIE`: 生成位置无关的可执行文件。全称是 "Position Independent Executable"，即位置无关可执行文件。这意味着代码中的所有地址引用都是相对于程序的加载地址的偏移量，而不是绝对地址。这样，无论程序加载到内存的哪个位置，它都可以正确执行。
* `-ffreestanding -nostdlib`: 编译的目标代码是独立的，不依赖于任何操作系统的特定功能或库。不使用标准库。

## objdump
* `-d`: 反汇编。 它告诉 objdump 显示每个目标文件或可执行文件中的代码和数据的反汇编。

## readelf
* ELF (Executable and Linkable Format)是一种用于可执行文件、目标文件和共享库的标准文件格式，广泛应用于类 Unix 操作系统中。
* `-a`: all，显示ELF文件的所有节（section）和段（segment）的信息，以及符号表、字符串表、重定位信息等。
* `-l`: 显示 ELF 文件的程序头（Program Header）信息。程序头包含了关于 ELF 文件如何被加载到内存中的信息，例如段（Segment）的大小、类型、偏移量等。

## gdb
* `starti`: 开始执行程序，并在执行第一条指令之前停止。这个命令通常用于在程序开始时设置断点，以便检查程序的初始状态。
* `layout asm`: 切换到汇编代码布局模式
* `layout src`: 切换到源代码布局模式
* `si`: 单步执行一条汇编指令。
* `enter键`：重复上一个命令。
* `p $rsp`: 打印寄存器 `rsp` 的当前值，也就是显示栈指针的值（一个地址，表示栈顶的内存地址）。例子输出`$1 = (void *) 0x7fffffffdcb0`
* `x/x $rsp`: 以十六进制格式检查内存地址 `$rsp` 处的值。x 命令是GDB中的一个检查内存的命令，/x 表示以十六进制格式显示，$rsp 是寄存器 `rsp` 的当前值，它指向栈顶。例子输出`0x7fffffffdcb0: 0x00000001`，这个`M[rsp]`是1显然非法，进而导致segmentation fault。
* `x/i $eip`: `/i`这是 x 命令的一个参数，表示以指令的形式显示内存内容。输出`0x0000000000007c00:  jmp    0x7c00`
* `x/16b 0x7c00`: 以字节为单位，从内存地址 0x7c00 开始，显示 16 个字节的内容。
* `tui disable`: 禁用 TUI（Text User Interface，文本用户界面）模式，使 GDB 恢复到默认的命令行界面。
* `info registers`: 显示当前所有寄存器的值。
* `info registers eflag`: 显示当前 eflag 寄存器的值。
* `start`: 开始调试。
* `s`: 单步执行。
* `-x`: 指定一个包含GDB命令的脚本文件，以便在启动GDB时自动执行这些命令。例如`gdb -x init.gdb` or `gdb -x debug.py`
* `c`: 继续执行程序，直到程序结束或遇到断点。
* `n`: 下一步。
* `rs`: 反向执行。需要.gdb文件里写了`target record-full`才能支持反向执行。
* `info proc mappings`: 显示当前进程的内存映射。
* `info inferiors`: 显示当前进程的子进程信息（pid等）。
* `!pmap pid`: 以pmap命令显示进程的内存映射。
* `p/s argv`: 打印 argv 变量的值。`/s` 是 print string 的缩写，用于以字符串的形式打印变量的值。
* `finish`: 在当前函数中执行到函数结束。
* `wa environ`: 设置watch point，当environ变量的值改变时，触发断点。设置`wa environ`好后再执行`c`即可。
* `-x init.gdb`: 执行 init.gdb 文件中的命令。
* `inferior 2`: 切换到子进程 2。
* `start minimal`: 以minimal这个可执行文件作为参数运行程序。（例如在debug loader程序时，想看loader是如何将minimal加载到内存的，可以在gdb loader后输入这个）

### init.gdb示例
```bash
set follow-fork-mode child # 设置GDB在遇到fork系统调用时，跟踪子进程。
set detach-on-fork off     # 设置GDB在fork后不分离 (detach) 子进程，即保持对父进程和子进程的跟踪。
set follow-exec-mode same  # 设置GDB在遇到exec系统调用时，继续跟踪当前进程。
set confirm off            # 关闭GDB的确认提示。
set pagination off         # 关闭GDB的分页输出。
tui disable                # 禁用GDB的文本用户界面 (TUI)。

skip function strlen       # 跳过对strlen函数的单步执行。
skip function strcpy
skip function strchr
skip function print
skip function zalloc
skip function peek
skip function gettoken

source visualize.py        # 加载一个名为visualize.py的脚本，用于可视化调试数据。

break runcmd               # 在runcmd函数处设置断点。

run                        # 运行程序。
n                          # 单步执行下一条指令。
define hook-stop           # 定义一个名为hook-stop的GDB钩子 (hook)，在每次停止时执行。
    pdump                  # 在hook-stop钩子中执行的命令，用于打印某些调试信息。
end
```

### visualize.py
```python
import gdb
import subprocess
import re

class ProcDump(gdb.Command):
    def __init__(self):
        super(ProcDump, self).__init__(
            "pdump", gdb.COMMAND_DATA, gdb.COMPLETE_SYMBOL
        )

    def invoke(self, *_):
        print()

        for proc in gdb.inferiors():
            pid = proc.pid
            if int(pid) == 0:
                continue

            print(f'Process {proc.num} ({pid})', end='')
            if proc is gdb.selected_inferior():
                print('*')
            else:
                print()

            for fd_desc in subprocess.check_output(
                ['ls', '-l', f'/proc/{pid}/fd'], encoding='utf-8'
            ).splitlines()[1:]:
                perm, *_, fd, _, fname = fd_desc.split()

                if int(fd) < 10:
                    if 'rw' in perm: rw = '<->'
                    elif 'r' in perm: rw = '<--'
                    elif 'w' in perm: rw = '-->'
                    if 'pipe:' in fname:
                        pipe_id = re.search(f'[0-9]+', fname).group()
                        print(f'    {fd} {rw} [=== {pipe_id} ===]')
                    else:
                        print(f'    {fd} {rw} {fname}')

            print()

ProcDump()
```

## strace
* strace: system call trace
* `-f`: 跟踪所有子进程，即如果程序创建了子进程，strace 也会跟踪这些子进程的系统调用。

## vim
* `%`：在 Vim 中，`%`符号表示当前文件名。所以，`%!grep -v -e open` 实际上是在对当前文件执行 grep 命令。
* `!`: 用于在 Vim 中执行外部命令。它告诉 Vim 接下来的部分是一个外部命令，而不是 Vim 内部命令。
* `grep -v`: `-v`表示反向匹配，即输出不包含匹配模式的行。
* `grep -e`: `-e`指定要搜索的pattern。
* `%!grep -v -e open`: 在当前文件中搜索不包含 "open" 字符串的行，并输出结果。

## make
* `-nB`: 它的作用是执行 Makefile 中的规则，但不实际执行任何命令，而是打印出如果执行这些命令会发生什么。`-n` 选项表示“干运行”，即只显示命令而不执行它们，而 `-B` 选项则表示“强制重新生成所有目标”，即使它们已经是最新的。
* `.PHONY: run clean`: 这是 Makefile 中的规则，用于定义伪目标 `run` 和 `clean`。伪目标不是实际的文件，而是一个标签，用于指定一个动作。在 Makefile 中，伪目标通常用于定义一些不依赖于文件的操作，比如编译、运行和清理。
* `$@`: 是一个自动变量，表示目标文件的名称。
* `$<`: 这个自动变量表示第一个依赖文件的名称。
* `$^`: 这个自动变量表示所有依赖文件的名称，以空格分隔。
* `NAME := hello`: 定义了一个变量 NAME，其值为 hello
* example: 记录编译过程中的详细信息，并将这些信息保存到一个名为 compile.log 的文件中。
    - `grep` 命令用于过滤输出，只保留不包含以 `#`、`echo`、`mkdir` 或 `make` 开头的行。
    - `sed` 命令用于文本替换。这里的替换规则是将环境变量 `AM_HOME` 替换为空字符串 `\AM`。`g` 表示全局替换，即替换所有匹配的字符串。接着继续将当前工作目录 `$(PWD)` 替换为 `.`

    ```bash
    make -nB \
          | grep -ve '^\(\#\|echo\|mkdir\|make\)' \
          | sed "s#$(AM_HOME)#\AM#g" \
          | sed "s#$(PWD)#.#g" \
          > compile.log
    ```
* 想要看到make过程中的详细信息，需要在makeFile里的CFLAGS里加上`-Wl,--verbose`


## qemu-system-x86_64
* `qemu-system-x86_64 -monitor stdio minimal.img`: 启动 QEMU 模拟器并加载一个名为 minimal.img 的磁盘镜像文件。这个指令的作用是创建一个虚拟的 x86_64 架构的计算机环境，并且通过标准输入输出（stdio）来与 QEMU 监视器进行交互(`-monitor stdio`)。
* `-s`: 表示启用GDB调试服务器，允许远程GDB调试器连接到QEMU实例。
* `-S`: 表示在启动时冻结CPU，等待远程GDB调试器连接后再继续执行。

## vscode
* ctl-p，然后输入#offsetof，选择中间有两杠的框框，即可找到offsetof函数的定义。

## example
1. hello.c
```C
#include <stdio.h>
int main()
{
    printf("Hello World\n");
}
```
```bash
linux$ gcc a.c                    # this generate a.out
linux$ ./a.out 
Hello World
linux$ hexdump a.out | less
linux$ gcc -c a.c                 # this generate a.o
linux$ objdump -d a.o             # disassemble

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

2. 
```bash
gcc -g -S minimal.S > minimal.s  # 使用 GCC 编译器将汇编代码文件 minimal.S 编译成汇编代码文件 minimal.s，并且生成调试信息。
as minimal.s -o minimal.o        # 使用 as 汇编器将汇编代码文件 minimal.s 编译成目标文件 minimal.o
ld -o minimal minimal.o          # 使用 ld 链接器将目标文件 minimal.o 链接成可执行文件 minimal
```