# OS X 小技巧

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


