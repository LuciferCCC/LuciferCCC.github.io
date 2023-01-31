## 文件检索

```shell linenums="1"
find /path/to/directory/ -name '*.py' # 搜索某个文件路径下的所有*.py文件

grep xxx # 从stdin中读入若干行数据，如果某行中包含xxx，则输出该行；否则忽略该行。

wc # 统计行数、单词数、字节数
    # 既可以从stdin中直接读入内容；也可以在命令行参数中传入文件名列表；
    wc -l # 统计行数
    wc -w # 统计单词数
    wc -c # 统计字节数

tree # 展示当前目录的文件结构
tree /path/to/directory/ # 展示某个目录的文件结构
tree -a # 展示隐藏文件

ag xxx # 搜索当前目录下的所有文件，检索xxx字符串

cut # 分割一行内容
# 从stdin中读入多行数据
echo $PATH | cut -d ':' -f 3,5 # 输出PATH用:分割后第3、5列数据
echo $PATH | cut -d ':' -f 3-5 # 输出PATH用:分割后第3-5列数据
echo $PATH | cut -c 3,5 # 输出PATH的第3、5个字符
echo $PATH | cut -c 3-5 # 输出PATH的第3-5个字符

sort # 将每行内容按字典序排序
    # 可以从stdin中读取多行数据
    # 可以从命令行参数中读取文件名列表

xargs # 将stdin中的数据用空格或回车分割成命令行参数
find . -name '*.py' | xargs cat | wc -l # 统计当前目录下所有python文件的总行数
find . -name '*.py' | xargs rm # 找到当前目录下所有python文件的并删除
find 指定目录 -type d -name ".DS_Store" | xargs rm -rf # 删除指定目录下的所有 .DS_Store 文件夹
```