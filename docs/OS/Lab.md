# Lab

!!! tip

    github 上搜索 os-workbench，可以看到一些参考实现。例如 [os-workbench](https://github.com/phoulx/nju-os-workbench/blob/main/M3%20-%20gpt/gpt.c)，还可以直接搜索实验查看到一些 [blog](https://jiaweihawk.github.io/2021/12/05/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F-%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0-%E5%85%AB/) 等。

## [M1: 打印进程树 (pstree)](https://jyywiki.cn/OS/2024/labs/M1.md)
1. 首先观察到Linux的pstree是这样实现的：打开 `/proc/pid/stat`，然后读取里面的信息。其次注意到 `pstree -V /dev/null` 依然输出了，是 fprintf 到 stderr 实现的。

    ```bash
    linux$ strace pstree
    openat(AT_FDCWD, "/proc/30206/stat", O_RDONLY) = 4
    newfstatat(AT_FDCWD, "/proc/30206", {st_mode=S_IFDIR|0555, st_size=0, ...}, 0) = 0
    ...
    linux$ pstree -V /dev/null
    pstree (PSmisc) 23.4
    Copyright (C) 1993-2020 Werner Almesberger and Craig Small

    PSmisc comes with ABSOLUTELY NO WARRANTY.
    This is free software, and you are welcome to redistribute it under
    the terms of the GNU General Public License.
    For more information about these matters, see the files named COPYING.
    ```
2. pstree.c

    ```C
    #include <stdio.h>
    #include <assert.h>
    #include <stdint.h>
    #include <ctype.h>
    #include <stdbool.h>
    #include <stdlib.h>
    #include <string.h>
    #include <dirent.h>
    //思路：先从1号进程为根进行搜索建树，然后搜索/proc的其他文件夹，把不在树中的加入进去

    #define pid_t int

    bool P = false;
    bool N = false;
    bool V = false;

    typedef struct childs {
        pid_t pid;
        struct childs* next;
    } childs;

    typedef struct proc {
        char name[128];
        pid_t pid;
        pid_t ppid;
        int child_num;
        pid_t child_pid[128];
        struct proc *next;
    } proc;

    proc root_proc= {.pid = 1, .ppid = 0, .child_num = 0, .next = NULL};
    proc *root = &root_proc;
    char path1[600];
    char path2[600];
    char tmp;


    bool is_num(char *str) {
        for (int i = 0; i < strlen(str); i++) {
            if (isdigit(str[i]) == 0) return false;
        }
        return true;
    }

    proc *get_last_proc() {
        proc *pr = root;
        while (pr->next != NULL) pr = pr->next;
        return pr;
    }

    proc *find_proc(int pid) {
        for (proc *p = root; p != NULL; p = p->next) {
            if (p->pid == pid) {
                return p;
            }
        }
        return NULL;
    }

    void print_tree(proc *p){
        if (P) {
            printf("%s(%d)", p->name, p->pid);
        } 
        else {
            printf("%s", p->name);
        }

        if (p->child_num == 0) {
            printf("\n");
            return;
        }
        else
        {
            printf(" - ");
        }

        for (int i = 0; i < p->child_num; i++) {
            proc *child = find_proc(p->child_pid[i]);
            if (child) {
                print_tree(child);
            }
        }
    }


    void get_procs() {
        DIR *dir = opendir("/proc");
        struct dirent *dire;
        while ((dire = readdir(dir)) != NULL) {
            if (dire->d_type == 4 && is_num(dire->d_name)) { // 检查类型是否为目录，以及是否只包含数字(pid)
                sprintf(path1, "/proc/%s/stat", dire->d_name);
                sprintf(path2, "/proc/%s/task/%s/children", dire->d_name, dire->d_name);
                FILE *fp = fopen(path1, "r");
                if (strcmp(dire->d_name, "1") == 0) { // 1号进程
                    if (fscanf(fp,"%d (%s %c %d", &root->pid, root->name, &tmp, &root->ppid))
                    {
                        root->name[strlen(root->name) - 1] = '\0';
                        fclose(fp);
                        fp = fopen(path2, "r");
                        pid_t child_pid;
                        while (fscanf(fp, "%d", &child_pid) != EOF) {
                            root->child_pid[root->child_num++] = child_pid;
                            assert(root->child_num <= 128);
                        }
                    }
                }
                else { // 其他进程
                    proc *last_proc = get_last_proc();
                    proc *cur_proc = malloc(sizeof(proc));
                    if (fscanf(fp,"%d (%s %c %d", &cur_proc->pid, cur_proc->name, &tmp, &cur_proc->ppid))
                    {
                        cur_proc->name[strlen(cur_proc->name) - 1] = '\0';
                        last_proc->next = cur_proc;
                        fclose(fp);
                        fp = fopen(path2, "r");
                        pid_t child_pid;
                        cur_proc->child_num = 0;
                        cur_proc->next = NULL;
                        while (fscanf(fp, "%d", &child_pid) != EOF) {
                            cur_proc->child_pid[cur_proc->child_num++] = child_pid;
                            assert(cur_proc->child_num <= 128);
                        }
                    }
                }
                fclose(fp); 
            }
        }
    }

    int main(int argc, char *argv[]) {
        for (int i = 0; i < argc; i++) {
            if (strcmp(argv[i], "-p") == 0 || strcmp(argv[i], "--show-pids") == 0) {
                P = true;
            }
            else if (strcmp(argv[i], "-n") == 0 || strcmp(argv[i], "--numeric-sort") == 0 ){
                N = true;
            }
            else if (strcmp(argv[i], "-V") == 0 || strcmp(argv[i], "--version") == 0) {
                V = true;
            }
        }
        if (V) {
            fprintf(stderr, "pstree version 1.0 CopyRight (C) 2024 Lingyu\n");
        }
        else {
            get_procs();
            print_tree(root);
        }
        return 0;
    }
    ```

## [M2: 协程库 (libco)](https://jyywiki.cn/OS/2024/labs/M2.md)

1. [`setjmp` 和 `longjmp`:](../ICS/编程与调试实践.md#setjmp_longjmp) `setjmp` 和 `longjmp` 是 C 语言标准库提供的一组用于非局部跳转的函数。它们通常一起使用，用于实现协程（coroutine）、异常处理机制或者在函数调用栈中进行长距离跳转。

    * 📥`setjmp` 函数用于保存当前的程序执行环境（上下文）到一个 `jmp_buf` 类型的缓冲区中。这个缓冲区通常是一个数组，用于存储寄存器的值、栈指针、程序计数器等关键信息。`setjmp` 函数的返回值有两种情况：

        - 如果是第一次调用 `setjmp`，它会返回 0，表示这是一个新的环境。
        - 如果是通过 `longjmp` 函数进行跳转，它会返回一个非零值，表示这是一个恢复的环境。

    * ↪️`longjmp` 函数用于从当前的执行点跳转到之前通过 `setjmp` 保存的环境中。它接受两个参数：

        - `jmp_buf` 类型的缓冲区，这个缓冲区是之前通过 `setjmp` 保存的。
        - 一个整数值，这个值会被用作 `setjmp` 的返回值。

    ```C
    #include <stdio.h>
    #include <setjmp.h>

    jmp_buf env;

    void foo() {
        printf("In foo()\n");
        longjmp(env, 1); // 跳转到 setjmp 处，并设置返回值为 1
    }

    int main() {
        int ret = setjmp(env); // 保存当前环境
        if (ret == 0) {
            printf("First time through\n");
            foo();
        } else {
            printf("Back in main via longjmp, ret = %d\n", ret);
        }
        return 0;
    }
    ```

    ```bash
    linux$ ./a.out 
    First time through
    In foo()
    Back in main via longjmp, ret = 1
    ```

2. co.c
```C
#include "co.h"
#include <stdint.h>
#include <stdio.h>
#include <setjmp.h>
#include <assert.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <time.h>

#define KB *1024
#define STACK_SIZE (64 KB)
// #define DEBUG 1
#ifdef DEBUG
#define Log(format, ...) \
    printf("\33[1;34m[%s, %d, %s] " format "\33[0m\n", \
        __FILE__, __LINE__, __func__, ## __VA_ARGS__)
#else
#define Log(format, ...)
#endif

static inline void
stack_switch_call(void *sp, void *entry, uintptr_t arg) {
  asm volatile (
#if __x86_64__
    "movq %0, %%rsp; movq %2, %%rdi; jmp *%1"
      :
      : "b"((uintptr_t)sp),
        "d"(entry),
        "a"(arg)
      : "memory"
    // 首先将新堆栈指针 sp 移动到 rsp 寄存器，然后将参数 arg 移动到 rdi 寄存器，最后跳转到 entry 指向的地址执行
#else
    "movl %0, %%esp; movl %2, 4(%0); jmp *%1"
      :
      : "b"((uintptr_t)sp - 8),
        "d"(entry),
        "a"(arg)
      : "memory"
#endif
    );
}

enum co_status {
  CO_NEW = 1, // 新创建，还未执行过
  CO_RUNNING, // 已经执行过
  CO_WAITING, // 在 co_wait 上等待
  CO_DEAD,    // 已经结束，但还未释放资源
};

struct co {
  const char *name;
  void (*func)(void *);   // co_start 指定的入口地址和参数
  void *arg;
  void *stackptr;
  enum co_status status;  // 协程的状态
  jmp_buf context;        // 寄存器现场 (setjmp.h)
  struct co *next;
  struct co *prev;
  struct co *waiter;
  uint8_t stack[STACK_SIZE] __attribute__ ((aligned(16))); // 协程的堆栈
};

struct co *co_main, *co_current;
int co_num = 0;

// __attribute__((constructor)) 属性的函数会在 main 执行前被运行
__attribute__((constructor)) void co_init() {
  co_main = malloc(sizeof(struct co));
  co_main->name = "main";
  co_main->stackptr = co_main->stack + sizeof(co_main->stack);
  co_main->status = CO_RUNNING;
  memset(co_main->stack, 0, sizeof(co_main->stack));
  co_main->next = co_main;
  co_main->prev = co_main;
  co_current = co_main;
  co_num += 1;
}

void debugprint() {
  struct co *p = co_main;
  printf("debugprint begin\n");
  for (int i = 0; i < co_num; i++) {
    Log("co: %s, co_num: %d", p->name, co_num);
    p = p->next;
  }
  printf("debugprint end\n");
}

// 创建一个新的协程
struct co* co_start(const char *name, void (*func)(void *), void *arg) {
  struct co *co_new = malloc(sizeof(struct co));
  co_new->name = name;
  co_new->func = func;
  co_new->arg = arg;
  co_new->status = CO_NEW;
  co_new->stackptr = co_new->stack + sizeof(co_new->stack);
  memset(co_new->stack, 0, sizeof(co_new->stack));
  struct co *prev = co_main->prev;
  co_new->next = co_main;
  co_main->prev = co_new;
  co_new->prev = prev;
  prev->next = co_new;
  co_num += 1;
  //debugprint();
  return co_new;
};

void wrapper() {
  co_current->status = CO_RUNNING;
  co_current->func(co_current->arg);
  co_current->status = CO_DEAD;
  if (co_current->waiter != NULL) {
    co_current->waiter->status = CO_RUNNING;
    co_current->waiter = NULL;
  }
  co_yield();
}

// 实现协程的切换
void co_yield() {
  int val = setjmp(co_current->context); // 将当前协程的上下文（寄存器状态等）保存到 co_current->context 中
  if (val == 0) { // 表示这是第一次调用 setjmp，此时我们需要选择下一个待运行的协程 (相当于修改 current)，并切换到这个协程运行
    struct co *next = co_current->next;
    while (next->status == CO_WAITING || next->status == CO_DEAD) next = next->next;
    co_current = next;
    if (next->status == CO_NEW) {
      if (sizeof(void*) == 4) stack_switch_call(next->stackptr, wrapper, (uintptr_t)NULL); // 32位系统
      else {                                                                               // 64位系统
        asm volatile("mov %0, %%rsp" :: "b"((uintptr_t)next->stackptr));
        wrapper();
      }
    }
    else { // next 不是新创建的，则调用 longjmp 函数恢复下一个协程的上下文，使其继续执行。
      longjmp(next->context, 1);
    } 
  }
  else { /* do nothing */}
  // setjmp 是由另一个 longjmp 返回的，此时一定是因为某个协程调用 co_yield()，此时代表了寄存器现场的恢复，
  // 因此不必做任何操作，直接返回即可。
}

// 表示当前协程需要等待，直到 co 协程的执行完成才能继续执行
void co_wait(struct co *co) {
  if (co->status == CO_DEAD) {
    Log("free %s", co->name);
    struct co *prev = co->prev;
    struct co *next = co->next;
    prev->next = next;
    next->prev = prev;
    free(co);
  }
  else {
    co->waiter = co_current;
    co_current->status = CO_WAITING;
    co_yield();
  }
}
```

## [M3: GPT-2 并行推理 (gpt.c)](https://jyywiki.cn/OS/2024/labs/M3.md)

## [M4: C Read-Eval-Print-Loop (crepl)](https://jyywiki.cn/OS/2024/labs/M4.md)
1. crepl.c
```C
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>
#include <unistd.h>
#include <assert.h>
#include <wait.h>
#include <dlfcn.h>
#include <stdbool.h>

#if defined(__i386__)
  #define TARGET "-m32"
#elif defined(__x86_64__)
  #define TARGET "-m64"
#endif

static char line[4096];
static char tmp[4];
static char src_filename[32];
static char dst_filename[32];
static int (*f)();
static int value;

void compile(bool func) {
  sprintf(src_filename, "/tmp/func_c_XXXXXX");
  sprintf(dst_filename, "/tmp/func_so_XXXXXX");
  if (mkstemp(src_filename) == -1) printf("\033[1;31m      Mkstemp Failed!\033[0m\n");
  if (mkstemp(dst_filename) == -1) printf("\033[1;31m      Mkstemp Failed!\033[0m\n");
  FILE *fp = fopen(src_filename, "w");
  if (func) fprintf(fp, "%s", line);
  else fprintf(fp, "int wrap_func() {return (%s);}", line);
  fclose(fp);
  char *exec_argv[] = {"gcc", TARGET, "-x", "c", "-fPIC", "-w", "-shared", "-o", dst_filename, src_filename, NULL};
  int pid = fork();
  if (pid == 0) {  // 子进程
    int fd = open("/dev/null", O_RDWR);
    dup2(fd, STDOUT_FILENO); // 将 fd 复制到标准输出（STDOUT_FILENO）和标准错误（STDERR_FILENO）
    dup2(fd, STDERR_FILENO); // 意味着子进程的标准输出和标准错误都将被重定向到 /dev/null，即丢弃所有输出
    execvp(exec_argv[0], exec_argv); // execve 接受一个路径名作为参数，execvp 接受一个文件名作为参数
    // example：execve("/bin/ls", args, envp);  execvp("ls", args);
  }
  else {           // 父进程
    int status;
    wait(&status); // 系统调用，用于等待子进程结束并获取其退出状态。
    if (status != 0) printf("\033[1;31m      Compile Error!\033[0m\n");
    else {
      // dlopen 函数: 加载一个动态链接库（.so 文件），并返回一个句柄，用于后续访问该库中的函数和变量
      void *handle = dlopen(dst_filename, RTLD_NOW | RTLD_GLOBAL);
      if (!handle) printf("\033[1;31m      Compile Error!\033[0m\n");
      else { 
        if (func) printf("\033[1;32m      Added: \033[1;30m%s\033[0m", line);
        else {
          f = dlsym(handle, "wrap_func"); // dlsym 函数: 从动态链接库中查找并获取一个符号的地址
          value = f();
          printf("\033[1;32m      Result: \033[1;30m%d\033[0m\n", value);
          dlclose(handle); 
        }
      }
    }
  }
  unlink(src_filename); // 系统调用，用于删除文件
  unlink(dst_filename);
}

int main(int argc, char *argv[]) {
  while (1) {
    printf("crepl> ");
    memset(line, '\0', sizeof(line));
    memset(tmp, '\0', sizeof(tmp));
    fflush(stdout);
    if (!fgets(line, sizeof(line), stdin)) {
      break;
    }
    sscanf(line, "%3s", tmp);
    compile(strncmp(line, "int", 3) == 0);
  }
}
```

## [M5: 系统调用 Profiler (sperf)](https://jyywiki.cn/OS/2024/labs/M5.md)
1. sperf.c
```C
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <assert.h>
#include <fcntl.h>
#include <regex.h>
#include <string.h>
#include <stdlib.h>
#include <time.h>
#include <ctype.h>
#include <dirent.h>
#include <stdbool.h>

#define BUFFER_SIZE 8192
#define PRINT_NUM 5
#define TIME_INTERVAL_SEC 1

// 链表
typedef struct list_node {
  char *name;
  double time;
  struct list_node *next;
} list_node;

// 链表插入节点，同时保证时间始终从大到小排序
list_node *insert_list_node(list_node *head, list_node *node) {
  if (node == NULL) {
    return head;
  }

  if (head == NULL || head->time < node->time) {
    if (head != NULL) {
      node->next = head;
    }
    return node;
  }

  list_node *prev = head;
  for (list_node *curr = head->next; curr != NULL; curr = curr->next) {
    if (curr->time < node->time) {
      break;
    }
    prev = curr;
  }

  // prev -> node -> curr (将 node 插入 prev 和 curr 之间)
  node->next = prev->next;
  prev->next = node;
  return head;
}

// 释放链表内存
void free_list(list_node *head) {
  list_node *curr = head;
  while (curr != NULL) {
    list_node *next = curr->next;
    if (curr->name != NULL) {
      free(curr->name);
    }
    curr->name = NULL;
    free(curr);
    curr = next;
  }
  return;
}


int main(int argc, char *argv[])  {
  int fd[2]; // fd[0]: 读口，fd[1]: 写口
  if (pipe(fd) == -1) {
    // 创建管道失败
    perror("pipe failed\n");
    exit(EXIT_FAILURE);
  }
  int pid = fork();
  if (pid == -1) {
    // 子进程创建失败
    perror("fork failed\n");
    exit(EXIT_FAILURE);
  } else if (pid == 0) {
    // 子进程，执行strace命令
    // 关闭管道读取端，并将管道写入端重定向到标准错误输出
    close(fd[0]);
    if (dup2(fd[1], STDERR_FILENO) == -1) {
      // 重定向失败
      perror("dup2 failed!\n");
      close(fd[1]);
      exit(EXIT_FAILURE);
    }
    // 执行strace
    char *exec_argv[argc + 2];
    exec_argv[0] = "strace";
    exec_argv[1] = "-T";
    for (int i = 1; i < argc; i++) {
      exec_argv[i + 1] = argv[i];
    }
    exec_argv[argc + 1] = NULL;

    char *exec_envp[] = {
        "PATH=/bin",
        NULL,
    };
    execve("strace", exec_argv, exec_envp);
    execve("/bin/strace", exec_argv, exec_envp);
    execve("/usr/bin/strace", exec_argv, exec_envp);
    perror(argv[0]); // 如果前面的三个 execve 调用都失败了，即没有
    // 找到 strace 命令或者执行 strace 命令失败，那么程序会执行到这里
    close(fd[1]);
    exit(EXIT_FAILURE);
  } else {
    // 父进程，读取strace输出并统计
    regex_t reg1, reg2; // reg1: 调用函数名  reg2: 花费时间
    if (regcomp(&reg1, "[^\\(\n\r\b\t]*\\(", REG_EXTENDED) ||
        regcomp(&reg2, "<.*>", REG_EXTENDED)) {
      perror("regcomp failed!\n");
      exit(EXIT_FAILURE);
    }
    // 链表记录各项调用时间信息
    list_node *head = NULL;
    // 调用总时间
    double total_time = 0;
    // 管道读取缓存
    char buffer[BUFFER_SIZE];
    // 关闭管道写入端，并从管道中读取数据
    close(fd[1]);
    FILE *fp = fdopen(fd[0], "r");
    // 当前时间
    int curr_time = 1;
    // 运行标志
    int run_flag = 1;
    while (run_flag) {
      while (fgets(buffer, BUFFER_SIZE, fp) > 0) {
        // 读取到 +++ exited with 1/0 +++ 退出
        if (strstr(buffer, "+++ exited with 0 +++") != NULL ||
            strstr(buffer, "+++ exited with 1 +++") != NULL) {
          run_flag = 0;
          break;
        }
        // 使用正则表达式获取信息
        regmatch_t regmat1, regmat2;
        if (regexec(&reg1, buffer, 1, &regmat1, 0) ||
            regexec(&reg2, buffer, 1, &regmat2, 0)) {
          continue;
        }
        // 读取调用名称信息
        int len = 0;
        len = regmat1.rm_eo - regmat1.rm_so;
        char *name = (char *)malloc(sizeof(char) * len);
        strncpy(name, buffer + regmat1.rm_so, len);
        name[len - 1] = '\0';
        // 读取时间信息
        len = regmat2.rm_eo - regmat2.rm_so - 2;
        char *value = (char *)malloc(sizeof(char) * len);
        strncpy(value, buffer + regmat2.rm_so + 1, len);
        double spent_time = atof(value);
        // 及时释放value内存
        free(value);
        value = NULL;
        // 更新总时间
        total_time += spent_time;
        // 将信息保存到链表中
        list_node *node = NULL;
        int find_flag = 0;
        list_node *prev = NULL;
        for (list_node *curr = head; curr != NULL; curr = curr->next) {
          // 若信息已存在，则将其更新后，重新插入
          if (strcmp(curr->name, name) == 0) {
            find_flag = 1;
            if (prev == NULL) {
              head = curr->next;
            } else {
              prev->next = curr->next;
            }
            node = curr;
            // 信息存在，及时释放name内存
            free(name);
            name = NULL;
            break;
          }
          prev = curr;
        }

        if (find_flag) {
          // 更新信息
          node->time += spent_time;
        } else {
          // 新建节点，初始化信息
          node = (list_node *)malloc(sizeof(list_node));
          node->name = name;
          node->time = spent_time;
        }
        // 插入链表
        head = insert_list_node(head, node);
      }

      // 格式化打印前五占比调用信息
      printf("Time: %ds\n", curr_time);
      int k = 0;
      for (list_node *node = head; node != NULL; node = node->next) {
        if (k++ >= PRINT_NUM) {
          break;
        }
        printf("%s (%0.1f%%)\n", node->name, node->time / total_time * 100);
      }
      printf("=============\n");
      curr_time += TIME_INTERVAL_SEC;
      sleep(TIME_INTERVAL_SEC);
    }
    // 关闭管道读取端，释放相关资源
    regfree(&reg1);
    regfree(&reg2);
    fclose(fp);
    close(fd[0]);
    free_list(head);
    exit(EXIT_SUCCESS);
  }
}
```

2. output
```bash
linux$ ./sperf-64 ls
Makefile  sperf-32  sperf-64  sperf.c
Time: 1s
mmap (21.5%)
mprotect (12.6%)
openat (7.5%)
close (6.8%)
newfstatat (6.1%)
=============
```

## [M6: 文件系统格式化恢复 (fsrecov)](https://jyywiki.cn/OS/2024/labs/M6.md)
1. FAT（File Allocation Table，文件分配表）是一种用于管理磁盘文件存储的文件系统。BPB（BIOS Parameter Block，BIOS参数块）是FAT文件系统中的一个重要组成部分，它包含了文件系统的关键参数和设置。

    BPB通常位于磁盘的第一个扇区（引导扇区），它提供了文件系统的基本信息，如每扇区字节数、每簇扇区数、保留扇区数、FAT表个数、根目录项数等。这些参数对于操作系统正确访问和管理文件系统至关重要。

2. fsrecov.c
```C
#include <stdio.h>
#include <assert.h>
#include <stdint.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <sys/mman.h>
#include <string.h>
#include <unistd.h>
#include <sys/stat.h>
#include <stdbool.h>
#include <ctype.h>
#include <math.h>
#include <time.h>

#define B 1

//#define __DEBUG__
//#define __DDEBUG__

#define KB (1024 * B)
#define MB (1024 * KB)
#define GB (1024 * KB)

#ifdef __DEBUG__
#define print(...) printf(__VA_ARGS__)
#else
#define print(...) 
#endif

#ifdef __DDEBUG__
#define printk(...) printf(__VA_ARGS__)
#else
#define printk(...) 
#endif

#define Unknown (-1)
#define DirEntry 1
#define beNotUsed 2
#define BMPContent 3

// FAT 的 BPB(BIOS Parameter Block) 是FAT文件系统中的一个重要组成部分，
// 它包含了文件系统的关键参数和设置。
int BPB_BytsPerSec;
int BPB_SecPerClus;
int BPB_RootClus;
int BPB_FATSz32;
int BPB_HiddSec;
int BPB_RsvdSecCnt;
int BPB_NumFATs;
int offset;
int clusSize;

inline bool inFile(const void* nowAddr, const void* fileStart, int fileSize) {
    return ((intptr_t)(nowAddr) - (intptr_t)(fileStart)) < fileSize ? true : false;
}
inline int getClusterIndex(const void* addr, const void* start, int clusterSize) {
    return ((intptr_t)addr - (intptr_t) start) / clusterSize;
}
inline void* nextClus(const void* Clus) {
    return (void*)((intptr_t)(Clus) + BPB_BytsPerSec * BPB_SecPerClus);
}

struct FATHeader {
    uint8_t  BS_jmpBoot[3];
    uint8_t  BS_OEMName[8];
    uint16_t BPB_BytsPerSec;
    uint8_t  BPB_SecPerClus;
    uint16_t BPB_RsvdSecCnt;
    uint8_t  BPB_NumFATs;
    uint16_t BPB_RootEntCnt;
    uint16_t BPB_TotSec16;
    uint8_t  BPB_Media;
    uint16_t BPB_FATSz16;
    uint16_t BPB_SecPerTrk;
    uint16_t BPB_NumHeads;
    uint32_t BPB_HiddSec;
    uint32_t BPB_TotSec32;
    uint32_t BPB_FATSz32;
    uint16_t BPB_ExtFlags;
    uint16_t BPB_FSVer;
    uint32_t BPB_RootClus;
    uint16_t BPB_FSInfo;
    uint16_t BPB_BkBootSec;
    uint8_t  BPB_Reserved[12];
    uint8_t  BS_DrvNum;
    uint8_t  BS_Reserved;
    uint8_t  BS_BootSig;
    uint32_t BS_VolID; 
    uint8_t  BS_VolLab[11];
    uint8_t  BS_FilSysType0;
    uint8_t  BS_FilSysType1;
    uint8_t  BS_FilSysType2;
    uint8_t  BS_FilSysType3;
    uint8_t  BS_FilSysType4;
    uint8_t  BS_FilSysType5;
    uint8_t  BS_FilSysType6;
    uint8_t  BS_FilSysType7;
    uint8_t  padding[420];
    uint16_t Signature_word;
}__attribute__((packed));

struct BMPHeader {
    uint8_t  bfType[2];
    uint32_t bfSize;
    uint16_t bfReserved1;
    uint16_t bfReserved2;
    uint32_t bfOffBits;
}__attribute__((packed));

struct BMPInfoHeader {
    uint32_t biSize;
    int32_t  biWidth;
    uint32_t biHeight;
    uint16_t biPlanes;
    uint16_t biBitCount;
    uint32_t biCompression;
    uint32_t biSizeImage;
    int32_t  biXPelsPerMeter;
    int32_t  biYPelsPerMeter;
    uint32_t biClrUsed;
    uint32_t biClrImportant;
}__attribute__((packed));

struct FATShortDirectory {
    uint8_t  DIR_Name[11];
    uint8_t  DIR_Attr;
    uint8_t  DIR_NTRes;
    uint8_t  DIR_CrtTimeTenth;
    uint16_t DIR_CrtTime;
    uint16_t DIR_CrtDate;
    uint16_t DIR_LstAccDate;
    uint16_t DIR_FstClusHI;
    uint16_t DIR_WrtTime;
    uint16_t DIR_WrtDate;
    uint16_t DIR_FstClusLO;
    uint32_t DIR_FileSize;
}__attribute__((packed));

struct FATLongDirectory {
    uint8_t  LDIR_Ord;
    uint16_t LDIR_Name1[5];
    uint8_t  LDIR_Attr;
    uint8_t  LDIR_Type;
    uint8_t  LDIR_Chksum;
    uint16_t LDIR_Name2[6];
    uint16_t LDIR_FstClusLO;
    uint16_t LDIR_Name3[2];
}__attribute__((packed));

bool isFATLongDirectory(const struct FATLongDirectory* pFATldir);
bool isFATShortDirectory(const struct FATShortDirectory*);
char* readCompleteInfoFromFATShortDirectory(struct FATShortDirectory* pFATsd);
static inline struct FATLongDirectory* nextLongDirectory(struct FATLongDirectory* longDirectory){
    return (struct FATLongDirectory*)((intptr_t)(longDirectory) + sizeof(struct FATLongDirectory));
    
}
double sobelY(uint8_t* lowerline, uint8_t* nowline, uint8_t* higherline, int pixels);
static inline struct FATShortDirectory* nextShortDirectory(struct FATShortDirectory* shortDirectory){
    return (struct FATShortDirectory*)((intptr_t)(shortDirectory) + sizeof(struct FATShortDirectory));
}
bool isValidFileName(char* name);
int dirClus[100];

void* getClusterFromIndex(int index_from_zero, void* initialClusterAddr) {
    return initialClusterAddr + index_from_zero * clusSize;
}

void initAttr(struct FATHeader* pfatheader) {
    BPB_BytsPerSec = pfatheader->BPB_BytsPerSec;
    BPB_SecPerClus = pfatheader->BPB_SecPerClus;
    BPB_RootClus = pfatheader->BPB_RootClus;
    BPB_FATSz32 = pfatheader->BPB_FATSz32;
    BPB_HiddSec = pfatheader->BPB_HiddSec;
    BPB_RsvdSecCnt = pfatheader->BPB_RsvdSecCnt;
    BPB_NumFATs = pfatheader->BPB_NumFATs;
    clusSize = BPB_SecPerClus * BPB_BytsPerSec;
}

void dirClusAdd(int index) {
    assert(index >= 0);
    for (int i = 0; i < 100; i++) {
        if (dirClus[i] == -1) {
            dirClus[i] = index;
            return;
        }
    }
    assert(0);
}

int main (int argc, char* argv[]) {
    srand(time(NULL));
    char* imgName = argv[1];
    struct stat statbuf;
    stat(imgName, &statbuf);    // 获取img文件的状态信息
    int imgSize = statbuf.st_size;
    int imgFd = open(imgName, O_RDONLY, 0);
    struct FATHeader* pFATHeader = (struct FATHeader *) mmap(NULL, imgSize, PROT_READ, MAP_SHARED, imgFd, 0);
    uint8_t picture[4 * MB];
    uint8_t lowerline[128 * KB];
    uint8_t nowline[128 * KB];
    uint8_t higherline[128 * KB];
    uint8_t tmpnowline[128 * KB];
    uint8_t tmphigherline[128 * KB];
    BPB_BytsPerSec = 512;
    BPB_SecPerClus = 8;
    BPB_RootClus   = pFATHeader->BPB_RootClus;
    BPB_FATSz32    = pFATHeader->BPB_FATSz32;
    BPB_HiddSec    = pFATHeader->BPB_HiddSec;
    BPB_RsvdSecCnt = pFATHeader->BPB_RsvdSecCnt;
    BPB_NumFATs    = pFATHeader->BPB_NumFATs;
    clusSize       = BPB_BytsPerSec * BPB_SecPerClus;
    // 计算图像数据在文件中的偏移量。保留扇区、FAT 表、根目录(簇号从2开始)、隐藏扇区、图像
    int imgOffset = (BPB_RsvdSecCnt + BPB_NumFATs * BPB_FATSz32 + (BPB_RootClus - 2) * BPB_SecPerClus + BPB_HiddSec) * BPB_BytsPerSec;
    struct FATShortDirectory* pFATshdir = (void* )pFATHeader + imgOffset;
    void* imgDataStart = (void* )pFATHeader + imgOffset;
    int imgDataSize = imgSize - imgOffset;
    int tmpi = 0;
    int clusNum = imgDataSize / clusSize;
    int* cluses = malloc(sizeof(int) * clusNum); // 在内存中动态分配一个整数数组，用于imgData中每个簇的状态
    for (int i = 0; i < clusNum; i++) 
        cluses[i] = Unknown;
    // 遍历imgData中的簇，并检查每个簇中的短目录项（FATShortDirectory）和长目录项（FATLongDirectory）
    // 这个循环的主要目的是识别和标记那些包含目录项的簇
    for (void* cluster = imgDataStart; inFile(cluster, imgDataStart, imgDataSize); cluster = nextClus(cluster)) {
        int countsh = 0; // 短目录项数量
        int countl = 0;  // 长目录项数量
        for (struct FATShortDirectory* ptmp = cluster; inFile(ptmp, cluster, clusSize); ptmp++) {
            if (isFATShortDirectory(ptmp)) {
                char nameTmp[12];
                memcpy(nameTmp, ptmp->DIR_Name, 11);
                nameTmp[11] = '\0';
                tmpi++;
                countsh++;
                if (isFATLongDirectory((void*)(ptmp - 1)))
                    countl++;
                if (isFATLongDirectory((void*)(ptmp - 2)))
                    countl++;
            }
        }
        if (countsh > 6 && countl >= countsh) {
            int index = getClusterIndex(cluster, imgDataStart, clusSize);
            cluses[index] = DirEntry;
        }
    }
    for (int i = 0; i < clusNum; i++) {
        if (cluses[i] == DirEntry) {
            void* cluster = getClusterFromIndex(i, imgDataStart);
            int j = 0;
            for (struct FATShortDirectory* ptmp = cluster; inFile(ptmp, cluster, clusSize); ptmp++) {
                if (isFATShortDirectory(ptmp) && isFATLongDirectory((void*)(ptmp - 1))) {
                    char* name = readCompleteInfoFromFATShortDirectory(ptmp);
                    if (name != NULL) {
                        char* prefix = "/tmp/";
                        int picNameSize = strlen(prefix) + strlen(name) + 1;
                        char* abspath = malloc(sizeof(char) * picNameSize);
                        memset(abspath, '\0', picNameSize);
                        strcat(strcat(abspath, prefix), name);

                        struct BMPHeader* picStart = (void*) ((uintptr_t)pFATHeader + imgOffset + (ptmp->DIR_FstClusLO - BPB_RootClus) * clusSize);
                        FILE* pfdpic = fopen(abspath, "w+");
                        fwrite(picStart, 1, sizeof(*picStart), pfdpic);
                        fclose(pfdpic);
                        pfdpic = fopen(abspath, "a+");
                        struct BMPInfoHeader* picInfo = (struct BMPInfoHeader*)(picStart + 1);
                        fwrite(picInfo, 1, picStart->bfOffBits - sizeof(*picStart), pfdpic);
                        fclose(pfdpic);

                        pfdpic = fopen(abspath, "a+");
                        void* picData = (struct BMPInfoHeader*)(picInfo + 1);
                        int picDataSize = picStart->bfSize - picStart->bfOffBits;
                        
                        int ByteperPixel = picInfo->biBitCount / 8;
                        int picHeight = picInfo->biHeight;
                        int realWidthSize = (picInfo->biWidth * picInfo->biBitCount + 31) / 32 * 4;
                        void* source = NULL;
                        source = picData;
                        int r = rand();
                        // 通过Sobel边缘检测算法来优化图像的边缘质量，特别是在数据跨簇存储的情况下，通过寻找最优的簇组合来减少边缘的不连续性。
                        for (int i = 0; i < picHeight; i++) {
                            memcpy(nowline, source + i * realWidthSize, realWidthSize);
                            if (i != 0) { // 只对非第一行的行进行处理
                                if (r % 10 > 1) {
                                    if (getClusterIndex(source + i * realWidthSize, imgDataStart, clusSize) != getClusterIndex(source + (i + 1) * realWidthSize, imgDataStart, clusSize)) {
                                        int nowIndex = getClusterIndex(source + i * realWidthSize, imgDataStart, clusSize);
                                        int nowLength = (intptr_t)(getClusterFromIndex(nowIndex + 1, imgDataStart)) - (intptr_t)(source + i * realWidthSize);
                                        
                                        int requiredLength = realWidthSize - nowLength;
                                        int pixels = realWidthSize / ByteperPixel;
                                        double mean = sobelY(lowerline, nowline, higherline, pixels);
                                        if (mean > 11500) {
                                            double tmpLow = mean;
                                            int tmpLowIndex = -1;
                                            for (int j = 0; j < clusNum; j++) {
                                                void* tmpcluster = getClusterFromIndex(j, imgDataStart);
                                                memcpy(tmpnowline, nowline, nowLength);
                                                memcpy(tmpnowline + nowLength, tmpcluster, requiredLength);
                                                double tmpd = sobelY(lowerline, tmpnowline, tmphigherline, pixels);
                                                if (tmpd < tmpLow) {
                                                    tmpLow = tmpd;
                                                    tmpLowIndex = j;
                                                }
                                            }
                                            void* newCluster = getClusterFromIndex(tmpLowIndex, imgDataStart);
                                            source = newCluster - i * realWidthSize + requiredLength - realWidthSize;
                                            memcpy(nowline+nowLength, newCluster, requiredLength);
                                            
                                        }
                                    }
                                }
                            }
                            memcpy(picture + i * realWidthSize, nowline, realWidthSize);
                            memcpy(lowerline, nowline, realWidthSize);
                        }
                        // 结束

                        fwrite(picture, 1, picDataSize, pfdpic);
                        fclose(pfdpic);
                        char buf[41];
                        buf[40] = 0;
                        char cmd[100];
                        memset(cmd, 0, 100);
                        int pipefds[2];
                        if (pipe(pipefds) < 0) {
                            assert(0);
                        }
                        int pid = fork();
                        char* argv[3];
                        argv[0] = "sha1sum";
                        argv[1] = abspath;
                        argv[2] = NULL;
                        if (pid == 0) {
                            close(pipefds[0]);
                            dup2(pipefds[1], fileno(stderr));
                            dup2(pipefds[1], fileno(stdout));
                            execvp("sha1sum", argv);
                        } else {
                            close(pipefds[1]);
                            read(pipefds[0], buf, 40);
                            printf("%s   %s\n", buf, name);
                            fflush(stdout);
                        }
                    }
                }
            }
        }
    }
    return 0;
}

bool isFATShortDirectory(const struct FATShortDirectory* ptmp) {
    if (strncmp((char*)&ptmp->DIR_Name[8], "BMP", 3) == 0) {
        return true;
    }
    return false;
}

bool isFATLongDirectory(const struct FATLongDirectory* pFATldir) {
    // 检查 pFATldir->LDIR_Ord 的高 4 位是否既不是 0x4 也不是 0
    if ((pFATldir->LDIR_Ord >> 4) != 0x4 && (pFATldir->LDIR_Ord >> 4) != 0) {
        return false;
    }
    if (pFATldir->LDIR_FstClusLO != 0 || pFATldir->LDIR_Type != 0)
        return false;
    else    
        return true;
}

char* readCompleteInfoFromFATShortDirectory(struct FATShortDirectory* pFATsd) {
    
    char* name = malloc(sizeof(char) * 200);
    memset(name, '\0', 200);
    struct FATLongDirectory* pFATld = (struct FATLongDirectory*) (pFATsd-1);
    int i = -1;
    if ((pFATld->LDIR_Ord >> 4) != 0x4 && (pFATld->LDIR_Ord >> 4) != 0) {
        return NULL;
    }
    while ((pFATld->LDIR_Ord >> 4) == 0) {
        i += 1;
        name[i*13+0] = (char) pFATld->LDIR_Name1[0];
        name[i*13+1] = (char) pFATld->LDIR_Name1[1];
        name[i*13+2] = (char) pFATld->LDIR_Name1[2];
        name[i*13+3] = (char) pFATld->LDIR_Name1[3];
        name[i*13+4] = (char) pFATld->LDIR_Name1[4];
        name[i*13+5] = (char) pFATld->LDIR_Name2[0];
        name[i*13+6] = (char) pFATld->LDIR_Name2[1];
        name[i*13+7] = (char) pFATld->LDIR_Name2[2];
        name[i*13+8] = (char) pFATld->LDIR_Name2[3];
        name[i*13+9] = (char) pFATld->LDIR_Name2[4];
        name[i*13+10] = (char) pFATld->LDIR_Name2[5];
        name[i*13+11] = (char) pFATld->LDIR_Name3[0];
        name[i*13+12] = (char) pFATld->LDIR_Name3[1];
        pFATld = pFATld-1;
    }
    i += 1;
    name[i*13+0] = (char) pFATld->LDIR_Name1[0];
    name[i*13+1] = (char) pFATld->LDIR_Name1[1];
    name[i*13+2] = (char) pFATld->LDIR_Name1[2];
    name[i*13+3] = (char) pFATld->LDIR_Name1[3];
    name[i*13+4] = (char) pFATld->LDIR_Name1[4];
    name[i*13+5] = (char) pFATld->LDIR_Name2[0];
    name[i*13+6] = (char) pFATld->LDIR_Name2[1];
    name[i*13+7] = (char) pFATld->LDIR_Name2[2];
    name[i*13+8] = (char) pFATld->LDIR_Name2[3];
    name[i*13+9] = (char) pFATld->LDIR_Name2[4];
    name[i*13+10] = (char) pFATld->LDIR_Name2[5];
    name[i*13+11] = (char) pFATld->LDIR_Name3[0];
    name[i*13+12] = (char) pFATld->LDIR_Name3[1];
    for (int j = 12; j > -1; j--) {
        if (name[i*13+j] == 'p' && name[i*13+j-1] == 'm' && name[i*13+j-2] == 'b' && name[i*13+j-3] == '.' ) {
            name[i*13+j+1]= 0;
            break;
        }
    }
    for (int i = 0; i < strlen(name); i++) {
        if (!isprint(name[i]))
            return NULL;
    }
    return name;
}

int comp(const void* a, const void* b) {
    return *(double*)a - *(double*)b;
}

double sobelY(uint8_t* lowerline, uint8_t* nowline, uint8_t* higherline, int pixels) {
    double r, g, b, sum;
    sum = r = g = b = 0;
    for (int i = 0; i < pixels; i++) {
        r = nowline[i*3+0]-lowerline[i*3+0];
        g = nowline[i*3+1]-lowerline[i*3+1];
        b = nowline[i*3+2]-lowerline[i*3+2];
        sum += pow(r,2)+pow(g,2)+pow(b,2);
    }
    double mean;
    mean = sum/pixels;
    return mean;
}
```

3. output:
```bash
linux$ ./fsrecov-64 /tmp/fsrecov.img | grep C0NrniO8Tu5T.bmp
a9d9d319d3ca10870e31d1595f00d5c27dee7f1b  C0NrniO8Tu5T.bmp
linux$ cd /mnt/DCIM && sha1sum *.bmp | grep C0NrniO8Tu5T.bmp
a9d9d319d3ca10870e31d1595f00d5c27dee7f1b  C0NrniO8Tu5T.bmp
```



## [L0: 为计算机硬件编程](https://jyywiki.cn/OS/2024/labs/L0.md)

## [L1: 物理内存管理 (pmm)](https://jyywiki.cn/OS/2024/labs/L1.md)

## [L2: 内核线程管理 (kmt)](https://jyywiki.cn/OS/2024/labs/L2.md)