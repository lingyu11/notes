## gcc
* `-c`: 只编译不链接。 它告诉编译器编译源代码文件，但不进行链接操作。当使用这个选项时，GCC 会将每个指定的源代码文件编译成一个对应的目标文件（通常是 .o 文件），而不是生成可执行文件。目标文件包含了编译后的机器代码，这些代码可以稍后被链接器（通常是 ld）用来生成可执行文件或库文件。
* `-I.`: 指定当前目录（`.`）为头文件搜索路径，这意味着编译器会在当前目录下查找头文件。
* `-L`: 增加 link search path，例如 `-L.`表示增加当前目录（`.`）为链接搜索路径
* `-l`: 代表链接某个库，链接时会自动加上 lib 的前缀。例如 `-lco-64` 表示会依次在库函数的搜索路径中查找 libco-64.so 和 libco-64.a，直到找到为止。
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
* `-fsanitize=undefined`: 用于启用 UndefinedBehaviorSanitizer（UBSan）。UBSan 是一个未定义行为检测工具，它可以帮助开发者发现和定位未定义行为，如使用未初始化的变量、除以零等。
* `-fPIE`: 生成位置无关的可执行文件。全称是 "Position Independent Executable"，即位置无关可执行文件。这意味着代码中的所有地址引用都是相对于程序的加载地址的偏移量，而不是绝对地址。这样，无论程序加载到内存的哪个位置，它都可以正确执行。
* `-fPIC`: 生成位置无关的代码。全称是 "Position Independent Code"，这种代码可以被加载到内存的任何位置并正确执行。它通常用于生成共享库（`.so` 文件），因为共享库需要在不同的进程中被加载到不同的内存地址。
* `-fno-pic`: 不生成位置无关的代码。这意味着如果代码被加载到不同的内存地址，它可能无法正常工作。因此，这个选项通常只在特定情况下使用，例如当你需要优化代码的性能或者当你知道代码将总是被加载到相同的内存地址时。
* `-ffreestanding -nostdlib`: 编译的目标代码是独立的，不依赖于任何操作系统的特定功能或库。不使用标准库。
* `-shared`: 生成共享库。这告诉编译器生成一个共享库（通常是 `.so` 文件shared object），而不是生成可执行文件。
* `-U_FORTIFY_SOURCE`: 用于取消 `_FORTIFY_SOURCE` 宏的定义，这意味着编译器不会对可能存在缓冲区溢出风险的函数调用进行额外的检查和保护。是用来防止 __longjmp_chk 代码检查到堆栈切换以后报错 (当成是 stack smashing)。
* `-x c`: 指定了输入文件的语言类型为C语言。
* `-w`: 关闭所有警告信息。
* `-m32`: 生成32位代码。


## objdump
* `-d`: 反汇编代码段。反汇编指定的目标文件或可执行文件中的代码段，不包括其他段（如数据段、BSS 段等）。
* `-D`: 反汇编所有段。包括代码段和数据段。
* `-h`: 显示 ELF 文件的头信息。
* `-S`: 在反汇编后的内容中添加源代码。
* `-t`: 显示符号表。包含所有函数、全局变量的名称和存储地址。

## readelf
* ELF (Executable and Linkable Format)是一种用于可执行文件、目标文件和共享库的标准文件格式，广泛应用于类 Unix 操作系统中。
* `-a`: all，显示ELF文件的所有节（section）和段（segment）的信息，以及符号表、字符串表、重定位信息等。
* `-l`: 显示 ELF 文件的程序头（Program Header）信息。程序头包含了关于 ELF 文件如何被加载到内存中的信息，例如段（Segment）的大小、类型、偏移量等。
* `-S`: 显示 ELF 文件中的所有 Section Header 信息。

## gdb
* [a small tutorial for GDB](https://linuxconfig.org/gdb-debugging-tutorial-for-beginners)
* `starti`: 开始执行程序，并在执行第一条指令之前停止。这个命令通常用于在程序开始时设置断点，以便检查程序的初始状态。
* `layout asm`: 切换到汇编代码布局模式
* `layout src`: 切换到源代码布局模式
* `l`: 列出当前行附近的源代码。
* `b main`: 设置一个断点，当程序执行到 main 函数时停止。
* `b *0x7c00`: 设置一个断点，当程序执行到地址 0x7c00 时停止。
* `si`: 单步执行一条汇编指令。
* `enter键`：重复上一个命令。
* `p $rsp`: 打印寄存器 `rsp` 的当前值，也就是显示栈指针的值（一个地址，表示栈顶的内存地址）。例子输出`$1 = (void *) 0x7fffffffdcb0`
* `x/x $rsp`: 以十六进制格式检查内存地址 `$rsp` 处的值。x 命令是GDB中的一个检查内存的命令，/x 表示以十六进制格式显示，$rsp 是寄存器 `rsp` 的当前值，它指向栈顶。例子输出`0x7fffffffdcb0: 0x00000001`，这个`M[rsp]`是1显然非法，进而导致segmentation fault。
* `x/i $eip`: `/i`这是 x 命令的一个参数，表示以指令的形式显示内存内容。输出`0x0000000000007c00:  jmp    0x7c00`
* `x/6i 0x3ffffff000`: 以十六进制格式显示地址 0x3ffffff000 处开始的 6 条指令。
* `x/16b 0x7c00`: 以字节为单位，从内存地址 0x7c00 开始，显示 16 个字节的内容。
* `x/2xw 0xffffd2bc`: 从0xffffd2bc地址开始的2个字（一个字是4个字节）的存储单元内容，并用十六进制表示。`/t`则是以二进制显示。
* `tui disable`: 禁用 TUI（Text User Interface，文本用户界面）模式，使 GDB 恢复到默认的命令行界面。
* `info registers` or `i r`: 显示当前所有寄存器的值。
* `info registers eflag` or `i r eflag`: 显示当前 eflag 寄存器的值。
* `start`: 开始调试。
* `s`: 单步执行。
* `r`: 重新执行程序，直到程序结束或遇到断点。
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
* `rwatch *0xfde0`: 在内存地址 0xfde0 处设置一个 read watch point，看下谁读了它。
* `-x init.gdb`: 执行 init.gdb 文件中的命令。
* `inferior 2`: 切换到子进程 2。
* `start minimal`: 以minimal这个可执行文件作为参数运行程序。（例如在debug loader程序时，想看loader是如何将minimal加载到内存的，可以在gdb loader后输入这个）
* `info locals`: 显示当前函数的局部变量的值。
* `info breakpoints`: 显示当前设置的所有断点信息。
* `bt`: backtrace，显示当前的调用栈。
* `where`: 显示当前执行位置的调用栈信息。和`bt`功能一样。
* `frame 1`: 切换到调用栈中的第 1 个帧，`info frame`: 显示当前帧的信息。
* `b sum_to if i==5`: 设置一个条件断点，当 i 的值等于 5 时触发断点。

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
* `-T`: 显示每个系统调用的执行时间。

## grep
* `grep -nr "\bmain\b" nemu/src`: 搜索 nemu/src 目录下所有文件中包含 "main" 字符串的行，并显示行号和文件名。`-r` 表示递归搜索子目录。`-n` 表示显示行号。`\b` 表示单词边界，确保 "main" 是一个完整的单词。
* `grep -v`: `-v`表示反向匹配，即输出不包含匹配模式的行。
* `grep -e`: `-e`指定要搜索的pattern。
* `grep -E`: 开启正则表达式。

## vim
* `vimtutor`: 启动 Vim 自带的交互式教程。
* `%`: 在 Vim 中，`%`符号表示当前文件名。所以，`%!grep -v -e open` 实际上是在对当前文件执行 grep 命令。
* `!`: 用于在 Vim 中执行外部命令。它告诉 Vim 接下来的部分是一个外部命令，而不是 Vim 内部命令。
* `%!grep -v -e open`: 在当前文件中搜索不包含 "open" 字符串的行，并输出结果。
* `i1<ESC>q1yyp<C-a>q98@1`: 一次性生成 1 ~ 100 之间的数字：

    ```bash
    1
    2
    3
    .....
    98
    99
    100
    ```

    - `i1<ESC>`: 进入插入模式，输入数字1，然后按Esc键退出插入模式。
    - `q1yyp`: 录制一个名为1的宏，内容是复制当前行并粘贴一次。
    - `<C-a>`: 执行Ctrl+a，Ctrl+a是一个特殊的按键组合，用于执行数字递增操作。
    - `q`: 退出宏录制模式。
    - `98@1`: 调用名为1的宏98次。

## make
* `-nB`: 它的作用是执行 Makefile 中的规则，但不实际执行任何命令，而是打印出如果执行这些命令会发生什么。`-n` 选项表示“干运行”，即只显示命令而不执行它们，而 `-B` 选项则表示“强制重新生成所有目标”，即使它们已经是最新的。
* `.PHONY: run clean`: 这是 Makefile 中的规则，用于定义伪目标 `run` 和 `clean`。伪目标不是实际的文件，而是一个标签，用于指定一个动作。在 Makefile 中，伪目标通常用于定义一些不依赖于文件的操作，比如编译、运行和清理。
* `$@`: 是一个自动变量，表示目标文件的名称。
* `$<`: 这个自动变量表示第一个依赖文件的名称。
* `$^`: 这个自动变量表示所有依赖文件的名称，以空格分隔。
* `NAME := hello`: 定义了一个变量 NAME，其值为 hello
* example 1: 记录编译过程中的详细信息，并将这些信息保存到一个名为 compile.log 的文件中。
    - `grep` 命令用于过滤输出，只保留不包含以 `#`、`echo`、`mkdir` 或 `make` 开头的行。
    - `sed` 命令用于文本替换。这里的替换规则是将环境变量 `AM_HOME` 替换为空字符串 `\AM`。`g` 表示全局替换，即替换所有匹配的字符串。接着继续将当前工作目录 `$(PWD)` 替换为 `.`

    ```bash
    make -nB \
          | grep -ve '^\(\#\|echo\|mkdir\|make\)' \
          | sed "s#$(AM_HOME)#\AM#g" \
          | sed "s#$(PWD)#.#g" \
          > compile.log
    ```

* example 2: 使用 grep 过滤掉以 `#`、`echo` 或 `mkdir` 开头的行。

    ```bash
    make -nB | grep -ve '^\(\#\|echo\|mkdir\)' | vim -
    :set nowrap  # vim 不换行显示
    :g/fixdep/d  # vim 删除所有含fixdep的行
    :g/mv/d      # vim 删除所有含mv的行
    :execute "normal \<c-v>" # wsl里CTRL+v是粘贴而不是进入visual block mode，只能先用这个workaround了
    :'<,'>s/ /\r/g # 将所有空格替换为回车符。其中'<,'>是用visual mode选中一行就自动出来了
    ```

* 想要看到 make 过程中的详细信息，需要在 makeFile 里的 CFLAGS 里加上`-Wl,--verbose`
* `make gdb`: 启动 GDB 调试器，并将程序加载到 GDB 中。


## qemu-system-x86_64
* `qemu-system-x86_64 -monitor stdio minimal.img`: 启动 QEMU 模拟器并加载一个名为 minimal.img 的磁盘镜像文件。这个指令的作用是创建一个虚拟的 x86_64 架构的计算机环境，并且通过标准输入输出（stdio）来与 QEMU 监视器进行交互(`-monitor stdio`)。
* `-s`: 表示启用GDB调试服务器，允许远程GDB调试器连接到QEMU实例。
* `-S`: 表示在启动时冻结CPU，等待远程GDB调试器连接后再继续执行。

## vscode
* ctl-p，然后输入#offsetof，选择中间有两杠的框框，即可找到offsetof函数的定义。
* 配置 vscode
    * 配置解析选项: c_cpp_properties.json
        - 解锁正确的代码解析
    * 配置构建选项: tasks.json
        - 解锁make (可跟命令行参数)
    * 配置运行选项: launch.json
        - 解锁单步调试

## vim 配置
1. [vim 教程: vim-galore](https://github.com/mhinz/vim-galore)
2. 配置vim如下：

    ```bash
    linux$ cp /etc/vim/vimrc ~/.vimrc
    linux$ vim ~/.vimrc
    ```

    在 vimrc 末尾中加入以下内容：

    ```bash
    syntax on
    set background=dark
    filetype plugin indent on
    set showmatch          " Show matching brackets.
    set ignorecase         " Do case insensitive matching
    set smartcase          " Do smart case matching
    set incsearch          " Incremental search
    set hidden             " Hide buffers when they are abandoned
    setlocal noswapfile " 不要生成swap文件
    set bufhidden=hide " 当buffer被丢弃的时候隐藏它
    "colorscheme evening " 设定配色方案
    set number " 显示行号
    set cursorline " 突出显示当前行
    set ruler " 打开状态栏标尺
    set shiftwidth=2 " 设定 << 和 >> 命令移动时的宽度为 2
    set softtabstop=2 " 使得按退格键时可以一次删掉 2 个空格
    set tabstop=2 " 设定 tab 长度为 2
    set nobackup " 覆盖文件时不备份
    set autochdir " 自动切换当前目录为当前文件所在的目录
    set backupcopy=yes " 设置备份时的行为为覆盖
    set hlsearch " 搜索时高亮显示被找到的文本
    set noerrorbells " 关闭错误信息响铃
    set novisualbell " 关闭使用可视响铃代替呼叫
    set t_vb= " 置空错误铃声的终端代码
    set matchtime=2 " 短暂跳转到匹配括号的时间
    set magic " 设置魔术
    set smartindent " 开启新行时使用智能自动缩进
    set backspace=indent,eol,start " 不设定在插入状态无法用退格键和 Delete 键删除回车符
    set cmdheight=1 " 设定命令行的行数为 1
    set laststatus=2 " 显示状态栏 (默认值为 1, 无法显示状态栏)
    set statusline=\ %<%F[%1*%M%*%n%R%H]%=\ %y\ %0(%{&fileformat}\ %{&encoding}\ Ln\ %l,\ Col\ %c/%L%) " 设置在状态行显示的信息
    set foldenable " 开始折叠
    set foldmethod=syntax " 设置语法折叠
    set foldcolumn=0 " 设置折叠区域的宽度
    setlocal foldlevel=1 " 设置折叠层数为 1
    nnoremap <space> @=((foldclosed(line('.')) < 0) ? 'zc' : 'zo')<CR> " 用空格键来开关折叠
    ```

## some example
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