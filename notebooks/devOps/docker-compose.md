# docker-compose

[toc]

## example

```yaml
version: '3.8'
services:
#  nacos:
#    ports:
#      - 8848:8848
#    image: nacos/nacos-server:v2.1.2
#    container_name: nacos
#    environment:
#      MODE: standalone
#    networks:
#      cloud-a10:
#        ipv4_address: 172.109.0.2
#    restart: always
  cloud-gateway:
#    depends_on:
#      - nacos
    ports:
      - 8080:8080
    image: colorlightwzg/svc-gateway:v0.1.1-beta
    container_name: cloud-gateway
    networks:
      cloud-a10:
        ipv4_address: 172.109.0.3
    restart: always
    volumes:
      - gateway_log:/logs
    entrypoint:
      - java
      - -Xmx512m
      - -Xms512m
      - org.springframework.boot.loader.JarLauncher
    command:
      - --server.port=8080
      - --spring.profiles.active=prod
  svc-user:
    #    depends_on:
    #      - nacos
#    ports:
#      - 8080:8080
    image: colorlightwzg/svc-user:v0.0.1-beta
    container_name: svc-user
    networks:
      cloud-a10:
        ipv4_address: 172.109.0.4
    restart: always
#    volumes:
#      - gateway_log:/logs
    entrypoint:
      - java
      - -Xmx512m
      - -Xms512m
      - org.springframework.boot.loader.JarLauncher
    command:
      - --server.port=8080
      - --spring.profiles.active=prod
      - --spring.cloud.nacos.config.server-addr=192.168.1.11:8849
      - --spring.cloud.nacos.config.namespace=product
      - --spring.cloud.nacos.config.namespace=product
      - --spring.cloud.nacos.discovery.group=a10
  svc-auth:
    image: colorlightwzg/svc-auth:v0.0.1-beta
    container_name: svc-auth
    networks:
      cloud-a10:
        ipv4_address: 172.109.0.5
    restart: always
    entrypoint:
      - java
      - -Xmx512m
      - -Xms512m
      - org.springframework.boot.loader.JarLauncher
    command:
      - --server.port=8080
      - --spring.profiles.active=prod
      - --spring.cloud.nacos.config.server-addr=192.168.1.11:8849
      - --spring.cloud.nacos.config.namespace=product
      - --spring.cloud.nacos.discovery.server-addr=192.168.1.11:8849
      - --spring.cloud.nacos.discovery.namespace=product
      - --spring.cloud.nacos.discovery.group=a10
  svc-terminal-account:
    image: colorlightwzg/svc-terminal-account:v0.0.1-beta
    container_name: svc-terminal-account
    networks:
      cloud-a10:
        ipv4_address: 172.109.0.6
    restart: always
    entrypoint:
      - java
      - -Xmx512m
      - -Xms512m
      - org.springframework.boot.loader.JarLauncher
    command:
      - --server.port=8080
      - --spring.cloud.nacos.config.server-addr=192.168.1.11:8849
      - --spring.cloud.nacos.config.namespace=product
      - --spring.cloud.nacos.discovery.server-addr=192.168.1.11:8849
      - --spring.cloud.nacos.discovery.namespace=product
      - --spring.cloud.nacos.discovery.group=a10
  svc-terminal-control:
    image: colorlightwzg/svc-terminal-control:v0.0.1-beta
    container_name: svc-terminal-control
    networks:
      cloud-a10:
        ipv4_address: 172.109.0.7
    restart: always
    entrypoint:
      - java
      - -Xmx512m
      - -Xms512m
      - org.springframework.boot.loader.JarLauncher
    command:
      - --server.port=8080
      - --spring.cloud.nacos.config.server-addr=192.168.1.11:8849
      - --spring.cloud.nacos.config.namespace=product
      - --spring.cloud.nacos.discovery.server-addr=192.168.1.11:8849
      - --spring.cloud.nacos.discovery.namespace=product
      - --spring.cloud.nacos.discovery.group=a10
  resource-server:
    image: nginx:latest
    container_name: resource-nginx
    networks:
      cloud-a10:
        ipv4_address: 172.109.0.21
    ports:
      - 18080:80
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - media_upload:/usr/local/nginx/resource
    restart: always
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "10"
networks:
  cloud-a10:
    driver: bridge
    ipam:
      driver: default
      config:
        - subnet: 172.109.0.0/16
          gateway: 172.109.0.1
volumes:
  gateway_log:
  media_upload:
```

