## mkdocs

* 从头教你用`mkdocs`构建个人博客系统: [blog.csdn.net](https://blog.csdn.net/qq_41261251/article/details/116021097).
* Mkdocs Material使用学习：[Mkdocs Material](https://shafish.cn/blog/mkdocs/)
* My notes: [local页面](http://127.0.0.1:8000/)，[public项目页面](https://lingyu11.github.io/notes/)，[public个人页面](https://lingyu11.github.io/).
* 本地文件存放目录为`C:\Users\username\ChaoNotes`，对应于GitHub仓库为：
    - [源文档](https://github.com/lingyu11/notes/tree/master)存放在`master`分支
    - [site目录下的文档](https://github.com/lingyu11/notes/tree/gh-pages)存放在`gh-pages`分支，用于部署
* 个人页面存放目录为`C:\Users\username\lingyu11.github.io`，对应于GitHub仓库为：
    - [site目录下的文档](https://github.com/lingyu11/lingyu11.github.io)存放在`master`分支，用于部署（仅仅是为了个人页面能够访问）
* 如何更新笔记：
    - Step1: 部署`C:\Users\username\lingyu11.github.io> python -m mkdocs gh-deploy --config-file ../ChaoNotes/mkdocs.yml --remote-branch master`
    - Step2: 存档 `C:\Users\username\ChaoNotes> git add ./git commit -m "commit message"/git push`
* 本地笔记重启：`C:\Users\username\ChaoNotes>python -m mkdocs serve`



## Useful links

* [开源世界旅行手册](https://i.linuxtoy.org/docs/guide/)
* [Linux C编程一站式学习](https://akaedu.github.io/book/)
* [awesome-c](https://github.com/oz123/awesome-c)
* [天坑专业转码自救指南](https://shuiyuan.sjtu.edu.cn/t/topic/267562)
