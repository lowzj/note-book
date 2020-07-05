# Sync.Pool



使用下列命令在执行 race detect 时, `sync.Pool.Put` 方法会1/4概率丢失一个数据，所以不能保证 `Put` 后的数据一定能 `Get` 出来，要小心。
```bash
go test -race
```



