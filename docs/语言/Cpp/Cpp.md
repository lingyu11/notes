# C++ primer 5th edition

## Ch1 开始

1. `iostream` - 标准输入输出流。

    | 基础类型 | 对象 |
    | ---- | ---- |
    | istream | cin - 标准输入 |
    | ostream | cout - 标准输出，cerr - 标准错误，clog |

<!-- ![alt text](/img/image.png) 文本渲染 -->


## Ch2 变量和基础类型

    mkdocs.yml    # The configuration file.
    docs/
        index.md  # The documentation homepage.
        ...       # Other markdown pages, images and other files.

```py
from tempfile import TemporaryFile

with TemporaryFile('w+t') as f:
    # Read/write to the file
    f.write('Hello World\n')
    f.write('Testing\n')

    # Seek back to beginning and read the data
    f.seek(0)
    data = f.read()

# Temporary file is destroyed
```

!!! note

    Lorem ipsum dolor sit amet, consectetur adipiscing elit. Nulla et euismod
    nulla. Curabitur feugiat, tortor non consequat finibus, justo purus auctor
    massa, nec semper lorem quam in massa.


