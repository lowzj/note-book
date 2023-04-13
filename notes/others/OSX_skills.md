# OS X 小技巧

### VSCode VIM: enable key-repeating

> https://marketplace.visualstudio.com/items?itemName=vscodevim.vim

```sh
defaults write com.microsoft.VSCode ApplePressAndHoldEnabled -bool false              # For VS Code
defaults write com.microsoft.VSCodeInsiders ApplePressAndHoldEnabled -bool false      # For VS Code Insider
defaults write com.visualstudio.code.oss ApplePressAndHoldEnabled -bool false         # For VS Codium
defaults write com.microsoft.VSCodeExploration ApplePressAndHoldEnabled -bool false   # For VS Codium Exploration users
defaults delete -g ApplePressAndHoldEnabled                                           # If necessary, reset global default
```

### 开启远程登陆，使用ssh互联互通。



* [官方指导](https://support.apple.com/kb/PH18726?locale=zh_CN&viewlocale=zh_CN)
* `系统偏好设置` -> `共享` -> `远程登录`

### 使用 emoji 表情

* `Command` + `Ctrl` + `Space`
* [How to use emoji on your Mac](http://www.imore.com/how-to-use-emoji-on-your-mac)

### 终端使用代理

```sh
# 如果使用ss
export ALL_PROXY=socks5://127.0.0.1:1080
# 清除代理
unset ALL_PROXY
```

### ItelliJ IDEA debug java服务时总是卡顿

> 解决方法: https://youtrack.jetbrains.com/issue/IDEA-157303

在`/etc/hosts`中设置: 

```
127.0.0.1 localhost <hostname>
```

其中`hostname`可以通过命令行获取


### 在OSX下使用sed替换字符串

```bash
os=`uname -s`
if [ "${os}" == "Darwin" ]; then
    # -i 后添加临时文件名称, 如果原地替换, 则为空, 不可省略
    sed -i "" 's/<your_regx>/<replace_string>/' <file>
else
    sed -i 's/<your_regx>/<replace_string>/' <file>
fi
```

**solution2**
```bash
# linux;   mac
sedi=(-i); test "$(uname -s)" = "Darwin" && sedi=(-i "")
sed "${sedi[@]}" 's/<your_regx>/<replace_string>/' <file>
```


### 关闭chrome的开发者工具下的请求时间线(overview)

[参考此文](https://blog.csdn.net/qq_15941409/article/details/103232414)

`Network` -> 右上角`x`下面的配置按钮 -> 勾掉`Show overview`

### 查看某一端口的占用情况

```bash
# lsof -i:<port>
lsof -i:1080
```

### 去除iTerm终端粘贴URL自动转译
实际这是`oh-my-zsh`的功能，在`~/.zshrc`最前面加上:
```bash
DISABLE_MAGIC_FUNCTIONS="true"
```
具体代码可参看: `~/.oh-my-zsh/lib/misc.zsh`

### ls: Operation not permitted

* 没有磁盘访问权限: https://osxdaily.com/2018/10/09/fix-operation-not-permitted-terminal-error-macos/
* 设置了文件保护: https://www.hjxfire.cn/2018/05/08/Mac-lsattr-chattr/

### 网页长截屏

Chrome 开发者模式 -> `Shift+Command+P` -> 搜索 `screenshot`，有四种截屏模式

* `Capture area sceenshot`
* `Capture full size sceenshot`
* `Capture node sceenshot`
* `Capture sceenshot`

### 查看ip端口关联的进程id

```bash
# 查看port是否被占用
netstat -an | grep ${port}
# 查看port关联的进程
lsof -i :${port}
```
