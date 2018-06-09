# 小命令

### unix和dos文件格式转换

* dos -> unix: 使用`vim -e $fileNmae`并输入以下命令
  ```
  set fileformat=unix
  wq
  ```
* unix -> dos: 使用`vim -e $fileNmae`并输入以下命令
  ```
  set fileformat=dos
  wq
  ```

### git 远程仓库强制覆盖本地修改

```sh
# 清除本地 modified files
git checkout .
# 清除本地 untracked files
git clean -df
# 强行重置 HEAD
git fetch origin
git reset --hard origin/master
```

### GitHub 自动认证
HTTPS方式，使用BASIC认证，修改git配置`remote.origin.url`，在url中加入认证信息即可。例子:
```
# 原URL
https://github.com/lowzj/github-auto-ops
# 加入BASIC认证信息的URL：
https://{username}:{password}@github.com/lowzj/github-auto-ops
```
其中`password`可以用`access_token`来代替，以避免泄漏密码。创建`access_token`见[这里](https://github.com/settings/tokens)。

这里有个例子: https://github.com/lowzj/github-auto-ops

### Linux下root删除文件提示: Operation not permitted
* 现象：使用root账户删除文件提示权限不够；linux下受保护文件即使用root也不能修改。
* 解决：http://blog.csdn.net/gxdvip/article/details/50808157
* 命令

```sh
# 检查文件是否受保护，带 i 标志即为受保护
$ lsattr a.txt
---i---------- a.txt
# 使用 chattr 解除保护
$ chattr -i a.txt
# 使用 chattr 增加保护
$ chattr +i a.txt
```


### shell 求两个文件的差集

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

### 将win格式文件转为unix文件

使用vim打开文件, 执行命令`: set ff=unix`

### Yum操作

```
yum clean all
yum makecache

createrepo --update ./
```

### makefile:4: xxx missing separator. Stop

参见: https://stackoverflow.com/questions/16931770/makefile4-missing-separator-stop
