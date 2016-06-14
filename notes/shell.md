# shell

### 求两个文件的差集

> **comm** 命令
> ```
> Compare sorted files FILE1 and FILE2 line by line.

> With no options, produce three-column output. 
> Column one contains lines unique to FILE1, column two contains lines unique to FILE2, and column three contains lines common to both files.
> The following options are available:
>    -1      Suppress printing of column 1.
>    -2      Suppress printing of column 2.
>    -3      Suppress printing of column 3.
> ```

比如文件`a`内容如下
```sh
a
a
b
b
b
c
```

现在想要取文件`a`中重复的行

```
zjmac ~ » comm -13 <(uniq -d a) <(uniq -D a)
a
b
b
```
