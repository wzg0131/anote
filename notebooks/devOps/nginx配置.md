# nginx配置参考

[toc]

## nginx.conf

```nginx
user www-data;
worker_processes auto;
pid /tmp/nginx.pid;
worker_rlimit_nofile 100000;
#https://hub.docker.com/_/nginx/
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
        #keepalive_timeout 300;
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
```



## default.conf

```nginx
#http
server {
    listen 8000;
    server_name AAAA;
    #rewrite ^(.*)$ https://${server_name}$1 permanent;
    #华为红线nginx扫描4.1.1
    return 301 https://$host$request_uri;
}
#https
server {
    #HW 2.6
    listen 8443;
    listen 4443;
    server_name AAAA;

    ssl on;
    ssl_certificate /etc/letsencrypt/live/AAAA/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/AAAA/privkey.pem;
    ssl_prefer_server_ciphers on;
    #ssl_session_timeout 10m;
    #HW 7.2
    ssl_session_timeout 5m;
    ssl_session_cache shared:SSL:1m;

    #ssl_protocols SSLv3 TLSv1 TLSv1.1 TLSv1.2;
    #华为红线nginx扫描4.1.4，但是盒子需要1.1
    ssl_protocols TLSv1.2;

    #ssl_ciphers "HIGH:!aNULL:!MD5 or HIGH:!aNULL:!MD5:!3DES";
    ssl_ciphers "ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES256-SHA:AES256-GCM-SHA384:AES128-GCM-SHA256:AES256-SHA256:AES256-SHA:HIGH:!MEDIUM:!LOW:!aNULL:!eNULL:!EXPORT:!DES:!MD5:!PSK:!RC4:@STRENGTH";


    error_page 497 301 https://$http_host$request_uri;
    error_page 500 501 400 401 402 403 404 405 406 407 413 414 /50x.html;

    resolver 114.114.114.114;
    charset utf8;
    tcp_nodelay on;

    if ($request_method !~ ^(GET|HEAD|POST|OPTION|PUT|DELETE|PATCH)$) {
            return 403;
    }

    server_tokens off;
    limit_conn limitperip 10;

    gzip on;
    gzip_comp_level 6;
    gzip_types text/css application/javascript application/x-javascript application/rss+xml application/json;
    gzip_disable    "MSIE [1-6]\.";

    client_max_body_size 2048M;
    client_body_buffer_size 8k;

    location ~ .*.(svn|Git|cvs) {
          deny all;
    }
    location /50x.html {
        root /usr/share/nginx/html;
    }
    location / {
          #HW 2.7
          #deny  172.31.1.1;
          #allow 172.31.1.0/16;
          index  index.html index.htm index.php;
          root /var/www/html;
          try_files $uri $uri/ /index.html;
          proxy_set_header Upgrade $http_upgrade;
          proxy_set_header Connection "upgrade";
    }
    #http接口
    location ~ /wp-json {
          add_header 'Access-Control-Allow-Credentials' 'true';
          add_header 'Access-Control-Allow-Headers' 'Origin, X-Requested-With, Content-Type, Accept';
          add_header 'Access-Control-Allow-Methods' 'POST, GET, PUT, OPTIONS, DELETE, PATCH';
          add_header 'Access-Control-Allow-Origin' *;
          add_header 'Access-Control-Expose-Headers' 'Location';
          add_header Content-Type 'application/json; charset=utf-8';
          add_header Content-Security-Policy "upgrade-insecure-requests;connect-src *";
          add_header X-Frame-Options SAMEORIGIN;
          proxy_cookie_path / "/; httponly";
          proxy_read_timeout 600;
          proxy_set_header host $host:$server_port;
          proxy_set_header X-Real-IP      $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_pass http://one-app:8080;
    }
    #登录
    location /wp-login.php {
          add_header 'Access-Control-Allow-Credentials' 'true';
          add_header 'Access-Control-Allow-Headers' 'Origin, X-Requested-With, Content-Type, Accept';
          add_header 'Access-Control-Allow-Methods' 'POST, GET, PUT, OPTIONS, DELETE';
          add_header 'Access-Control-Allow-Origin' *;
          add_header 'Access-Control-Expose-Headers' 'Location';
          add_header Content-Type 'application/json; charset=utf-8';
          add_header Content-Security-Policy "upgrade-insecure-requests;connect-src *";
          add_header X-Frame-Options SAMEORIGIN;
          proxy_cookie_path / "/; httponly";
          proxy_read_timeout 600;
          proxy_set_header host $host:$server_port;
          proxy_set_header X-Real-IP      $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_pass http://one-app:8080;
    }
    #静态资源（截图）
    location  /wp-content/uploads/ {
          alias /usr/share/nginx/html/backup/wp-content/uploads/;
    }
    #静态资源（媒体）
    location /wp-content/ {
          alias /usr/share/nginx/html/backup/wp-content/uploads/;
    }
    #Yahoo 天气转发
    #location ^~ /news {
    #      proxy_pass https://www.yahoo.com;
    #      #     proxy_set_header host $host:$server_port;
    #      #     proxy_set_header X-Real-IP      $remote_addr;
    #      #     proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    #       expires 0;
    #}
    #天气服务器转发
    location ~ /api/weather {
          add_header 'Access-Control-Allow-Credentials' 'true';
          add_header 'Access-Control-Allow-Headers' 'Origin, X-Requested-With, Content-Type, Accept';
          add_header 'Access-Control-Allow-Methods' 'POST, GET, PUT, OPTIONS, DELETE';
          add_header 'Access-Control-Allow-Origin' *;
          add_header 'Access-Control-Expose-Headers' 'Location';
          add_header Content-Type 'application/json; charset=utf-8';
          add_header Content-Security-Policy "upgrade-insecure-requests;connect-src *";
          add_header X-Frame-Options SAMEORIGIN;
          proxy_cookie_path / "/; httponly";
          proxy_read_timeout 600;
          proxy_set_header host $host:$server_port;
          proxy_set_header X-Real-IP      $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_pass http://backup.lednets.com:3000;
    }
    #获取截图走app
    location /wp-content/uploads/screenshot/ {
          add_header 'Access-Control-Allow-Credentials' 'true';
          add_header 'Access-Control-Allow-Headers' 'Origin, X-Requested-With, Content-Type, Accept';
          add_header 'Access-Control-Allow-Methods' 'POST, GET, PUT, OPTIONS, DELETE';
          add_header 'Access-Control-Allow-Origin' *;
          add_header 'Access-Control-Expose-Headers' 'Location';
          add_header Content-Type 'application/json; charset=utf-8';
          add_header Content-Security-Policy "upgrade-insecure-requests;connect-src *";
          add_header X-Frame-Options SAMEORIGIN;
          proxy_cookie_path / "/; httponly";
          proxy_read_timeout 600;
          proxy_set_header host $host:$server_port;
          proxy_set_header X-Real-IP      $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_pass http://one-app:8080;
    }
    #洲明
    location /webuser/domain/ {
         add_header 'Access-Control-Allow-Credentials' 'true';
         add_header 'Access-Control-Allow-Headers' 'Origin, X-Requested-With, Content-Type, Accept';
         add_header 'Access-Control-Allow-Methods' 'POST, GET, PUT, OPTIONS, DELETE';
         add_header 'Access-Control-Allow-Origin' *;
         add_header 'Access-Control-Expose-Headers' 'Location';
         add_header Content-Type 'application/json; charset=utf-8';
         add_header Content-Security-Policy "upgrade-insecure-requests;connect-src *";
         add_header X-Frame-Options SAMEORIGIN;
         proxy_read_timeout 600;
         proxy_set_header host $host:$server_port;
         proxy_set_header X-Real-IP      $remote_addr;
         proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
         proxy_pass http://one-app:8080;
         allow 172.31.2.133; deny all;
    }
    #前端websocket
    location /ColorWebSocket  {
         proxy_redirect off;
         proxy_pass http://one-ws:8443/ColorWebSocket;
         proxy_http_version 1.1;
         proxy_set_header Upgrade $http_upgrade;
         proxy_set_header Connection "Upgrade";
         proxy_set_header X-real-ip $remote_addr;
         proxy_set_header X-Forwarded-For $remote_addr;
    }
    #盒子websocket
    location /ColorWebSocket/websocket/chat  {
         proxy_redirect off;
         proxy_pass http://one-ws:8443/ColorWebSocket/websocket/chat;
         proxy_http_version 1.1;
         proxy_set_header Upgrade $http_upgrade;
         proxy_set_header Connection "Upgrade";
         proxy_set_header X-real-ip $remote_addr;
         proxy_set_header X-Forwarded-For $remote_addr;
    }
}

```

