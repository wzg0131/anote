# openssl生成自签证书

```shell
# genra	生成RSA私钥
# -des3	des3算法
# -out server.key 生成的私钥文件名
# 2048 私钥长度

openssl genrsa -des3 -out server.pass.key 2048
#去除私钥中的密码
#有密码的私钥是server.pass.key，没有密码的私钥是server.key
openssl rsa -in server.pass.key -out server.key
# req 生成证书签名请求
# -new 新生成
# -key 私钥文件
# -out 生成的CSR文件
# -subj 生成CSR证书的参数
openssl req -new -key server.key -out server.csr -subj "/C=CN/ST=Guangdong/L=ShenZhen/O=colorlight/OU=rd/CN=192.168.1.11"
# -days 证书有效期
openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt
https://blog.csdn.net/nklinsirui/article/details/89432430



https://github.com/cookcodeblog/OneDayDevOps/blob/master/components/ssl/create_self_signed_cert.sh
#!/usr/bin/env bash

set-e

#Locate shell script path
SCRIPT_DIR=$(dirname $0)
if[ ${SCRIPT_DIR}!='.']
then
  cd${SCRIPT_DIR}
fi

#Generate RSA private key
openssl genrsa -des3 -passout pass:x -out server.pass.key 2048

#Remove password in the private key
openssl rsa -passin pass:x -in server.pass.key -out server.key
rm -f server.pass.key

#Generate CSR sign request
SUBJ="$1"
openssl req -new -key server.key -out server.csr -subj "$SUBJ"

#Generate CRT signed cert
openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt

```

 



 

来自 <https://github.com/cookcodeblog/OneDayDevOps/blob/master/components/ssl/create_self_signed_cert.sh> 