# dnsmasq



## Docker运行

```shell
docker run \
--name dnsmasq \
-d \
-p 53:53/udp \
-p 5380:8080 \
-v /opt/dnsmasq/dnsmasq.conf:/etc/dnsmasq.conf \
--log-opt "max-size=100m" \
-e "HTTP_USER=admin" \
-e "HTTP_PASS=Colorlight1" \
--restart always \
jpillora/dnsmasq
```

## 配置文件

```
#dnsmasq config, for a complete example, see:
#  http://oss.segetech.com/intra/srv/dnsmasq.conf
#log all dns queries
log-queries
#dont use hosts nameservers
no-resolv
#use cloudflare as default nameservers, prefer 1^4
server=1.0.0.1
server=1.1.1.1
strict-order
#serve all .company queries using a specific nameserver
server=/company/10.0.0.1
#explicitly define host-ip mappings
address=/myhost.company/10.0.0.2
```

## 解决systemd-resolve占用53端口问题

1. 修改`/etc/ststemd/resolved.conf`,修改DNS
2. `systemctl restart systemd-resolved.service`重启服务
3. 查看结果`systemd-resolve --status`看是否生效