

## V3 基本命令

查询所有键
```
./etcdctl get / --prefix --keys-only
```

查询所有的值
```
./etcdctl get / --prefix --print-value-only
```