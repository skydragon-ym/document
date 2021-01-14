# CSAPP

## 第一篇：环境搭建

### BSD链接库-Libbsd

由于作者使用的是BSD系统，如果在GNU/Linux系统上，如CentOS上编译示例程序的话，我们需要安装libbsd库

> **libbsd - Library providing BSD-compatible functions for portability**

具体的rpm的url我们可以通过下面的网站查询

https://pkgs.org/search/?q=Libbsd

https://pkgs.org/search/?q=Libbsd-devel

![image-20201103155330566](C:\Users\v0cn155\AppData\Roaming\Typora\typora-user-images\image-20201103155330566.png)

我是用的系统是CentOS8，所以找到对应的libbsd的下载路径如下：

```
wget https://download-ib01.fedoraproject.org/pub/epel/8/Everything/x86_64/Packages/l/libbsd-0.9.1-4.el8.x86_64.rpm

wget https://download-ib01.fedoraproject.org/pub/epel/8/Everything/x86_64/Packages/l/libbsd-devel-0.9.1-4.el8.x86_64.rpm

rpm -ivh libbsd-0.9.1-4.el8.x86_64.rpm

rpm -ivh libbsd-devel-0.9.1-4.el8.x86_64.rpm
```

### 编译-GCC

在编译任何示例代码之前，我们注意到大部分的示例代码都包含头文件 *\#include "apue.h"*，具体功能实现在libapue.a静态链接库中，所以我们要先生成这个库文件。

```
cd /root/workspace/apue/apue.3e
make
```

现在可以s使用gcc进行编译：

```
gcc -g /root/workspace/apue/apue.3e/intro/ls1.c -o /root/workspace/apue/apue.3e/intro/ls1 -lapue
```

这里要注意一下参数 *-lapue*, 这个参数是告诉gcc在链接时使用libapue.a静态链接库文件。这里我们可以看到，我们提供给编译器的库文件的名字需要去掉前面的lib和文件后缀.a，这是编译器的默认规则。

那么gcc怎么知道去哪里查找这个库文件呢？这时就需要使用 -L这个参数来告诉gcc查找库文件的路径。比如：-L /usr/local/mylib。gcc会优先到-L所指定的目录查找库文件，如果没找到，则按照以下顺序查找 /usr/lib --> /usr/local/lib。

### VS Code



