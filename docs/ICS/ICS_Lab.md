# Lab

## 参考博客
* [NOSAE's Blog](https://www.cnblogs.com/nosae/p/17045249.html)

## PA0 开发环境配置

1. 本实验使用的 ISA 是 riscv32。

2. 参考[博客](https://www.cnblogs.com/DreamW1ngs/p/18430400)中关于clangd的配置，这样vscode就可以正确跳转到各个函数了。（亲测，不是很好用，有的时候没法跳转🤷‍♂️）

    `clangd`是一个开源的语言服务器，可以配合`CompileDB`项目生成`compile_commands.json`（所有符号表索引数据库），方便只参与编译的文件代码进行快速跳转。
    
    具体使用方式：

    1. 安装compiledb：(注意wsl没法通过`pip3 install compiledb`来安装，需要通过以下方式安装)

        ```bash
        # 安装pipx
        sudo apt update
        sudo apt install pipx

        # 确保pipx在PATH中
        pipx ensurepath

        # 重新加载bash配置
        source ~/.bashrc

        # 使用pipx安装compiledb
        pipx install compiledb
        ```
        
    2. 在NEMU_PATH下运行`compiledb make`，从而在NEMU_PATH下生成`compile_commands.json`文件。
    3. 在wsl中使用`sudo apt-get install clangd`安装clangd
    4. vscode安装clangd插件后，设置`--compile-commands-dir=nemu`。这样会找到你vscode工作目录下nemu目录里的`compile_commands.json`文件。

3. 实验代码 git 跟踪

    整套实验代码有一个很有趣很神奇的功能，就是在每次编译、运行的时候都会自动提交修改，并生成一条 commit。PA 中称之为“开发跟踪”，通过这个手段来记录实验流程，并检查是否存在作弊行为。

    这一功能的实现得益于整个框架都使用提供好的 Makefile，且编译运行都通过 make 来进行。其原理为：

    * 根目录下 Makefile：

        ```Makefile
        STUID = 231220000
        STUNAME = lingyu

        GITFLAGS = -q --author='tracer-ics2024 <tracer@njuics.org>' --no-verify --allow-empty

        define git_commit
         	-@git add $(NEMU_HOME)/.. -A --ignore-errors
         	-@while (test -e .git/index.lock); do sleep 0.1; done
         	-@(echo "> $(1)" && echo $(STUID) $(STUNAME) && uname -a && uptime) | git commit -F - $(GITFLAGS)
         	-@sync
        endef
        ```

        * 定义了一个函数 git_commit
            - add 所有内容（NEMU_HOME/.. 即为实验根目录）并确保完成
            - 将参数、学号姓名、系统信息作为 commit message 进行 commit，author 在 GITFLAGS 中指定

    * NEMU 的相关 Makefile：

        ```Makefile
        -include $(NEMU_HOME)/../Makefile

        compile_git: 
            $(call git_commit, "compile NEMU")
        $(BINARY): compile_git

        run-env: $(BINARY) $(DIFF_REF_SO)
        run: run-env
            $(call git_commit, "run NEMU")
            $(NEMU_EXEC)
        gdb: run-env
            $(call git_commit, "gdb NEMU")
            gdb -s $(BINARY) --args $(NEMU_EXEC)
        ```

        * 导入了前面的 Makefile
        * 编译时添加了依赖 compile_git，其中会调用 git_commit 函数进行 commit，msg 开头为参数 compile NEMU
        * 对于 run 和 gdb，在编译后、运行前还会多加一条 run NEMU 或 gdb NEMU 的空 commit msg 作为记录

    由于实验从头到尾都跑在 NEMU 上（native 测试不算），所以每次修改、运行都会经过这些部分，进行 commit 记录，达到自动跟踪的目的。

## PA1 最简单的计算机

1. 在`cmd_c()`函数中, 调用`cpu_exec()`的时候传入了参数-1, 你知道这是什么意思吗?

    因为此时用户的目的是继续执行程序，执行多少步是未知的，所以期望下程序会执行无限步直至CPU进入停止状态。为了无限步执行，所以这里需要传入一个参数的最大值。由于在NEMU中，参数是uint32_t类型，所以其最大值是全1二进制串，对应为int32的-1，这不属于未定义行为。

2. 优美地退出: 在运行NEMU之后直接键入q退出, 终端输出了一些错误信息: 

    ```
    (nemu) q
    make: *** [/home/lingyu/ics2024/nemu/scripts/native.mk:38: run] Error 1
    ```

    Root Cause: 是由于`is_exit_status_bad`函数返回了-1，`main`函数直接返回了此函数返回的结果，make检测到该可执行文件返回了-1，因此报错。通过分析该函数得到解决方案：在输入q中途退出nemu后，将`nemu_state.state`设成`NEMU_QUIT`即可。

    Fix:

    ```C
    static int cmd_q(char *args) {
        nemu_state.state = NEMU_QUIT; // ✅Fix
        return -1;
    }
    ```

### 单步执行 si n
3. 实现单步执行 si n

    第一步，在`cmd_table`注册一条命令`si`。

    ```C
    static struct {
        const char *name;
        const char *description;
        int (*handler) (char *);
        } cmd_table [] = {
        { "help", "Display information about all supported commands", cmd_help },
        { "c", "Continue the execution of the program", cmd_c },
        { "q", "Exit NEMU", cmd_q },
        { "si", "Continue the execution in N steps, default 1", cmd_si }, // ✅
    };
    ```

    第二步，编写`cmd_si`函数，即`si`具体要执行的东西。

    ```C
    static int cmd_si(char *args) {
        char *arg = strtok(NULL, " ");
        int n;

        if (arg == NULL) {
            n = 1;
        } else {
            n = strtol(arg, NULL, 10);
        }
        cpu_exec(n);
        return 0;
    }
    ```

### 打印寄存器 info r
4. 实现打印寄存器 info r

    第一步，注册命令。

    ```C
    { "info", "Display the info of registers & watchpoints", cmd_info },
    ```

    第二步，编写`cmd_info`，其中调用的`isa_reg_display`函数就是PA文档里介绍的，简易调试器为了屏蔽ISA的差异，框架代码已经为大家准备了的API之一。

    ```C
    static int cmd_info(char *args) 
    {
        /* extract the first argument */
        char *arg = strtok(NULL, " ");
        if (arg == NULL) 
        {
            printf("Usage: info r (registers) or info w (watchpoints)\n");
        } 
        else 
        {
            if (strcmp(arg, "r") == 0) 
            {
                isa_reg_display();
            } 
            else if (strcmp(arg, "w") == 0) 
            {
                // todo
            } 
            else 
            {
                printf("Usage: info r (registers) or info w (watchpoints)\n");
            }
        }
        
        return 0;
    }
    ```

    第三步，实现`isa_reg_display`函数。

    ```C
    void isa_reg_display() 
    {
        int reg_num = ARRLEN(regs);
        int i;

        for (i = 0; i < reg_num; i++) 
        {
            printf("%s\t0x%08x\n", regs[i], cpu.gpr[i]);
        }
    }
    ```

### 扫描内存 x N EXPR
5. 实现扫描内存 x N EXPR

    第一步，注册命令。

    ```C
    { "x", "Usage: x N EXPR. Scan the memory from EXPR by N bytes", cmd_x },
    ```

    第二步，编写`cmd_x`函数，注意这里有两个参数N和EXPR，因此需要分别检查参数是否存在，并转换类型。然后就是利用`vaddr_read`每次读取4个字节并打印。打印格式参照了一下gdb的x命令，每行打印4个4字节。

    ```C
    static int cmd_x(char *args) 
    {
        char *arg1 = strtok(NULL, " ");
        if (arg1 == NULL) 
        {
            printf("Usage: x N EXPR\n");
            return 0;
        }
        char *arg2 = strtok(NULL, " ");
        if (arg2 == NULL) 
        {
            printf("Usage: x N EXPR\n");
            return 0;
        }

        int n = strtol(arg1, NULL, 10);
        vaddr_t expr = strtol(arg2, NULL, 16);

        int i, j;
        for (i = 0; i < n;) 
        {
            printf(ANSI_FMT("%#010x: ", ANSI_FG_CYAN), expr);
            
            for (j = 0; i < n && j < 4; i++, j++) 
            {
                word_t w = vaddr_read(expr, 4);
                expr += 4;
                printf("%#010x ", w);
            }
            puts(""); // 换行
        }
        
        return 0;
    }
    ```

    第三步，由于用到了`vaddr_read`，需要`#include <memory/vaddr.h>`。

### 表达式求值 p EXPR
6. 实现表达式求值 p EXPR

    第一步，注册命令。

    ```C
    {"p", "Usage: p EXPR. Calculate the expression, e.g. p $eax + 1", cmd_p },
    ```

    第二步，编写`cmd_p`函数，直接调用框架提供的`expr`函数即可。

    ```C
    static int cmd_p(char *args) 
    {
        if (args == NULL) 
        {
            printf("Usage: p EXPR\n");
            return 0;
        }

        bool success;
        word_t result = expr(args, &success);
        if (success) 
        {
            printf("%#010x\n", result);
        } 
        else 
        {
            printf("Invalid expression!\n");
        }

        return 0;
    }
    ```

2. 任务 PA1.2: 实现算术表达式求值
3. 任务 PA1.3: 实现所有要求, 提交完整的实验报告

## PA2 冯诺依曼计算机系统

## PA3 批处理系统

## PA4 分时多任务