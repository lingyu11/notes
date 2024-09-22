## Useful cmds

1. `find . | grep '\.cpp$' | xargs cat | wc -l`
    * 解释：统计当前目录下所有以 .cpp 为后缀的文件的总行数。
    * `find .`：查找当前目录下的所有文件和目录。
    * `grep '\.cpp$'`：使用 grep 命令过滤出以 .cpp 为后缀的文件。`\.` 表示匹配点号，`$` 表示匹配行尾。
    * `xargs cat`：使用 xargs 命令将过滤后的文件名传递给 cat 命令。cat 命令用于读取文件内容并输出。
    * `wc -l`：使用 wc 命令统计输入的行数。-l 选项表示只统计行数。

