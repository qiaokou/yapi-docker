# yapi docker部署

​		YApi 是**高效**、**易用**、**功能强大**的 api 管理平台，旨在为开发、产品、测试人员提供更优雅的接口管理服务。可以帮助开发者轻松创建、发布、维护 API，YApi 还为用户提供了优秀的交互体验，开发人员只需利用平台提供的接口数据写入工具以及简单的点击操作就可以实现接口的管理。详细介绍参见官网

## 制作镜像

注意：yapi已停止维护许久，1.12.0 tag是最新镜像，镜像制作过程中需要联网

### Dockerfile

```bash
FROM node:12-alpine as builder
WORKDIR /yapi
RUN apk add --no-cache git python3 make openssl tar gcc
ENV VERSION=1.12.0
RUN wget https://github.com/YMFE/yapi/archive/refs/tags/v${VERSION}.zip
RUN unzip v${VERSION}.zip && mv yapi-${VERSION} vendors
RUN cd /yapi/vendors && rm package-lock.json && cp config_example.json ../config.json && npm install --production --registry https://registry.npmmirror.com

FROM node:12-alpine
MAINTAINER wangksb@dcits.com
ENV TZ="Asia/Shanghai"
WORKDIR /yapi/vendors
COPY --from=builder /yapi/vendors /yapi/vendors
EXPOSE 3000
ENTRYPOINT ["node"]
```

### 示例命令

```bash
cd /data
mkdir yapi
touch Dockerfile
# 复制Dockerfile信息，保存
docker build -t yapi:1.12.0 .
```

## 内网无网环境部署

### 导出镜像

MongoDB镜像需要提前进行pull，版本为4.2.21

```bash
docker save -o yapi-v1.10.2.tar yapi:1.12.0
docker save -o mongo-v4.2.21.tar mongo:4.2.21
```

导出成功后上传镜像至服务器

### 加载镜像

```bash
docker load -i yapi-v1.10.2.tar
docker load -i mongo-v4.2.21.tar
```

### 安装MongoDB

#### 创建基础目录

```bash
mkdir -p /root/yapi/mongo/data
mkdir -p /root/yapi/mongo/conf
mkdir -p /root/yapi/mongo/log    
```

 /root/yapi/目录可自行更改合适的目录

#### 创建MongoDB配置文件

```bash
cd /root/yapi/mongo/conf
vim mongodb.conf
# 复制下面的conf文件信息，保存
```

conf文件信息

```bash
#端口
port=27017
#数据库文件存放目录
dbpath=/data/db
#日志文件存放路径
logpath=/data/log
#使用追加方式写日志
logappend=true
#以守护线程的方式运行，创建服务器进程
fork=true
#最大同时连接数
maxConns=100
#不启用验证
#noauth=true
#每次写入会记录一条操作日志
journal=true
#存储引擎有mmapv1、wiredTiger、mongorocks
storageEngine=wiredTiger
#访问IP
bind_ip=0.0.0.0
#用户验证
#auth=true
```

#### 安装配置

#### docker安装

密码注意修改，这里为123456，

```bash
docker run -d \
--name mongodb  \
-p 17017:27017 \
-v /root/yapi/mongo/data:/data/db \
-v /root/yapi/mongo/conf:/data/conf \
-v /root/yapi/mongo/log:/data/log \
-e MONGO_INITDB_ROOT_USERNAME=admin \
-e MONGO_INITDB_ROOT_PASSWORD=123456 \
--privileged=true \
--restart always \
mongo:4.2.21
```

#### 测试是否成功

```bash
docker exec -it mongodb mongo admin
db.auth("admin","123456")
# 没有报错即可
# 进入MongoDB容器，如需维护时可使用此命令进入容器
docker exec -it mongodb /bin/bash
```

可使用navicat连接MongoDB，或使用其他方法连接创建数据库yapi

### 安装yapi

#### 配置config.json文件

```bash
cd /root/yapi
vim config.json
# 参考下面的config示例编辑，保存
```

config示例，下面配置根据实际情况进行修改，如adminAccount、db相关

```json
{
  "port": "3000",
  "adminAccount": "admin@qq.com",
  "timeout":120000,
  "closeRegister": false,
  "db": {
    "servername": "x.x.x.x",
    "DATABASE": "yapi",
    "port": 17017,
    "user": "admin",
    "pass": "123456",
    "authSource": "admin"
  }
}
```

#### 初始化数据

需要初始化基本信息，包括管理员账号登

```bash
docker run -it --rm \
  --link mongodb:mongodb \
  --entrypoint npm \
  --workdir /yapi/vendors \
  -v /root/yapi/config.json:/yapi/config.json \
  yapi:1.12.0 \
  run install-server
```

  进行数据初始化，打印信息如下，为安装成功
  初始化管理员账号成功,账号名："admin@qq.com"，密码："ymfe.org"

#### 启动yapi

```bash
docker run -d \
  --name yapi \
  --link mongodb:mongodb \
  --workdir /yapi/vendors \
  -p 13001:3000 \
  -v /root/yapi/config.json:/yapi/config.json \
  yapi:1.12.0 \
  server/app.js
```

启动成功后，输入http://ip:13001 访问系统，使用账号名："admin@qq.com"，密码："ymfe.org"登录，注意修改初始化密码

如何使用就不在此赘述了。

注意：及时关闭注册功能，修改config.json中closeRegister为ture，重启docker容器即可。

# 参考网站

官网：https://github.com/YMFE/yapi/

源码最新tag下载地址：https://github.com/YMFE/yapi/archive/refs/tags/v1.12.0.zip

MongoDB安装参考：

​	https://blog.csdn.net/fen_fen/article/details/126359997

​	https://www.jb51.net/server/330288siv.htm

yapi安装参考：

​	https://www.jianshu.com/p/a97d2efb23c5

​	https://www.cnblogs.com/dream-ze/p/17596481.html

​	https://blog.csdn.net/xiaoyukongyi/article/details/123599230

​	https://www.cnblogs.com/dream-ze/p/17596481.html

