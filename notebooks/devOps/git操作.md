# git操作

[toc]

## 配置代理

clone的时候配置这个仓库的代理：

```shell
git clone -c http.proxy="http://120.55.88.213:8893" 
```

全局代理：

```shell
git config --global http.proxy "http://120.55.88.213:8893"
```



## 现有项目迁移远程仓库

已有的项目，更改远程仓库

```shell
#删除远程仓库
git remote remove origin
#添加新的远程仓库地址
git remote add origin https://gitgub.com/wzg0131/clt-ws.git
#推送主分支
git branch -M main
git push -u origin main
```

