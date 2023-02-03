# Docker运维

[toc]

## docker占用空间查看

使用`du -sh`查看Docker的目录：

```shell
root@k8s-node-3:~# du -sh /var/lib/docker/
129G    /var/lib/docker/
```

使用`docker system`相关命令：

```shell
root@k8s-node-3:~# docker system df
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          199       31        56.07GB   47.03GB (83%)
Containers      62        22        6.006GB   1.128GB (18%)
Local Volumes   478       50        40.02GB   17.3GB (43%)
Build Cache     0         0         0B        0B
```

清理命令：

```shell
docker system prune	
docker system prune -a 	#-a清理更彻底，stop的容器，没运行的镜像都清除
```



## docker容器资源占用

```shell
root@k8s-node-3:~# docker stats
CONTAINER ID   NAME                      CPU %     MEM USAGE / LIMIT     MEM %     NET I/O           BLOCK I/O         PIDS
b63f499c261d   one-ui                    0.00%     26.18MiB / 31.32GiB   0.08%     566kB / 567kB     4.68MB / 0B       25
38446864aed5   one-video                 0.02%     539MiB / 31.32GiB     1.68%     7.18MB / 10.9MB   261MB / 2.26MB    66
4808837e8332   one-nginx                 0.00%     38.67MiB / 31.32GiB   0.12%     580MB / 1.39GB    972MB / 0B        25
eb71623a118d   one-app                   0.09%     1.293GiB / 31.32GiB   4.13%     5.15GB / 2.44GB   130MB / 8.58MB    156
cff65ce6e855   ccloud-monitor-provider   0.07%     394.2MiB / 31.32GiB   1.23%     13MB / 18.5MB     16.1MB / 3.4MB    173
82d54437ca9b   one-ws                    0.02%     668.9MiB / 31.32GiB   2.09%     13.4MB / 4.06MB   144MB / 0B        87

```

