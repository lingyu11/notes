# C++ primer 5th edition

## Ch1 开始

1. `iostream` - 标准输入输出流。

    | 基础类型 | 对象 |
    | ---- | ---- |
    | istream | cin - 标准输入 |
    | ostream | cout - 标准输出，cerr - 标准错误，clog |

    | cout << "hello" << endl; | cin >> v1 >> v2; |
    | ---- | ---- |
    | <<是输出运算符，其左侧运算对象：ostream对象，右侧运算对象：要打印的内容，返回左侧运算对象。右侧运算对象的值写到左侧运算对象表示的输出流 | >>是输入运算符，其左侧运算对象：istream对象，右侧：要输入的内容，返回左侧运算对象。从左侧运算对象所指定的输入流读取数据，存入右侧运算对象中 |

    ```C++
    while (std::cin >> value)
    ```
    此表达式从标准输入读取下一个数，保存在value中。输入运算符（参见1.2节，第7页）返回其左侧运算对象，在本例中是std::cin。因此，此循环条件实际上检测的是std::cin。

    当我们使用一个istream对象作为条件时，其效果是检测流的状态。如果流是有效的，即流未遇到错误，那么检测成功。当遇到文件结束符（end-of-file），或遇到一个无效输入时（例如读入的值不是一个整数），istream对象的状态会变为无效。处于无效状态的istream对象会使条件变为假。

2. 注释界定符不能嵌套：`/*.../*..*/...*/` 是错误的
3. 头文件：使类或其他名字的定义可被多个程序使用的一种机制
4. C++：静态数据类型语言，它的类型检查发生在编译时，因此编译器必须知道程序中每一个变量对应的数据类型
5. C++标准规定的算术类型尺寸的最小值：int最小占16位，但我们用的一般是占32位的int

    | 类型 | 含义 | 最小尺寸 |
    | ---- | ---- | --- |
    | bool | 布尔类型 | 未定义 |
    | char | 字符 | 8 位 |
    | wchar_t | 宽字符 | 16 位 |
    | char16_t | Unicode 字符 | 16 位 |
    | char32_t | Unicode 字符 | 32 位 |
    | short | 短整型 | 16 位 |
    | int | 整型 | 16 位 |
    | long | 长整型 | 32 位 |
    | long long | 长整型 | 64 位 |
    | float | 单精度浮点数 | 6 位有效数字 |
    | double | 双精度浮点数 | 6 位有效数字 |
    | long double | 扩展精度浮点数 | 10位有效数字 |

6. 字节：可寻址的最小内存块。32位机器上一个字为4个字节：

    | 地址 | 每个字节具体内容 |
    | ---- | ---- |
    | 736424 | 00111011 |
    | 736425 | 00011011 |
    | 736426 | 01110001 |
    | 736427 | 01100100 |

    !!! 字长

        字长（也称为机器字长或处理器字长）是指计算机处理器一次性能够处理的数据位数。现代 32 位处理器意味着它们可以处理 32 位的数据，即 4 个字节。

7. 方法



<!-- <table>  
    <tr>    
        <td colspan="3" style="text-align:center;"><b>C++算数类型</b></td>  
    </tr>
    <tr>    
        <td colspan="1" style="text-align:left;"><b>类型</b></td>
        <td colspan="1" style="text-align:left;"><b>含义</b></td>
        <td colspan="1" style="text-align:left;"><b>最小尺寸</b></td>
    </tr>
    <tr>    
        <td style="width:33%; text-align:left;">第二行，第一列</td>   
        <td style="width:33%; text-align:left;">第二行，第二列</td>   
        <td style="width:33%; text-align:left;">第二行，第三列</td>  
    </tr> 
    <tr>   
        <td style="width:33%; text-align:left;">第三行，第一列</td>    
        <td style="width:33%; text-align:left;">第三行，第二列</td>    
        <td style="width:33%; text-align:left;">第三行，第三列</td>  
    </tr>  
    <tr>   
        <td style="width:33%; text-align:left;">第四行，第一列</td>   
        <td style="width:33%; text-align:left;">第四行，第二列</td>   
        <td style="width:33%; text-align:left;">第四行，第三列</td> 
    </tr> 
    <tr>    
        <td style="width:33%; text-align:left;">第五行，第一列</td>    
        <td style="width:33%; text-align:left;">第五行，第二列</td>    
        <td style="width:33%; text-align:left;">第五行，第三列</td>  
    </tr>
</table> -->

## Ch2 变量和基础类型




