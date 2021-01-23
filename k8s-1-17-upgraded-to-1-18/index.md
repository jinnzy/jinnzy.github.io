# k8s 1.17 升级到 1.18 

# k8s 1.17 升级到 1.18

# 环境信息

当前版本：`v1.17.5`

目标版本：`1.18.15`

升级方式：`kubeadm`

推荐升级的版本为当前版本+1如`1.16` 升级到`1.17` ，`1.17` 升级到`1.18`

查看当前`yum` 源里是否有模板版本，如果没有的话需要更新`yum` 源。

```
yum list --showduplicates kubeadm --disableexcludes=kubernetes
···
kubeadm.x86_64                                                                       1.18.13-0                                                                        kubernetes 
kubeadm.x86_64                                                                       1.18.14-0                                                                        kubernetes 
kubeadm.x86_64                                                                       1.18.15-0                                                                        kubernetes 
kubeadm.x86_64                                                                       1.19.0-0                                                                         kubernetes 
kubeadm.x86_64                                                                       1.19.1-0                                                                         kubernetes 
kubeadm.x86_64                                                                       1.19.2-0                                                                         kubernetes
···
```

# 升级master节点

## 准备升级所需镜像

查看所需镜像，将这些镜像拉取下来传到私有仓库里，可使用`kubeadm config images list` 查看所需镜像版本。

```
k8s.gcr.io/kube-apiserver:v1.18.15
k8s.gcr.io/kube-controller-manager:v1.18.15
k8s.gcr.io/kube-scheduler:v1.18.15
k8s.gcr.io/kube-proxy:v1.18.15
k8s.gcr.io/pause:3.2
k8s.gcr.io/etcd:3.4.3-0
k8s.gcr.io/coredns:1.6.7
```

我目前是从`registry.cn-hangzhou.aliyuncs.com/google_containers` 这个仓库拉取镜像。

```
pullReg=registry.cn-hangzhou.aliyuncs.com/google_containers
pushReg=reg.xxx.cn/k8s.gcr.io
for i in $(echo -n 'k8s.gcr.io/kube-apiserver:v1.18.15
k8s.gcr.io/kube-controller-manager:v1.18.15
k8s.gcr.io/kube-scheduler:v1.18.15
k8s.gcr.io/kube-proxy:v1.18.15
k8s.gcr.io/pause:3.2
k8s.gcr.io/coredns:1.6.7');do app=$(echo $i|awk -F 'k8s.gcr.io/' '{print $2}') && \
docker pull ${pullReg}/${app} && \
docker tag ${pullReg}/${app} ${pushReg}/${app} && \
docker push ${pushReg}/${app}; done
```

## 升级master节点

首先升级`kubeadm`

```
yum -y install kubeadm-1.18.15-0 kubectl-1.18.15
```

查看升级计划

```
[root@vlnx107024 ~]# kubeadm upgrade plan
···
External components that should be upgraded manually before you upgrade the control plane with 'kubeadm upgrade apply':
COMPONENT   CURRENT   AVAILABLE
Etcd        3.4.9     3.4.3-0

Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
COMPONENT   CURRENT       AVAILABLE
Kubelet     5 x v1.17.7   v1.18.15

Upgrade to the latest version in the v1.17 series:

COMPONENT            CURRENT   AVAILABLE
API Server           v1.17.7   v1.18.15
Controller Manager   v1.17.7   v1.18.15
Scheduler            v1.17.7   v1.18.15
Kube Proxy           v1.17.7   v1.18.15
CoreDNS              1.6.5     1.6.7

You can now apply the upgrade by executing the following command:

  kubeadm upgrade apply v1.18.15
```

因为这个`k8s`集群是使用集群外的`etcd` 方式，所以版本比较新，如果版本低于`AVAILABLE` 的版本，需要先升级`etcd`

开始升级第一个节点：

```
kubeadm upgrade apply -y 1.18.15 --config=/root/kubeadm-config.yml --ignore-preflight-errors=all --force
```

`kubeadm upgrade apply` 参数：

- `--allow-experimental-upgrades` : 显示不稳定版本，允许升级到alpha/beta或rc版本。
- `--certificate-renewal` : 默认为true，执行升级期间更新证书
- `--config` : kubeadm配置文件路径，如果没有更新配置文件可以不添加
- `--force` : 强制升级，非交互模式
- `--ignore-preflight-errors` : 为`all` 时忽略检查中的所有错误
- `--print-config` : 是否打印升级中使用的配置文件
- -`y, --yes` : 非交互模式。

查看更新节点的`apiserver`  `controler manager`  `kube-schdule`  状态及版本是否正常

```bash
kubectl -n kube-system  get po
```

更新成功后重启`kubelet` 

```bash
systemctl restart kubelet
```

查看集群内当前节点的版本，为`1,18,15` 就是正常了

```bash
kubectl get nodes
```

随后依次更新另外两个master节点

```go
kubeadm    upgrade apply -y 1.18.15   --ignore-preflight-errors=all    --force
```

## 升级node节点

node节点升级操作就更简单了

首先同样是安装包

```bash
yum -y install kubeadm-1.18.15-0 kubectl-1.18.15
```

最好是清空当前节点，不过由于我是升级测试环境就省略这一步，升级完节点由于结构体有变化，hash值变更会重启当前节点所有的pod进行更新。

```bash
kubectl drain --delete-local-data --force --ignore-daemonsets --grace-period=0  ${currentNodeName}
```

开始更新

```bash
kubeadm upgrade node
```

重启`kubelet`

```bash
systemctl restart kubelet
```

检查节点状态

```bash
kubectl get nodes
```

没问题后重复此步骤，依次升级剩余节点即可。

# 参考资料

[https://v1-18.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/](https://v1-18.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)

[https://kubernetes.io/zh/docs/reference/setup-tools/kubeadm/kubeadm-upgrade/](https://kubernetes.io/zh/docs/reference/setup-tools/kubeadm/kubeadm-upgrade/)
