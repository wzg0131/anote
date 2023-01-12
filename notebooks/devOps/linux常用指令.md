

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

