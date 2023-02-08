# nginx流媒体服务器

[toc]

基于Nginx http-flv 模块



## 构建

### Dockerfile

```dockerfile
##Nginx with http-flv module
FROM debian:buster
MAINTAINER jerrywong@lednets.com

ENV NGINX_VERSION 1.21.6

# 编译需要的工具
RUN buildDeps=" \
                build-essential \
                ca-certificates \
                curl \
                git \
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
        && mkdir -p /usr/src/nginx \
        && tar -xvf nginx.tar.gz -C /usr/src/nginx --strip-components=1 \
        && rm nginx.tar.gz* \
        && cd /usr/src/nginx \
        && curl -SL "https://github.com/winshining/nginx-http-flv-module/archive/refs/tags/v1.2.10.tar.gz" -o nginx-http-flv-module.tar.gz \
        && tar -xzvf nginx-http-flv-module.tar.gz -C /usr/src/ \
        && mv /usr/src/nginx-http-flv-module-1.2.10 /usr/src/nginx-http-flv-module \
        && rm nginx-http-flv-module.tar.gz \
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
                --with-http_dav_module \
                --with-http_gzip_static_module \
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
                --add-dynamic-module=/usr/src/nginx-http-flv-module \
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


EXPOSE 80
EXPOSE 1935
CMD ["nginx"]

```

## 配置

```nginx
worker_processes auto;
pid /tmp/nginx.pid;
worker_rlimit_nofile 100000;
events {
        worker_connections 1024;
        #multi_accept on;
        use epoll;
}

http {

        ##
        # Basic Settings
        ##
        #Nginx will not add the port in the url when the request is redirected
	    port_in_redirect off;
	    proxy_hide_header X-Powered-By;
        proxy_hide_header Server;

        sendfile on;
        tcp_nopush on;
        tcp_nodelay on;

        proxy_connect_timeout 300;
        proxy_read_timeout 300;
        proxy_send_timeout 300;

        types_hash_max_size 2048;

        client_header_buffer_size 2k;
        large_client_header_buffers 4 8k;
        client_body_buffer_size 200M;
        client_max_body_size 2048M;


        server_tokens off;
        #华为红线5.2.1
        client_body_timeout 10;
        client_header_timeout 10;
        #红线要求最多10s
        keepalive_timeout 10;
        send_timeout 10;

        #华为红线nginx扫描4.1.6,HW 7.5
        ssl_dhparam /etc/nginx/ssl/dhparam.pem;
        ssl_ecdh_curve brainpoolP512r1;

        # server_names_hash_bucket_size 64;
        # server_name_in_redirect off;

        include /etc/nginx/mime.types;
        default_type application/octet-stream;

        client_body_temp_path /tmp/client_temp;
        proxy_temp_path       /tmp/proxy_temp_path;
        fastcgi_temp_path     /tmp/fastcgi_temp;
        uwsgi_temp_path       /tmp/uwsgi_temp;
        scgi_temp_path        /tmp/scgi_temp;
        ##
        # Logging Settings
        ##

        log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                                        '"$status" $body_bytes_sent "$http_referer" '
                                        '"$http_user_agent" "$http_x_forwarded_for" '
                                        '"$gzip_ratio" $request_time $bytes_sent $request_length';


        access_log /var/log/nginx/access.log main;
        error_log /var/log/nginx/error.log info;
        open_log_file_cache max=1000 inactive=60s;

        ##
        # Gzip Settings
        ##

        gzip on;
        gzip_disable "msie6";

        # gzip_vary on;
        # gzip_proxied any;
        gzip_comp_level 6;
        gzip_buffers 16 8k;
        gzip_http_version 1.1;
        # gzip_types text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript;

        limit_conn_zone $binary_remote_addr zone=limitperip:10m;

        #file cache
        open_file_cache max=100000 inactive=20s;
        open_file_cache_valid 30s;
        open_file_cache_min_uses 2;
        open_file_cache_errors on;


         fastcgi_buffers 256 16k;
         fastcgi_buffer_size 128k;
         fastcgi_connect_timeout 3s;
         fastcgi_send_timeout 600s;
         fastcgi_read_timeout 600s;
         fastcgi_busy_buffers_size 256k;
         fastcgi_temp_file_write_size 256k;

        ##
        # nginx-naxsi config
        ##
        # Uncomment it if you installed nginx-naxsi
        ##

        #include /etc/nginx/naxsi_core.rules;

        ##
        # nginx-passenger config
        ##
        # Uncomment it if you installed nginx-passenger
        ##

        #passenger_root /usr;
        #passenger_ruby /usr/bin/ruby;

        ##
        # Virtual Host Configs
        ##

        include /etc/nginx/conf.d/*.conf;
        include /etc/nginx/sites-enabled/*;
}
rtmp {
    out_queue               4096;
    out_cork                8;
    max_streams             128;
    timeout                 15s;
    drop_idle_publisher     15s;

    include /etc/nginx/conf.d/rtmp/*.conf;
}

```

### http-flv配置

```nginx
server {
    listen                          80;
    root                            /var/www;
    charset                         utf-8;
    access_log                      off;
    error_log                       off;

    location / {
        index                       index.html;
    }

    location /live {
        flv_live                    on;
        chunked_transfer_encoding   on;
        add_header                  'Access-Control-Allow-Origin' '*';
        add_header                  'Access-Control-Allow-Credentials' 'true';
    }

    location /hls {
        types {
            application/vnd.apple.mpegurl m3u8;
            video/mp2t              ts;
        }

        root                        /tmp/hls;
        add_header                  'Cache-Control' 'no-cache';
    }

    location /dash {
        root                        /tmp/dash;
        add_header                  'Cache-Control' 'no-cache';
    }

    location /stat {
        rtmp_stat                   all;
        rtmp_stat_stylesheet        stat.xsl;
    }

    location /stat.xsl {
        root                        /var/www;
    }

    location = /favicon.ico {
        access_log                  off;
        log_not_found               off;
        expires                     30d;
    }

    location = /robots.txt  {
        access_log                  off;
        log_not_found               off;
        expires                     30d;
    }

    location ~* \.(css|js)(\?.*)?$ {
        access_log                  off;
        expires                     24h;
    }

    location ~* \.(ico|gif|jpg|jpeg|png)(\?.*)?$ {
        access_log                  off;
        expires                     30d;
    }

    location ~* \.(eot|ttf|otf|woff|woff2|svg)$ {
        access_log                  off;
        expires                     max;
    }
}

```

### rtmp配置

```nginx
server {
    listen              1935;

    application screen {
        live            on;
        gop_cache       on;
    }

    application hls {
        live            on;
        hls             on;
        hls_path        /tmp/hls;
    }

    application dash {
        live            on;
        dash            on;
        dash_path       /tmp/dash;
    }
}


```

## 运行

```shell
#! /bin/bash

DOCKER_IMG_VERSION=1.2.2

docker volume create rtmp_tmp && \
docker run --restart=always -d \
--log-opt max-size=100M \
-p 11935:1935 \
-p 8088:80 \
-p 7443:443 \
--name rtmp-server \
-v "$(pwd)"/nginx.conf:/etc/nginx/nginx.conf \
-v "$(pwd)"/rtmp.conf:/etc/nginx/conf.d/rtmp/rtmp.conf \
-v "$(pwd)"/http_flv.conf:/etc/nginx/conf.d/http_flv.conf \
-v rtmp_tmp:/tmp \
--network one-nw \
colorlightwzg/streaming-server:${DOCKER_IMG_VERSION} \
nginx -g 'daemon off;'

```

