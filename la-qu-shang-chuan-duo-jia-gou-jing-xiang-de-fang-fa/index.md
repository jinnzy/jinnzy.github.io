# 拉取上传多架构镜像的方法


# 使用docker拉取上传

1. 拉取amd64镜像，并重命名`tag` 上传
```bash
docker pull --platform amd64 k8s.gcr.io/kube-apiserver:v1.18.5
docker tag k8s.gcr.io/kube-apiserver:v1.18.5 reg.xxx.cn/kube-apiserver:v1.18.5-amd64
docker push reg.xxx.cn/kube-apiserver:v1.18.5-amd64
```
2. 拉取arm64镜像
```bash
docker pull --platform arm64 k8s.gcr.io/kube-apiserver:v1.18.5
docker tag k8s.gcr.io/kube-apiserver:v1.18.5 reg.xxx.cn/kube-apiserver:v1.18.5-arm64
docker push reg.xxx.cn/kube-apiserver:v1.18.5-arm64
```
3. 创建清单
```bash
docker manifest create reg.xxx.cn/kube-apiserver:v1.18.5 \
   reg.xxx.cn/kube-apiserver:v1.18.5-arm64 \
   reg.xxx.cn/kube-apiserver:v1.18.5-amd64
```
4. 指定清单镜像中对应的架构
```bash
docker manifest annotate reg.xxx.cn/kube-apiserver:v1.18.5 \
   reg.xxx.cn/kube-apiserver:v1.18.5-arm64 --arch arm64

docker manifest annotate reg.xxx.cn/kube-apiserver:v1.18.5 \
   reg.xxx.cn/kube-apiserver:v1.18.5-amd64 --arch amd64
```
5. 上传镜像
```bash
docker manifest push reg.xxx.cn/kube-apiserver:v1.18.5
```

使用这种方式特别繁琐，查找一番发现了一个工具`skopeo`  可以使用该工具简化上传

# skopeo同步

项目地址：[https://github.com/containers/skopeo](https://github.com/containers/skopeo)

## 安装

该存储库可能包含比较新的内容，用于生产环境前请检查一下。

centos7:

```bash
sudo curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable.repo https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/CentOS_7/devel:kubic:libcontainers:stable.repo
sudo yum -y install skopeo
```

## 使用

```bash
skopeo copy --all docker://k8s.gcr.io/kube-apiserver:v1.18.15  docker://reg.xxx.cn/kube-apiserver:1.18.15
```
- —all 同步清单中所有镜像，如果不加的话只会复制当前系统架构的镜像

这样就很简单了，而且同步速度也特别快

