## Useful cmds

1. `find . | grep '\.cpp$' | xargs cat | wc -l`
    * 解释：统计当前目录下所有以 .cpp 为后缀的文件的总行数。
    * `find .`：查找当前目录下的所有文件和目录。
    * `grep '\.cpp$'`：使用 grep 命令过滤出以 .cpp 为后缀的文件。`\.` 表示匹配点号，`$` 表示匹配行尾。
    * `xargs cat`：使用 xargs 命令将过滤后的文件名传递给 cat 命令。cat 命令用于读取文件内容并输出。
    * `wc -l`：使用 wc 命令统计输入的行数。-l 选项表示只统计行数。

2. `history | tr -s ' ' | cut -d ' ' -f3 | sort | uniq -c | sort -nr`
    * 解释：统计命令行命令的频率。
    * `history`：显示当前用户的命令历史记录。
    * `tr -s ' '`：使用 tr 工具来压缩（或删除）输入中的重复空格。这使得每个单词之间只有一个空格，无论原始输入中有多少个空格。tr 代表 "translate"（转换），而 -s 选项则表示 "squeeze"（压缩）。 当使用 tr -s 时，它会将输入中的重复字符序列压缩成单个字符。
    * `cut -d ' ' -f3`：使用 cut 工具从输入中提取第三个字段（以空格为分隔符`-d ' '`）。f1是history输出的空格，f2是命令出现的编号，f3才是命令。
    * `sort`：对输入进行排序。在这种情况下，它会按照字母顺序对提取的命令进行排序。
    * `uniq -c`：删除相邻的重复行，-c表示每行的前面显示出现的次数。
    * `sort -nr`：按照数值（-n）从大到小（-r）对输出进行排序，这使得最常用的命令出现在列表的顶部。

    ```bash
    linux:~$ history
        1  history | tr -s ' ' | cut -d ' ' -f3 | sort | uniq -c | sort -nr
        2  ls
        3  history | tr -s ' ' | cut -d ' ' -f3 | sort | uniq -c | sort -nr
        4  cat
        5  history | tr -s ' ' | cut -d ' ' -f3 | sort | uniq -c | sort -nr
        6  history
    linux:~$ history | tr -s ' '
        1 history | tr -s ' ' | cut -d ' ' -f3 | sort | uniq -c | sort -nr
        2 ls
        3 history | tr -s ' ' | cut -d ' ' -f3 | sort | uniq -c | sort -nr
        4 cat
        5 history | tr -s ' ' | cut -d ' ' -f3 | sort | uniq -c | sort -nr
        6 history
        7 history | tr -s ' '
    linux:~$ history | tr -s ' ' | cut -d ' ' -f3
        history
        ls
        history
        cat
        history
        history
        history
        history
    linux:~$ history | tr -s ' ' | cut -d ' ' -f3 | sort
        cat
        history
        history
        history
        history
        history
        history
        history
        ls
    linux:~$ history | tr -s ' ' | cut -d ' ' -f3 | sort | uniq -c
        1 cat
        8 history
        1 ls
    linux:~$ history | tr -s ' ' | cut -d ' ' -f3 | sort | uniq -c | sort -nr
        9 history
        1 ls
        1 cat
    linux:~$ 
    ```

3. `command1 |& command2`: 会将`command1`的标准输出和标准错误输出合并，然后将合并后的输出传递给`command2`

4. `xxd -r -p`:
    * 解释：将输入的纯十六进制数据转换为二进制数据。
    * `-r`: 表示反向操作，即从十六进制转换为二进制。
    * `-p`: 表示以纯十六进制格式读取输入，即不包含任何额外的格式信息，只读取十六进制数据。

5. `find . -name "*.[ch]" | xargs grep "#include" | sort | uniq`:
    * 列出一个C语言项目中所有被包含过的头文件。

6. `find . -name "*.c" -o -name "*.h" | xargs cat | wc -l`
    * `-o`: 表示 "or"，表示同时匹配多个条件。
    * 统计当前目录下所有 .c 和 .h 文件的代码行数。
    * 等价于 `find . -name "*.[ch]" | xargs cat | wc -l`
