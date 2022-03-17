---
title: MacOS Big Sur 更新后，Parallels Desktop 14 无法正常启动的解决办法
date: 2022-03-17 13:00:56
tags:
- MAC skills
---

Macbook 更新 MacOS Big Sur 版本后，原安装的 Parallels Desktop 14 无法正常启动， 报错如下：

后来在网上找到解决办法，打开终端命令窗口，输入：

```shell
export SYSTEM_VERSION_COMPAT=1
#再输入以下命令，打开应用：
open -a "Parallels Desktop"
```

作者：hal
链接：https://ld246.com/article/1606492419224
来源：链滴
协议：CC BY-SA 4.0 https://creativecommons.org/licenses/by-sa/4.0/



