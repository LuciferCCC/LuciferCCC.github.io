## 一、概论

shell是我们通过命令行与操作系统沟通的语言。

shell脚本可以直接在命令行中执行，也可以将一套逻辑组织成一个文件，方便复用。
AC Terminal中的命令行可以看成是一个“**shell脚本在逐行执行**”。

Linux中常见的shell脚本有很多种，常见的有：

- Bourne Shell(/usr/bin/sh或/bin/sh)
- Bourne Again Shell(/bin/bash)
- C Shell(/usr/bin/csh)
- K Shell(/usr/bin/ksh)

- zsh
- …

Linux系统中一般默认使用 `bash`，所以接下来讲解 `bash` 中的语法。
文件开头需要写 `#! /bin/bash`，指明`bash`为脚本解释器。