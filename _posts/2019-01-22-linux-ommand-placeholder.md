---
categories: linux
layout: post
---

linux中允许你使用命令占位符，占位符中的内容可以是其它命令的输出。

命令占位符有两种写法，一种是`$(command)`，另一种是`` `command` ``，两种写法是等价的。

下面举几个例子：

```sh
echo `pwd` # 显示当前目录
rm $(ls) #删除当前目录下的所有文件
```
