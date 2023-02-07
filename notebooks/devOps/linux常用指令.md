# Linux日常指令

[toc]

## 进程

`netstat`查看占用端口的进程：

```shell
root@k8s-node-3:~# netstat -ntulp | grep kubelet
tcp        0      0 127.0.0.1:10248         0.0.0.0:*               LISTEN      6235/kubelet
tcp6       0      0 :::10250                :::*                    LISTEN      6235/kubelet
root@k8s-node-3:~# netstat -ntulp | grep 3306
tcp        0      0 0.0.0.0:3306            0.0.0.0:*               LISTEN      2254/docker-proxy
tcp6       0      0 :::3306                 :::*                    LISTEN      2261/docker-proxy
```



## SSH

ssh绑定远程端口：

```shell
#将本地3310端口绑定到远程3306端口
$ssh -i remote.pem -L 3310:127.0.0.1:3306 ubuntu@192.168.1.12
```



## 挂载

目录到目录的挂载：

```shell
mount --bind /mnt/a /home/wzg/a
```



## 加密解密

### 利用`/dev/urandom`生成随机密码

```shell
colorlight@k8s-node-2:~$ cat  /dev/urandom | head -1 | sha256sum | awk '{print $1}'
80f8853616d68b485d81af10a227a96e223e24f5befd03be918e9b96f72084ae
colorlight@k8s-node-2:~$ cat  /dev/urandom | head -1 | sha256sum | awk '{print $1}'
465492df9ad823fed5a52361a4b745e4fe84d5cd44e229a69a2ae294f8d87e4f
```

### `base64`编码

```shell
root@k8s-node-2:/tmp# echo "hello" > hello.txt
root@k8s-node-2:/tmp# base64 hello.txt
aGVsbG8K
root@k8s-node-2:/tmp# cat hello.txt | base64
aGVsbG8K
root@k8s-node-2:/tmp# echo "hello" | base64
aGVsbG8K
root@k8s-node-2:/tmp# echo -n "hello" | base64
aGVsbG8=

```

注意：echo "hello"包含换行符, echo -n "hello"不包含换行符。

通过解码(`base64 -d`)查看：

```shell
root@k8s-node-2:/tmp# echo aGVsbG8K | base64 -d
hello
root@k8s-node-2:/tmp# echo aGVsbG8= | base64 -d
helloroot@k8s-node-2:/tmp#

```

### openssl生成随机数

```shell
helloroot@k8s-node-2:/tmp# openssl rand -base64 16
zarQlFtRNmbevGXr6t0Q1g==

```

