# 问题总结

## link: combining dwarf failed: Unknown load command 0x32

```bash
/usr/local/Cellar/go/1.10.3/libexec/pkg/tool/darwin_amd64/link: /usr/local/Cellar/go/1.10.3/libexec/pkg/tool/darwin_amd64/link: combining dwarf failed: Unknown load command 0x32 (50)
```

macOS 升级 golang 版本到 `1.10.4` 以上

```bash
# 升级到最新版本
brew upgrade go
# 升级到指定版本
brew upgrade go -v=1.10.4
```

