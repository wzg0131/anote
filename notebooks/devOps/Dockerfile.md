# Dockerfile构建镜像

## Dockerfile最佳实践

### 分层构建

通过`docker image history [--no-trunc] image ` 查看镜像创建每个层的命令：

```shell
root@k8s-node-2:~# docker image history colorlightwzg/one-app:develop-221011-1408
IMAGE          CREATED        CREATED BY                                      SIZE      COMMENT
bc71a07aad0a   2 months ago   /bin/sh -c #(nop)  ENTRYPOINT ["java" "org.s…   0B
e30b930839cd   2 months ago   /bin/sh -c #(nop)  VOLUME [/tmp /data /logs]    0B
abd2568cd1e8   2 months ago   /bin/sh -c #(nop) COPY --chown=colorlight:co…   6.3MB
8b80d6a6acb4   2 months ago   /bin/sh -c #(nop) COPY --chown=colorlight:co…   0B
66d3c5bcb491   2 months ago   /bin/sh -c #(nop) COPY --chown=colorlight:co…   254kB
bc30caca37d8   2 months ago   /bin/sh -c #(nop) COPY --chown=colorlight:co…   105MB
0f6566af139d   2 months ago   /bin/sh -c apk update && apk add ttf-dejavu …   21.9MB
25eda991e26d   4 months ago   /bin/sh -c #(nop)  ENV LOG_DIR=/logs            0B
51bd29db5e94   4 months ago   /bin/sh -c #(nop)  ENV TMP_DIR=/tmp             0B
2f7199ebbf55   4 months ago   /bin/sh -c #(nop)  ENV DATA_DIR=/data           0B
cfbe8cd39ba9   4 months ago   /bin/sh -c #(nop)  ENV JAVA_OPTS=               0B
b185576e23ee   4 months ago   /bin/sh -c #(nop) WORKDIR /application          0B
6dfd56514fd8   4 months ago   /bin/sh -c #(nop)  ENV PATH=/usr/local/sbin:…   0B
<missing>      4 months ago   /bin/sh -c #(nop)  ENV JAVA_HOME=/usr/lib/jv…   0B
<missing>      4 months ago   /bin/sh -c #(nop)  ENV LANG=C.UTF-8             0B
<missing>      4 months ago   |1 version=11.0.16.8.1 /bin/sh -c wget -O /T…   320MB
<missing>      4 months ago   /bin/sh -c #(nop)  ARG version=11.0.16.8.1      0B
<missing>      4 months ago   /bin/sh -c #(nop)  CMD ["/bin/sh"]              0B
<missing>      4 months ago   /bin/sh -c #(nop) ADD file:f77e3f51f020890d2…   5.59MB

```

利用层的**构建缓存机制**，对于将不常改变的操作放在前面，可以提高构建效率。

### 多阶段构建

将构建时依赖与运行时依赖分开。

maven打包和tomcat运行war包的例子：
```dockerfile
# syntax=docker/dockerfile:1
FROM maven AS build
WORKDIR /app
COPY . .
RUN mvn package

FROM tomcat
COPY --from=build /app/target/file.war /usr/local/tomcat/webapps #将上一个阶段构建的war包拷贝到当前阶段
```

### 其他文章

https://blog.51cto.com/lizhenliang/2363565

## Dockerfile参考

例子：[Mysql的Dockerfile](https://github.com/docker-library/mysql/blob/223f0be1213bbd8647b841243a3114e8b34022f4/5.7/Dockerfile)

### springboot2.4+应用

```dockerfile
FROM eclipse-temurin:11-jre as builder
WORKDIR application
ARG JAR_FILE=./*.jar
COPY ${JAR_FILE} application.jar
RUN java -Djarmode=layertools -jar application.jar extract

FROM eclipse-temurin:11-jre
WORKDIR application
ENV JAVA_OPTS ""
ENV DATA_DIR "/data"
ENV TMP_DIR "/tmp"
ENV LOG_DIR "/logs"
RUN groupadd -g 3991 colorlight && useradd colorlight -m -u 3991 -g colorlight -s /sbin/nologin && \
    mkdir -p $DATA_DIR && \
    mkdir -p $LOG_DIR && \
    mkdir -p $TMP_DIR && \
    chown colorlight:colorlight $DATA_DIR $LOG_DIR $TMP_DIR
USER colorlight

COPY --from=builder --chown=colorlight:colorlight application/dependencies/ ./
COPY --from=builder --chown=colorlight:colorlight application/spring-boot-loader/ ./
COPY --from=builder --chown=colorlight:colorlight application/snapshot-dependencies/ ./
COPY --from=builder --chown=colorlight:colorlight application/application/ ./

VOLUME ["${TMP_DIR}","${DATA_DIR}","${LOG_DIR}"]
ENTRYPOINT ["java", "org.springframework.boot.loader.JarLauncher", "$0", "$@"]

```

### 普通jar应用

```dockerfile
FROM openjdk:8
MAINTAINER "jerry.wong@lednets.com"

ARG PROJECT_NAME=one-0.1.5-SNAPSHOT.jar
ENV PARAMS=""
ENV UPLOAD_DIR=/usr/share/nginx/html/backup/wp-content/uploads
ENV CONFIG_DIR=/var/lib/app
ENV LOG_DIR=/logs
ENV TEMP_DIR=/tmp

RUN groupadd -g 3991 colorlight && useradd colorlight -m -u 3991 -g colorlight -s /sbin/nologin
COPY --chown=colorlight:colorlight ${PROJECT_NAME} app.jar
RUN mkdir -p $UPLOAD_DIR && \
    mkdir -p $CONFIG_DIR && \
    mkdir -p $LOG_DIR && \
    mkdir -p $TEMP_DIR && \
    chown colorlight:colorlight $UPLOAD_DIR $CONFIG_DIR $LOG_DIR $TEMP_DIR

EXPOSE 8080

#VOLUME ["$LOG_DIR","$UPLOAD_DIR","TEMP_DIR","$CONFIG_DIR"]

ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom ${PARAMS}","-Dspring.config.location=${CONFIG_DIR}/application.yml","-jar","/app.jar"]

```



### nginx自定义模块基础镜像

```dockerfile
FROM debian:buster
# note: we use jessie instead of wheezy because our deps are easier to get here

# runtime dependencies
# (packages are listed alphabetically to ease maintenence)
RUN apt-get update && apt-get install -y --no-install-recommends build-essential \
        && rm -rf /var/lib/apt/lists/*

ENV NGINX_VERSION 1.21.0

# All our runtime and build dependencies, in alphabetical order (to ease maintenance)
RUN buildDeps=" \
                ca-certificates \
                curl \
                gcc \
                libc-dev-bin \
                libc6-dev \
                libexpat1-dev \
                libfontconfig1-dev \
                libfreetype6-dev \
                libgd-dev \
                #libgd2-dev \
                libgeoip-dev \
                libice-dev \
                libjbig-dev \
                #libjpeg8-dev \
        libjpeg62-turbo-dev \
                liblzma-dev \
                libpcre3-dev \
                libperl-dev \
                #libpng12-dev \
                libpthread-stubs0-dev \
                libsm-dev \
                libssl-dev \
                libssl-dev \
                libtiff5-dev \
                libvpx-dev \
                libx11-dev \
                libxau-dev \
                libxcb1-dev \
                libxdmcp-dev \
                libxml2-dev \
                libxpm-dev \
                libxslt1-dev \
                libxt-dev \
                linux-libc-dev \
                make \
                manpages-dev \
                x11proto-core-dev \
                x11proto-input-dev \
                x11proto-kb-dev \
                xtrans-dev \
                zlib1g-dev \
        "; \
        apt-get update && apt-get install -y --no-install-recommends $buildDeps && rm -rf /var/lib/apt/lists/*

RUN curl -SL "http://nginx.org/download/nginx-$NGINX_VERSION.tar.gz" -o nginx.tar.gz \
        && curl -SL "http://nginx.org/download/nginx-$NGINX_VERSION.tar.gz.asc" -o nginx.tar.gz.asc \
        && mkdir -p /usr/src/nginx \
        && tar -xvf nginx.tar.gz -C /usr/src/nginx --strip-components=1 \
        && rm nginx.tar.gz* \
        && cd /usr/src/nginx \
        && ./configure \
                --user=www-data \
                --group=www-data \
                --prefix=/usr/sbin \
                --sbin-path=/usr/sbin/nginx \
                --conf-path=/etc/nginx/nginx.conf \
                --http-log-path=/var/log/nginx/access.log \
                --error-log-path=/var/log/nginx/error.log \
                --with-http_addition_module \
                --with-http_auth_request_module \
                --with-http_geoip_module \
                --with-http_image_filter_module \
                --with-http_perl_module \
                --with-http_realip_module \
                --with-http_ssl_module \
                --with-http_stub_status_module \
                --with-http_sub_module \
                --with-http_xslt_module \
                --with-ipv6 \
                --with-mail \
                --with-mail_ssl_module \
                --with-pcre-jit \
        && make -j"$(nproc)" \
        && make install \
        && cd / \
        && rm -r /usr/src/nginx \
        && mkdir /usr/local/nginx \
        && chown -R www-data:www-data /usr/local/nginx \
        && chown -R www-data:www-data /var/log/nginx \
        && { \
                echo; \
                echo '# stay in the foreground so Docker has a process to track'; \
                echo 'daemon off;'; \
        } >> /etc/nginx.conf \
        && apt-get remove --purge --auto-remove -y $buildDeps \
        && apt-get remove --purge --auto-remove -y gcc

ENV PATH /usr/sbin:$PATH
WORKDIR /usr/local/nginx/html

# TODO USER www-data

EXPOSE 80
CMD ["nginx"]
```

## Entrypoint.sh启动脚本

例子：[Mysql的**docker-entrypoint.sh**](https://github.com/docker-library/mysql/blob/223f0be1213bbd8647b841243a3114e8b34022f4/8.0/docker-entrypoint.sh)
