# 查看 Linux 服务器日志的技巧


## 倒叙查看最近的异常

```Bash
tac logs/example.log | grep -B 100 -A 100 -i exception | more
```