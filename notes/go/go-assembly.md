# Go-Assembly

今天发现一个好玩的网站，可以在线将 GoLang 代码编译成汇编，而且可以每条语句都可以一一对应。

例如: https://godbolt.org/z/fib1x1

比较字符串判空的两种写法效率是否一样，可以直接看编译后的汇编，结果显示汇编是一样的，所以这两种写法对于 GoLang 编译器来说其实是一样的。

```golang
func isEmpty1(s string) bool {
    return s == ""                      // pcdata  $2, $0
                                        // pcdata  $0, $1
                                        // movq    "".s+16(SP), AX
                                        // testq   AX, AX
                                        // seteq   "".~r1+24(SP)
                                        // ret
}
func isEmpty2(s string) bool {
    return len(s) == 0                  // pcdata  $2, $0
                                        // pcdata  $0, $1
                                        // movq    "".s+16(SP), AX
                                        // testq   AX, AX
                                        // seteq   "".~r1+24(SP)
                                        // ret
}
```


