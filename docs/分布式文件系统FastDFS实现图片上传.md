# 分布式文件系统 FastDFS实现图片上传

##### @author 程威

### FastDFS简介

```text
FastDFS是一个开源的轻量级分布式文件系统，它对文件进行管理，功能包括：文件存储、文件同步、文件访问（文件上传、文件下载）等，解决了大容量存储和负载均衡的问题。特别适合以文件为载体的在线服务，如相册网站、视频网站等等。

FastDFS为互联网量身定制，充分考虑了冗余备份、负载均衡、线性扩容等机制，并注重高可用、高性能等指标，使用FastDFS很容易搭建一套高性能的文件服务器集群提供文件上传、下载等服务。

FastDFS服务端有两个角色：跟踪器（tracker）和存储节点（storage）。跟踪器主要做调度工作，在访问上起负载均衡的作用。

存储节点存储文件，完成文件管理的所有功能：就是这样的存储、同步和提供存取接口，FastDFS同时对文件的metadata进行管理。所谓文件的meta data就是文件的相关属性，以键值对（key valuepair）方式表示，如：width=1024，其中的key为width，value为1024。文件metadata是文件属性列表，可以包含多个键值对。

跟踪器和存储节点都可以由一台或多台服务器构成。跟踪器和存储节点中的服务器均可以随时增加或下线而不会影响线上服务。其中跟踪器中的所有服务器都是对等的，可以根据服务器的压力情况随时增加或减少。

为了支持大容量，存储节点（服务器）采用了分卷（或分组）的组织方式。存储系统由一个或多个卷组成，卷与卷之间的文件是相互独立的，所有卷的文件容量累加就是整个存储系统中的文件容量。一个卷可以由一台或多台存储服务器组成，一个卷下的存储服务器中的文件都是相同的，卷中的多台存储服务器起到了冗余备份和负载均衡的作用。

在卷中增加服务器时，同步已有的文件由系统自动完成，同步完成后，系统自动将新增服务器切换到线上提供服务。

当存储空间不足或即将耗尽时，可以动态添加卷。只需要增加一台或多台服务器，并将它们配置为一个新的卷，这样就扩大了存储系统的容量。

FastDFS中的文件标识分为两个部分：卷名和文件名，二者缺一不可。
```

### 上传交互过程

- client 询问 tracker 上传到的 storage，不需要附加参数
- tracker 返回一台可用的 storage
- client 直接和 storage 通讯完成文件上传

### 下载交互过程

- client 询问 tracker 下载文件的 storage，参数为文件标识（卷名和文件名）
- tracker 返回一台可用的 storage
- client 直接和 storage 通讯完成文件下载

```
需要说明的是，client 为使用 FastDFS 服务的调用方，client 也应该是一台服务器，它对 tracker 和 storage 的调用均为服务器间的调用。
```



### FastDFS 为什么要结合 Nginx

```text
我们在使用 FastDFS 部署一个分布式文件系统的时候，通过 FastDFS 的客户端 API 来进行文件的上传、下载、删除等操作。同时通过 FastDFS 的 HTTP 服务器来提供 HTTP 服务。但是 FastDFS 的 HTTP 服务较为简单，无法提供负载均衡等高性能的服务，我们使用 FastDFS 的 Nginx 模块来弥补这一缺陷。

FastDFS 通过 Tracker 服务器,将文件放在 Storage 服务器存储,但是同组之间的服务器需要复制文件,有延迟的问题.假设 Tracker 服务器将文件上传到了 192.168.1.80,文件ID已经返回客户端,这时,后台会将这个文件复制到 192.168.1.30,如果复制没有完成,客户端就用这个 ID 在 192.168.1.30 取文件,肯定会出现错误。这个 fastdfs-nginx-module 可以重定向连接到源服务器取文件,避免客户端由于复制延迟的问题,出现错误。
```

### FastDFS 安装

```
我们基于docker来安装FastDFS
```

### 环境准备

```
libfastcommon.tar.gz
fastdfs-5.11.tar.gz
nginx-1.13.6.tar.gz
fastdfs-nginx-module_v1.16.tar.gz
```

### 创建工作目录

```
在 Linux 服务器上创建 /usr/local/docker/fastdfs/environment 目录

说明：

/usr/local/docker/fastdfs：用于存放 docker-compose.yml 配置文件及 FastDFS 的数据卷
/usr/local/docker/fastdfs/environment：用于存放 Dockerfile 镜像配置文件及 FastDFS 所需环境
```

### docker-compose.yml

```yaml
version: '3.1'
services:
  fastdfs:
    build: environment
    restart: always
    container_name: fastdfs
    volumes:
      - ./storage:/fastdfs/storage
    network_mode: host
```

### Dockerfile

```yaml
FROM ubuntu:xenial
MAINTAINER topsale@vip.qq.com

# 更新数据源
WORKDIR /etc/apt
RUN echo 'deb http://mirrors.aliyun.com/ubuntu/ xenial main restricted universe multiverse' > sources.list
RUN echo 'deb http://mirrors.aliyun.com/ubuntu/ xenial-security main restricted universe multiverse' >> sources.list
RUN echo 'deb http://mirrors.aliyun.com/ubuntu/ xenial-updates main restricted universe multiverse' >> sources.list
RUN echo 'deb http://mirrors.aliyun.com/ubuntu/ xenial-backports main restricted universe multiverse' >> sources.list
RUN apt-get update

# 安装依赖
RUN apt-get install make gcc libpcre3-dev zlib1g-dev --assume-yes

# 复制工具包
ADD fastdfs-5.11.tar.gz /usr/local/src
ADD fastdfs-nginx-module_v1.16.tar.gz /usr/local/src
ADD libfastcommon.tar.gz /usr/local/src
ADD nginx-1.13.6.tar.gz /usr/local/src

# 安装 libfastcommon
WORKDIR /usr/local/src/libfastcommon
RUN ./make.sh && ./make.sh install

# 安装 FastDFS
WORKDIR /usr/local/src/fastdfs-5.11
RUN ./make.sh && ./make.sh install

# 配置 FastDFS 跟踪器
ADD tracker.conf /etc/fdfs
RUN mkdir -p /fastdfs/tracker

# 配置 FastDFS 存储
ADD storage.conf /etc/fdfs
RUN mkdir -p /fastdfs/storage

# 配置 FastDFS 客户端
ADD client.conf /etc/fdfs

# 配置 fastdfs-nginx-module
ADD config /usr/local/src/fastdfs-nginx-module/src

# FastDFS 与 Nginx 集成
WORKDIR /usr/local/src/nginx-1.13.6
RUN ./configure --add-module=/usr/local/src/fastdfs-nginx-module/src
RUN make && make install
ADD mod_fastdfs.conf /etc/fdfs

WORKDIR /usr/local/src/fastdfs-5.11/conf
RUN cp http.conf mime.types /etc/fdfs/

# 配置 Nginx
ADD nginx.conf /usr/local/nginx/conf

COPY entrypoint.sh /usr/local/bin/
ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]

WORKDIR /
EXPOSE 8888
CMD ["/bin/bash"]
```

### entrypoint.sh

```shell
#!/bin/sh
/etc/init.d/fdfs_trackerd start
/etc/init.d/fdfs_storaged start
/usr/local/nginx/sbin/nginx -g 'daemon off;'
```

注：Shell 创建后是无法直接使用的，需要赋予执行的权限，使用 `chmod +x entrypoint.sh` 命令

### 各种配置文件说明

#### tracker.conf

FastDFS 跟踪器配置，容器中路径为：/etc/fdfs，修改为：

```conf
base_path=/fastdfs/tracker
```

#### storage.conf

FastDFS 存储配置，容器中路径为：/etc/fdfs，修改为：

```conf
base_path=/fastdfs/storage
store_path0=/fastdfs/storage
tracker_server=192.168.75.128:22122
http.server_port=8888
```

#### client.conf

FastDFS 客户端配置，容器中路径为：/etc/fdfs，修改为：

```conf
base_path=/fastdfs/tracker
tracker_server=192.168.75.128:22122
```

#### config

fastdfs-nginx-module 配置文件，容器中路径为：/usr/local/src/fastdfs-nginx-module/src，修改为：

```conf
# 修改前
CORE_INCS="$CORE_INCS /usr/local/include/fastdfs /usr/local/include/fastcommon/"
CORE_LIBS="$CORE_LIBS -L/usr/local/lib -lfastcommon -lfdfsclient"

# 修改后
CORE_INCS="$CORE_INCS /usr/include/fastdfs /usr/include/fastcommon/"
CORE_LIBS="$CORE_LIBS -L/usr/lib -lfastcommon -lfdfsclient"
```

#### mod_fastdfs.conf

fastdfs-nginx-module 配置文件，容器中路径为：/usr/local/src/fastdfs-nginx-module/src，修改为：

```
connect_timeout=10
tracker_server=192.168.75.128:22122
url_have_group_name = true
store_path0=/fastdfs/storage
```

#### nginx.conf

Nginx 配置文件，容器中路径为：/usr/local/src/nginx-1.13.6/conf，修改为：

```conf
user  root;
worker_processes  1;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile        on;

    keepalive_timeout  65;

    server {
        listen       8888;
        server_name  localhost;

        location ~/group([0-9])/M00 {
            ngx_fastdfs_module;
        }

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
}
```

### 启动容器

```sh
docker-compose up -d
```

### 测试上传

#### 交互式进入容器

```sh
docker exec -it fastdfs /bin/bash
```

#### 测试文件上传

```sh
/usr/bin/fdfs_upload_file /etc/fdfs/client.conf /usr/local/src/fastdfs-5.11/INSTALL
```

#### 服务器反馈上传地址

```sh
group1/M00/00/00/wKhLi1oHVMCAT2vrAAAeSwu9TgM3976771
```

### 测试 Nginx 访问

```sh
http://192.168.75.128:8888/group1/M00/00/00/wKhLi1oHVMCAT2vrAAAeSwu9TgM3976771
```

### Java操作FastDFS

##### Maven导入jar包

```maven
<dependency>
	  <groupId>com.github.tobato</groupId>
	  <artifactId>fastdfs-client</artifactId>
</dependency>
```

##### 图片上传工具类

```java
package com.syj.aihome.goods.common.util;

import com.github.tobato.fastdfs.domain.conn.FdfsWebServer;
import com.github.tobato.fastdfs.domain.fdfs.StorePath;
import com.github.tobato.fastdfs.domain.proto.storage.DownloadByteArray;
import com.github.tobato.fastdfs.service.FastFileStorageClient;
import org.apache.commons.io.FilenameUtils;
import org.apache.commons.lang3.StringUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import org.springframework.web.multipart.MultipartFile;

import java.io.File;
import java.io.FileInputStream;
import java.io.IOException;

@Component
public class FastDfsUtil {

    @Autowired
    private FastFileStorageClient fastFileStorageClient;
    @Autowired
    private FdfsWebServer fdfsWebServer;

    /**
     * 文件上传-spring mvc
     *
     * @param file
     * @return
     * @throws IOException
     */
    public String uploadFile(MultipartFile file) throws IOException {
        StorePath storePath = fastFileStorageClient.uploadFile(file.getInputStream(), file.getSize(), FilenameUtils.getExtension(file.getOriginalFilename()), null);
        String fullPath = storePath.getFullPath();
        getResAccessUrl(fullPath);
        return fullPath;

    }

    /**
     * 文件上传-本地文件
     *
     * @param file
     * @return
     */
    public String uploadFile(File file) {
        try {
            FileInputStream inputStream = new FileInputStream(file);
            StorePath storePath = fastFileStorageClient.uploadFile(inputStream, file.length(), FilenameUtils.getExtension(file.getName()), null);
            return storePath.getFullPath();
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }
    }

    /**
     * 文件下载
     *
     * @param filePath
     * @return
     */
    public byte[] downloadFile(String filePath) {
        StorePath storePath = StorePath.parseFromUrl(filePath);
        byte[] bytes = fastFileStorageClient.downloadFile(storePath.getGroup(), storePath.getPath(), new DownloadByteArray());
        return bytes;
    }

    /**
     * 删除文件
     *
     * @param filePath
     * @return
     */
    public Boolean deleteFile(String filePath) {
        if (StringUtils.isEmpty(filePath)) {
            return false;
        }
        try {
            StorePath storePath = StorePath.parseFromUrl(filePath);
            fastFileStorageClient.deleteFile(storePath.getGroup(), storePath.getPath());
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
        return true;
    }

    // 封装文件完整URL地址
    public String getResAccessUrl(String path) {
        String url = fdfsWebServer.getWebServerUrl() + path;
        System.out.println("上传文件地址为：\n" + url);
        return url;
    }
}

```

##### 上传图片Controller

```java
package com.syj.aihome.goods.goodsmgt.controller;


import cn.hutool.core.util.StrUtil;
import com.shangyi.aihome.common.result.CommonResult;
import com.syj.aihome.goods.common.enums.response.GoodsImageResposneCodeEnum;
import com.syj.aihome.goods.common.util.FastDfsUtil;
import com.syj.aihome.goods.goodsmgt.entity.po.GoodImage;
import com.syj.aihome.goods.goodsmgt.service.GoodImageService;
import io.swagger.annotations.Api;
import io.swagger.annotations.ApiImplicitParam;
import io.swagger.annotations.ApiImplicitParams;
import io.swagger.annotations.ApiOperation;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;

import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.multipart.MultipartFile;

import java.io.IOException;
import java.time.LocalDateTime;

/**
 * @since 2020-04-27
 */
@RestController
@RequestMapping("/Image")
@Slf4j
@Api(value = "图片上传controller", tags = {"图片上传API"})
public class GoodImageController {
    @Autowired
    private FastDfsUtil fastDfsUtil;
    @Autowired
    private GoodImageService goodImageService;


    @ApiOperation(value = "图片上传", notes = "图片上传")
    @RequestMapping(value = "/upload", method = RequestMethod.POST)
    @ApiImplicitParams({
            @ApiImplicitParam(name = "sourceType", value = "图片类型", required = true)
    })
    public CommonResult uploadFile(MultipartFile file, Integer sourceType) throws IOException {

        //1.校验
        if (file == null) {
            return new CommonResult(GoodsImageResposneCodeEnum.UPLOAD_FAILED.getValue(), GoodsImageResposneCodeEnum.UPLOAD_FAILED.getDesc(), "文件为空");
        }
        String originalFilename = file.getOriginalFilename();
        long fileSize = file.getSize();
        //2.文件上传
        log.info("开始上传文件 ....");
        String s = fastDfsUtil.uploadFile(file);
        String resAccessUrl = fastDfsUtil.getResAccessUrl(s);
        log.info("上传文件结束，文件存储地址：" + resAccessUrl);
        //3.保存元数据信息
        GoodImage imageInfo = null;
        if (StrUtil.isNotEmpty(resAccessUrl)) {
            imageInfo = new GoodImage();
            String suffix = originalFilename.substring(originalFilename.lastIndexOf("."));
            imageInfo.setFileName(originalFilename);
            imageInfo.setFilePath(resAccessUrl);
            imageInfo.setFileType(suffix);
            imageInfo.setFileSize(fileSize);
            imageInfo.setSourceType(sourceType);
            imageInfo.setCreateTime(LocalDateTime.now());
            imageInfo.setCreateUser("SYSTEM");
            goodImageService.saveGoodsServiceImageInfo(imageInfo);
            log.info("保存图片元数据：" + imageInfo.toString());
        } else {
            log.info("上传文件失败 ....");
            return new CommonResult(GoodsImageResposneCodeEnum.UPLOAD_FAILED.getValue(), GoodsImageResposneCodeEnum.UPLOAD_FAILED.getDesc(), null);
        }
        return new CommonResult(GoodsImageResposneCodeEnum.UPLOAD_SUCCESS.getValue(), GoodsImageResposneCodeEnum.UPLOAD_SUCCESS.getDesc(), imageInfo);
    }


    @ApiOperation(value = "图片删除", notes = "图片删除")
    @PostMapping(value = "/remove")
    @ApiImplicitParams({
            @ApiImplicitParam(name = "filePath", value = "存储路径", required = true),
            @ApiImplicitParam(name = "fileID", value = "ID", required = true)
    })
    public CommonResult removeFile(String filePath, Long fileID) throws IOException {
        //1.校验
        if (StrUtil.isEmpty(filePath) || fileID == null) {
            return new CommonResult(GoodsImageResposneCodeEnum.UPLOAD_FAILED.getValue(), GoodsImageResposneCodeEnum.UPLOAD_FAILED.getDesc(), "参数为空");
        }
        //2.删除文件
        log.info("开始删除文件 ....");
        boolean isOk = fastDfsUtil.deleteFile(filePath);
        log.info("删除文件成功");
        //3.保存元数据信息
        log.info("开始删除图片元数据 ....");
        if (isOk) {
            boolean isDelFileOk = goodImageService.removeById(fileID);
            if (isDelFileOk) {
                log.info("删除图片元数据成功");
            } else {
                log.info("删除图片元数据失败");
            }
        } else {
            return new CommonResult(GoodsImageResposneCodeEnum.UPLOAD_FAILED.getValue(), GoodsImageResposneCodeEnum.UPLOAD_FAILED.getDesc(), "删除图片成功");
        }
        return new CommonResult(GoodsImageResposneCodeEnum.UPLOAD_SUCCESS.getValue(), GoodsImageResposneCodeEnum.UPLOAD_SUCCESS.getDesc(), null);
    }
}


```

### 总结

```text
使用Java操作FastDFS实现图片上传
1.导入jar
2.编写工具类操作FastDFS
3.编写业务逻辑代码，上传成功后，会返回一个图片地址和id
4.将图片id存储到数据库对应的字段，前端进行回显
```

##### 

