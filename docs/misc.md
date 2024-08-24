## mkdocs

* 从头教你用`mkdocs`构建个人博客系统: [blog.csdn.net](https://blog.csdn.net/qq_41261251/article/details/116021097).
* My notes: [local网址](http://127.0.0.1:8000/), [public网址](https://lingyu11.github.io/notes/).
* 本地文件存放目录为`C:\Users\chaofu\ChaoNotes`, 对应于GitHub仓库为：
    - [源文档](https://github.com/lingyu11/notes/tree/master)存放在`master`分支
    - [site目录下的文档](https://github.com/lingyu11/notes/tree/gh-pages)存放在`gh-pages`分支，用于部署
* 如何更新笔记：
    - dd



## Commands

* `mkdocs new [dir-name]` - Create a new project.
* `mkdocs serve` - Start the live-reloading docs server.
* `mkdocs build` - Build the documentation site.
* `mkdocs -h` - Print help message and exit.

## Project layout

    mkdocs.yml    # The configuration file.
    docs/
        index.md  # The documentation homepage.
        ...       # Other markdown pages, images and other files.
