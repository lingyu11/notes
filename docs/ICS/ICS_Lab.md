# Lab

## 参考资料
* [实验网站](https://nju-projectn.github.io/ics-pa-gitbook/ics2024/)
* [NOSAE's Blog](https://www.cnblogs.com/nosae/category/2263575.html)

## PA0 开发环境配置

1. 本实验使用的 ISA 是 riscv32。

2. 参考[博客](https://www.cnblogs.com/DreamW1ngs/p/18430400)中关于clangd的配置，这样vscode就可以正确跳转到各个函数了。（亲测，不是很好用，不建议，有的时候没法跳转🤷‍♂️）

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

    ```bash
    ~/ics2024/nemu$ make run
    (nemu) q
    make: *** [/home/lingyu/ics2024/nemu/scripts/native.mk:38: run] Error 1
    ```

    然而，若是直接运行编译出来的可执行文件的话是不会报错的：

    ```bash
    ~/ics2024/nemu$ ./build/riscv32-nemu-interpreter
    (nemu) q
    ```

    Root Cause: 是由于`is_exit_status_bad`函数返回了1，`main`函数直接返回了此函数返回的结果，make检测到该可执行文件返回了1，因此报错。[make手册](https://www.gnu.org/software/make/manual/html_node/Errors.html)：make执行shell命令时，如果返回值是1，则退出当前rule执行，显示报错。
    
    通过分析该函数得到解决方案：在输入q中途退出nemu后，将`nemu_state.state`设成`NEMU_QUIT`即可。

    Fix:

    ```C
    static int cmd_q(char *args) 
    {
        nemu_state.state = NEMU_QUIT; // ✅Fix
        return -1;
    }

    int is_exit_status_bad() 
    {
        int good = (nemu_state.state == NEMU_END && nemu_state.halt_ret == 0) ||
            (nemu_state.state == NEMU_QUIT);
        return !good;
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
    static int cmd_si(char *args) 
    {
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

    输出：

    ```bash
    (nemu) info r
    $0      0x00000000
    ra      0x00000000
    sp      0x00000000
    gp      0x00000000
    tp      0x00000000
    t0      0x80000000
    t1      0x00000000
    t2      0x00000000
    s0      0x00000000
    s1      0x00000000
    a0      0x00000000
    a1      0x00000000
    a2      0x00000000
    a3      0x00000000
    a4      0x00000000
    a5      0x00000000
    a6      0x00000000
    a7      0x00000000
    s2      0x00000000
    s3      0x00000000
    s4      0x00000000
    s5      0x00000000
    s6      0x00000000
    s7      0x00000000
    s8      0x00000000
    s9      0x00000000
    s10     0x00000000
    s11     0x00000000
    t3      0x00000000
    t4      0x00000000
    t5      0x00000000
    t6      0x00000000
    ```

### 扫描内存 x N EXPR
5. 实现扫描内存 x N EXPR: 求出表达式EXPR的值, 将结果作为起始内存地址, 以十六进制形式输出连续的 N 个 4 字节。

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

    输出：

    ```bash
    (nemu) x 10 0x80000000
    0x80000000: 0x00000297 0x00028823 0x0102c503 0x00100073 
    0x80000010: 0xdeadbe00 0xdcdcdcdc 0xdcdcdcdc 0xdcdcdcdc 
    0x80000020: 0xdcdcdcdc 0xdcdcdcdc
    ```

    这和读入的客户程序镜像文件对应上了（客户程序读入`RESET_VECTOR` 0x80000000处）：

    ```C
    static const uint32_t img [] = 
    {
        0x00000297,  // auipc t0,0
        0x00028823,  // sb  zero,16(t0)
        0x0102c503,  // lbu a0,16(t0)
        0x00100073,  // ebreak (used as nemu_trap)
        0xdeadbeef,  // some data
    };
    ```

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

    第三步，完善 `expr` 函数：首先调用`make_token`提取符号，再调用`eval`进行计算。

    ```C
    word_t expr(char *e, bool *success) 
    {
        if (!make_token(e)) 
        {
            *success = false;
            return 0;
        }

        return eval(0, nr_token - 1, success); // ✅
    }
    ```

    第四步，实现`make_token`的功能：每次都用所有的正则来匹配当前位置的字符，如果有匹配成功的就加入这个token（空字符除外），如果都匹配不成功就打印错误信息并返回false给上层函数。

    ```C
    static bool make_token(char *e) 
    {
        int position = 0;
        int i;
        regmatch_t pmatch;

        nr_token = 0;

        while (e[position] != '\0') 
        {
            /* Try all rules one by one. */
            for (i = 0; i < NR_REGEX; i ++) 
            {
                if (regexec(&re[i], e + position, 1, &pmatch, 0) == 0 && pmatch.rm_so == 0) 
                {
                    char *substr_start = e + position;
                    int substr_len = pmatch.rm_eo;

                    Log("match rules[%d] = \"%s\" at position %d with len %d: %.*s",
                        i, rules[i].regex, position, substr_len, substr_len, substr_start);
                    
                    position += substr_len;
                    
                    // ================= solution =================
                    if (rules[i].token_type == TK_NOTYPE) break;

                    tokens[nr_token].type = rules[i].token_type;
                    switch (rules[i].token_type) 
                    {
                        case TK_NUM:
                        case TK_REG:
                        case TK_VAR:
                            strncpy(tokens[nr_token].str, substr_start, substr_len);
                            tokens[nr_token].str[substr_len] = '\0';
                            // todo: handle overflow (token exceeding size of 32B)
                    }
                    nr_token++;
                    // ===========================================

                    break;
                }
            }

            if (i == NR_REGEX) 
            {
                printf("no match at position %d\n%s\n%*.s^\n", position, e, position, "");
                return false;
            }
        }

        return true;
    }
    ```

    make_token用到的token类型以及对应的正则表达式如下：

    ```C
    enum 
    {
        TK_NOTYPE = 256, TK_EQ,
        TK_NUM, // 10 & 16
        TK_REG,
        TK_VAR,
    };

    static struct rule 
    {
        const char *regex;
        int token_type;
    } rules[] = 
    {
        {" +", TK_NOTYPE},    // spaces
        {"\\+", '+'},         // plus
        {"-", '-'},
        {"\\*", '*'},
        {"/", '/'},
        {"==", TK_EQ},        // equal
        {"\\(", '('},
        {"\\)", ')'},

        {"[0-9]+", TK_NUM},   // TODO: non-capture notation (?:pattern) makes compilation failed
        {"\\$\\w+", TK_REG},
        {"[A-Za-z_]\\w*", TK_VAR},
    };
    ```

    第五步，完善`eval`函数:

    ```C
    bool check_parentheses(int p, int q) 
    {
        if (tokens[p].type == '(' && tokens[q].type == ')') 
        {
            int par = 0; // 括号计数器
            for (int i = p; i <= q; i++) 
            {
                if (tokens[i].type == '(') par++;
                else if (tokens[i].type == ')') par--;

                if (par == 0) return i == q; // 最外层括号完整匹配
            }
        }
        return false;
    }

    // 查找主操作符的位置
    // 主操作符是指整个表达式中优先级最低的操作符，它是表达式求值时最后执行的操作符
    int find_major(int p, int q) 
    {
        int ret = -1, par = 0, op_type = 0;
        for (int i = p; i <= q; i++) 
        {
            if (tokens[i].type == TK_NUM) 
            {
                continue;
            }
            if (tokens[i].type == '(') 
            {
                par++;
            } 
            else if (tokens[i].type == ')') 
            {
                if (par == 0) 
                {
                    return -1;
                }
                par--;
            } 
            else if (par > 0) // 忽略括号内的操作符
            {
                continue;
            } 
            else              // 处理括号外的操作符
            {
                int tmp_type = 0;
                switch (tokens[i].type) 
                {
                    case '*': case '/': tmp_type = 1; break;
                    case '+': case '-': tmp_type = 2; break;
                    default: assert(0);
                }
                if (tmp_type >= op_type) 
                {
                    op_type = tmp_type;
                    ret = i;
                }
            }
        }
        if (par != 0) return -1;
        return ret;
    }

    word_t eval(int p, int q, bool *ok) 
    {
        *ok = true;
        if (p > q) 
        {
            *ok = false;
            return 0;
        } 
        else if (p == q) 
        {
            if (tokens[p].type != TK_NUM) 
            {
                *ok = false;
                return 0;
            }
            word_t ret = strtol(tokens[p].str, NULL, 10);
            return ret;
        } 
        else if (check_parentheses(p, q)) 
        {
            return eval(p + 1, q - 1, ok);
        } 
        else 
        {    
            int major = find_major(p, q);
            if (major < 0) 
            {
                *ok = false;
                return 0;
            }

            word_t val1 = eval(p, major - 1, ok);
            if (!*ok) return 0;
            word_t val2 = eval(major + 1, q, ok);
            if (!*ok) return 0;
            
            switch(tokens[major].type) 
            {
                case '+': return val1 + val2;
                case '-': return val1 - val2;
                case '*': return val1 * val2;
                case '/': 
                    if (val2 == 0) 
                    {
                    *ok = false;
                    return 0;
                    } 
                    return (sword_t)val1 / (sword_t)val2; // e.g. -1/2, may not pass the expr test
                default: assert(0);
            }
        }
    }
    ```

2. 任务 PA1.2: 实现算术表达式求值
3. 任务 PA1.3: 实现所有要求, 提交完整的实验报告

## PA2 冯诺依曼计算机系统

## PA3 批处理系统

## PA4 分时多任务