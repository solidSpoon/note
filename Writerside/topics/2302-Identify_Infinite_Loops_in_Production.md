# 查看生产死循环

查看进程 id

```bash
jps -mlv
```

获取堆栈信息

```bash
jstack 23757 > loop.txt 
```

看看 pid 为 23757 的进程中线程的具体情况

```bash
top -p 23757 -H 
```

![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/202207271357646.png)

可以看到PID 为 23772, 23773 和 23774 的线程占用 CPU 较高

将 10 进制的 23772 转为 16 进制，因为 jstack 中 PID 用的是 16 进制

```bash
printf "%x" 23772
5cdc 
```

打开 loop.txt 文件，搜 5cdc

![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/202207271404646.jpeg)

