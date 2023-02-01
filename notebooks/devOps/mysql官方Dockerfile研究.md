# Mysql官方Dockerfile

[toc]

只看官文就好了：https://hub.docker.com/_/mysql

Mysql 的dockerfilehttps://github.com/docker-library/mysql/blob/7a850980c4b0d5fb5553986d280ebfb43230a6bb/8.0/Dockerfile

最佳文档：https://www.jianshu.com/p/12fc253fa37d



## /docker-entrypoint-initdb.d

如果存在mysql/data目录，就不会执行初始化

有初始化过，就不会执行。只在空目录下初始化新的MYSQL才执行