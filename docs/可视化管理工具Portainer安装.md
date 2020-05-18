## Portainer简介

```
Portainer 是一款轻量级的应用，它提供了图形化界面，用于方便的管理Docker环境，包括单机环境和集群环境，下面我们将用Portainer来管理Docker容器中的应用
```

### 获取Docker镜像文件

```
docker pull portainer/portainer
```

### 使用docker容器运行Portainer

```
docker run -p 9000:9000 -p 8000:8000 --name portainer \
--restart=always \
-v /var/run/docker.sock:/var/run/docker.sock \
-v /mydata/portainer/data:/data \
-d portainer/portainer
```

### 查看Portainer的DashBoard信息

![](http://www.wink.run:9090/wink/q.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=minioadmin%2F20200518%2F%2Fs3%2Faws4_request&X-Amz-Date=20200518T033140Z&X-Amz-Expires=432000&X-Amz-SignedHeaders=host&X-Amz-Signature=34bec265002caa35bda5840f4f1ffbe23ea4f909a184376c88f11e0b316c5311)

### 查看所有运行中的容器信息

![](http://www.wink.run:9090/wink/w.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=minioadmin%2F20200518%2F%2Fs3%2Faws4_request&X-Amz-Date=20200518T033309Z&X-Amz-Expires=432000&X-Amz-SignedHeaders=host&X-Amz-Signature=8cc5263ca6c3a262d3f0a64b8710f0692a5020a8cb5ff79e5038c56f5191e887)

### 查看所有已经下载的Docker镜像

![](http://www.wink.run:9090/wink/e.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=minioadmin%2F20200518%2F%2Fs3%2Faws4_request&X-Amz-Date=20200518T033918Z&X-Amz-Expires=432000&X-Amz-SignedHeaders=host&X-Amz-Signature=45ba8dbd585369c03da9b96e7b2807cc679a25d659237cfcd427f3ab3e0f1f06)

### 查看`mall-portal`应用的统计信息

![](http://www.wink.run:9090/wink/r.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=minioadmin%2F20200518%2F%2Fs3%2Faws4_request&X-Amz-Date=20200518T033554Z&X-Amz-Expires=432000&X-Amz-SignedHeaders=host&X-Amz-Signature=c0d5d4e4a4d3a820efac7d4d5f5cf2f5aadda18d42d9c7439e99f7f2227b968f)

### 查看`mall-portal`应用的运行过程中打印的日志信息

![](http://www.wink.run:9090/wink/t.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=minioadmin%2F20200518%2F%2Fs3%2Faws4_request&X-Amz-Date=20200518T033625Z&X-Amz-Expires=432000&X-Amz-SignedHeaders=host&X-Amz-Signature=df5b1107e899963481962a891733c7527880a6e57f705850e8b78c53956a0373)

### 进入`mall-portal`应用的容器内部来操作容器内部系统

![](http://www.wink.run:9090/wink/y.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=minioadmin%2F20200518%2F%2Fs3%2Faws4_request&X-Amz-Date=20200518T033651Z&X-Amz-Expires=432000&X-Amz-SignedHeaders=host&X-Amz-Signature=8ba9cf8e8f76a27fb2114b0274517cd8fb1e30fab3d7a0731bdf362f4d0fe79d)