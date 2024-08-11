# Linux如何查看一个端口是否被占用？

## 1. `lsof` 指令

```shell
# 查看指定端口（8080）占用情况
lsof -i:8080

# 返回值示例
COMMAND PID  USER   FD   TYPE   DEVICE     SIZE/OFF NODE NAME
nginx  26993 root   4u   IPv4   37999514      0t0    TCP *:http-alt (LISTEN)
```

## 2. `netstat` 指令

```shell
# 显示tcp、upd的端口和进程等相关情况
netstat -tunlp | grep 8080

## -t (tcp) 仅显示tcp相关选项
## -u (udp)仅显示udp相关选项
## -n 拒绝显示别名，能显示数字的全部转化为数字
## -l 仅列出在Listen(监听)的服务状态
## -p 显示建立相关链接的程序名
tcp        0      0 0.0.0.0:8000            0.0.0.0:*               LISTEN      26993/wroker proce
```

```cpp
# 其他netstat相关指令
netstat -ntlp   //查看当前所有tcp端口
```

