# 使用buildx构建多平台镜像


# 构建多平台镜像的几种方法

## QEMU仿真

> 使用`qemu` 仿真出很多平台，`buildx`集成了该特性，可以实现在其他平台构建不需要修改`dockerfile` ，当它需要对不同的架构运行一个二进制文件时，它会自己从binfmt_misc 处理器中已经注册的架构去加载对应的二进制文件。当然，我们需要手动在`binfmt_misc`处理器里去注册我们想要的架构的。

本篇文章主要介绍使用该方法来实现多平台镜像构建。

## 使用不同节点来构建

这种方式性能较好，但是并未选择这种方式，如果想深入了解，可以参考其他资料。

官方例子：

```Bash
$ docker buildx create --use --name mybuild node-amd64
mybuild
$ docker buildx create --append --name mybuild node-arm64
$ docker buildx build --platform linux/amd64,linux/arm64 .
```

## 交叉编译

如果开发语言对交叉编译支持较好，可以使用`dockerfiles`的多阶段构建，可以构建时传入`BUILDPLATFORM` 和`TARGETPLATFORM` 参数选择运行的平台及编译的平台。

```Bash
FROM --platform=$BUILDPLATFORM golang:alpine AS build
ARG TARGETPLATFORM
ARG BUILDPLATFORM
RUN echo "I am running on $BUILDPLATFORM, building for $TARGETPLATFORM" > /log
FROM alpine
COPY --from=build /log /log
```

# 使用buildx构建

## 开启docker buildx

开启docker buildx插件，要求版本不低于`19.03`

第一种方法临时开启：

```Bash
$ export DOCKER_CLI_EXPERIMENTAL=enabled

```

第二种方法：在`config.json` 中开启`"experimental": "enabled"`

```Bash
$ vim ~/.docker/config.json
{
  ...
  "experimental": "enabled"
}
 
```

验证

```Bash
$ docker buildx version 
github.com/docker/buildx v0.3.1-tp-docker 6db68d029599c6710a32aa7adcba8e5a344795a7


```

## 开启`binfmt_misc` 

> 如果使用的是docker desktop默认是已经启用`binfmt_misc`的了 
内核版本最好大于3.x，我这里用的是5.x的内核。

这里使用`docker/binfmt` 开启特权模式启用`binfmt_misc` 

```Bash
docker run --privileged --rm docker/binfmt:a7996909642ee92942dcd6cff44b9b95f08dad64
```

检查是否开启成功

```Bash
$ ls -l /proc/sys/fs/binfmt_misc/
total 0
-rw-r--r--. 1 root root 0 Jan  7 17:41 jexec
-rw-r--r--. 1 root root 0 Jan  7 17:43 qemu-aarch64
-rw-r--r--. 1 root root 0 Jan  7 17:43 qemu-arm
-rw-r--r--. 1 root root 0 Jan  7 17:43 qemu-ppc64le
-rw-r--r--. 1 root root 0 Jan  7 17:43 qemu-riscv64
-rw-r--r--. 1 root root 0 Jan  7 17:43 qemu-s390x
--w-------. 1 root root 0 Jan  7 17:41 register
-rw-r--r--. 1 root root 0 Jan  7 17:41 status
 
```

检查是否启用处理器

```Bash
$ grep -r "enabled" /proc/sys/fs/binfmt_misc/ 
/proc/sys/fs/binfmt_misc/qemu-riscv64:enabled
/proc/sys/fs/binfmt_misc/qemu-s390x:enabled
/proc/sys/fs/binfmt_misc/qemu-ppc64le:enabled
/proc/sys/fs/binfmt_misc/qemu-arm:enabled
/proc/sys/fs/binfmt_misc/qemu-aarch64:enabled
/proc/sys/fs/binfmt_misc/jexec:enabled
grep: /proc/sys/fs/binfmt_misc/register: Invalid argument
/proc/sys/fs/binfmt_misc/status:enabled
```

## 创建builder

需要创建一个新的docker builder，名称叫`builder`

```Bash
$ docker buildx create --name builder --use
builder
$ docker buildx ls
NAME/NODE      DRIVER/ENDPOINT             STATUS   PLATFORMS
builder    docker-container                     
  builder0 unix:///var/run/docker.sock inactive 
default *      docker                               
  default      default                     running  linux/amd64, linux/386

```

这时可以看到一个默认builder和我们新建的一个builder，目前新建的builder还处于inactive的状态。

通过如下方式启动新的builder。

```Bash
$ docker buildx inspect builder --bootstrap
[+] Building 63.8s (1/1) FINISHED                                                                                                                                                
 => [internal] booting buildkit                                                                                                                                            63.8s
 => => pulling image moby/buildkit:buildx-stable-1                                                                                                                         61.9s
 => => creating container buildx_buildkit_builder0                                                                                                                      1.9s
Name:   builder
Driver: docker-container

Nodes:
Name:      builder0
Endpoint:  unix:///var/run/docker.sock
Status:    running
Platforms: linux/amd64, linux/arm64, linux/riscv64, linux/ppc64le, linux/s390x, linux/386, linux/arm/v7, linux/arm/v6


```

## 构建多平台镜像

当前目录结构：

```Bash
$ tree  
├── Dockerfile
└── test.go

```

Dockerfile:

```Bash
FROM golang:1.15-alpine AS builder
WORKDIR /opt
RUN go build -o test .

FROM alpine
WORKDIR /opt
COPY --from=builder /opt/test .
CMD ["./test"]

```

test.go

```Go
package main

import (
        "fmt"
)

func main() {
  fmt.Println("test")
}

```

开始构建并上传，注意`harbor` 版本要在`2.0` 以上才支持。

```Bash
$ docker buildx build -t reg.xxxxxx.cn/app/test-golang:v1 --platform=linux/arm64,linux/amd64 . 

[+] Building 5.2s (22/22) FINISHED                                                                                                                                               
 => [internal] load build definition from Dockerfile                                                                                                                        0.0s
 => => transferring dockerfile: 92B                                                                                                                                         0.0s
 => [internal] load .dockerignore                                                                                                                                           0.0s
 => => transferring context: 2B                                                                                                                                             0.0s
 => [linux/amd64 internal] load metadata for docker.io/library/alpine:latest                                                                                                2.3s
 => [linux/amd64 internal] load metadata for docker.io/library/golang:1.15-alpine                                                                                           2.0s
 => [linux/arm64 internal] load metadata for docker.io/library/alpine:latest                                                                                                3.1s
 => [linux/arm64 internal] load metadata for docker.io/library/golang:1.15-alpine                                                                                           2.1s
 => [linux/amd64 builder 1/4] FROM docker.io/library/golang:1.15-alpine@sha256:49b4eac11640066bc72c74b70202478b7d431c7d8918e0973d6e4aeb8b3129d2                             0.0s
 => => resolve docker.io/library/golang:1.15-alpine@sha256:49b4eac11640066bc72c74b70202478b7d431c7d8918e0973d6e4aeb8b3129d2                                                 0.0s
 => [linux/arm64 stage-1 1/3] FROM docker.io/library/alpine@sha256:3c7497bf0c7af93428242d6176e8f7905f2201d8fc5861f45be7a346b5f23436                                         0.1s
 => => resolve docker.io/library/alpine@sha256:3c7497bf0c7af93428242d6176e8f7905f2201d8fc5861f45be7a346b5f23436                                                             1.8s
 => [linux/arm64 builder 1/4] FROM docker.io/library/golang:1.15-alpine@sha256:49b4eac11640066bc72c74b70202478b7d431c7d8918e0973d6e4aeb8b3129d2                             0.1s
 => => resolve docker.io/library/golang:1.15-alpine@sha256:49b4eac11640066bc72c74b70202478b7d431c7d8918e0973d6e4aeb8b3129d2                                                 0.0s
 => [linux/amd64 stage-1 1/3] FROM docker.io/library/alpine@sha256:3c7497bf0c7af93428242d6176e8f7905f2201d8fc5861f45be7a346b5f23436                                         0.1s
 => => resolve docker.io/library/alpine@sha256:3c7497bf0c7af93428242d6176e8f7905f2201d8fc5861f45be7a346b5f23436                                                             0.0s
 => [internal] load build context                                                                                                                                           0.0s
 => => transferring context: 178B                                                                                                                                           0.0s
 => CACHED [linux/amd64 stage-1 2/3] WORKDIR /opt                                                                                                                           0.0s
 => CACHED [linux/amd64 builder 2/4] WORKDIR /opt                                                                                                                           0.0s
 => CACHED [linux/amd64 builder 3/4] ADD . /opt/                                                                                                                            0.0s
 => CACHED [linux/amd64 builder 4/4] RUN go build -o test .                                                                                                                 0.0s
 => CACHED [linux/amd64 stage-1 3/3] COPY --from=builder /opt/test .                                                                                                        0.0s
 => CACHED [linux/arm64 stage-1 2/3] WORKDIR /opt                                                                                                                           0.0s
 => CACHED [linux/arm64 builder 2/4] WORKDIR /opt                                                                                                                           0.0s
 => CACHED [linux/arm64 builder 3/4] ADD . /opt/                                                                                                                            0.0s
 => CACHED [linux/arm64 builder 4/4] RUN go build -o test .                                                                                                                 0.0s
 => CACHED [linux/arm64 stage-1 3/3] COPY --from=builder /opt/test .                                                                                                        0.0s
 => exporting to image                                                                                                                                                      1.8s
 => => exporting layers                                                                                                                                                     0.6s
 => => exporting manifest sha256:714e4f18895bb698c65392eda3b39e35b38df0e84c34ce1f604bfa8fdb466a77                                                                           0.0s
 => => exporting config sha256:0339ab69edd09e4038046c8c056305733f050918725b6f265780189c0bdd76e3                                                                             0.0s
 => => exporting manifest sha256:a5c4695278c99a1801c557e68145d7a1e61fa82288a9d2b43b1f85219dea5b8a                                                                           0.0s
 => => exporting config sha256:a69d7dfa854e466d743d09c3c95ebca68f2feb6cb335bc0aecc4a256cc95e68f                                                                             0.0s
 => => exporting manifest list sha256:77677ebebf2dda103c6cd8598018a1d9bb79e06102173588fef5c501e81f6cb3                                                                      0.0s
 => => pushing layers                                                                                                                                                       0.5s
 => => pushing manifest for reg.xxxxxx.cn/app/test-golang:v1  

```

# 遇到的错误

## 上传镜像时x509: certificate signed by unknown authority

完整报错：

```Bash
failed to solve: rpc error: code = Unknown desc = failed to do request: Head [https://reg.xxxxxx.cn/v2/app/test-golang/blobs/sha256:0c1f186a05c7a4d0cb23b4a339473c5e96115be11f4deb24f74d3f2324120c87:](https://reg.xxxxxx.cn/v2/app/test-golang/blobs/sha256:0c1f186a05c7a4d0cb23b4a339473c5e96115be11f4deb24f74d3f2324120c87:) x509: certificate signed by unknown authority
```

这是因为公司所用的harbor证书是自签名证书，暂时的解决方法是将harbor证书加到builder容器中。

```Bash
$ BUILDER=$(docker ps | grep buildkitd  | awk '{print $1}')
$ docker cp ./harbor.crt $BUILDER:/usr/local/share/ca-certificates/ # harbor.crt为harbor ca证书
$ sudo docker exec $BUILDER update-ca-certificates
$ sudo docker restart $BUILDER 
```

重新上传镜像即可。

# 参考链接

[https://docs.docker.com/buildx/working-with-buildx/](https://docs.docker.com/buildx/working-with-buildx/)

[https://zhuanlan.zhihu.com/p/227048978](https://zhuanlan.zhihu.com/p/227048978)

[https://github.com/docker/buildx/issues/80](https://github.com/docker/buildx/issues/80)
