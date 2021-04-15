# 使用kubeadm手动更新证书


由于线上很多k8s集群的证书要过期了，虽然升级集群会更新证书，但是线上很多专属云k8s集群不方便升级，选择了手动更新证书，这里记录下更新步骤。

> 本次操作已在1.18.x/1.17.x版本中验证，如果是高版本需要去掉alpha命令。
> kubeadm不能管理外部生成的CA证书。

### 1. 查看证书过期时间

```
# kubeadm alpha certs check-expiration
[check-expiration] Reading configuration from the cluster...
[check-expiration] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'

CERTIFICATE                EXPIRES                  RESIDUAL TIME   CERTIFICATE AUTHORITY   EXTERNALLY MANAGED
admin.conf                 Apr 29, 2021 06:17 UTC   14d                                     no      
apiserver                  Apr 29, 2021 06:17 UTC   14d             ca                      no      
apiserver-kubelet-client   Apr 29, 2021 06:17 UTC   14d             ca                      no      
controller-manager.conf    Apr 29, 2021 06:17 UTC   14d                                     no      
front-proxy-client         Apr 29, 2021 06:17 UTC   14d             front-proxy-ca          no      
scheduler.conf             Apr 29, 2021 06:17 UTC   14d                                     no      

CERTIFICATE AUTHORITY   EXPIRES                  RESIDUAL TIME   EXTERNALLY MANAGED
ca                      Feb 20, 2030 05:59 UTC   8y              no      
front-proxy-ca          Feb 20, 2030 05:59 UTC   8y              no
```

**注意**：搭建集群时我是使用的外部etcd,所以并没有显示etcd相关的证书。

### 2. 备份

证书

```
cp -rp /etc/kubernetes /etc/kubernetes.bak

```

etcd,如果是使用kubeadm创建的etcd最好备份一下etcd,我这里使用的外部etcd并且有定时备份，所以就不备份了

### 3. 更新证书，三台master都执行

```
# kubeadm alpha certs renew all 
[renew] Reading configuration from the cluster...
[renew] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'

certificate embedded in the kubeconfig file for the admin to use and for kubeadm itself renewed
certificate for serving the Kubernetes API renewed
certificate for the API server to connect to kubelet renewed
certificate embedded in the kubeconfig file for the controller manager to use renewed
certificate for the front proxy client renewed
certificate embedded in the kubeconfig file for the scheduler manager to use renewed
```

也可以使用—config=/root/kubeadm-config.yml手动指定配置文件，我这里没有加，默认使用kube-system kubeadm-config配置

### 4. 重启kube-apiserver,kube-controller,kube-scheduler，使用新证书，如果etcd使用kubeadm启动的，也需要重启

```
docker ps |egrep "k8s_kube-apiserver|k8s_kube-scheduler|k8s_kube-controller"|awk '{print $1}'|xargs docker restart
```

### 5. 检查证书

```
# kubeadm alpha certs check-expiration
[check-expiration] Reading configuration from the cluster...
[check-expiration] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'

CERTIFICATE                EXPIRES                  RESIDUAL TIME   CERTIFICATE AUTHORITY   EXTERNALLY MANAGED
admin.conf                 Apr 15, 2022 03:15 UTC   364d                                    no      
apiserver                  Apr 15, 2022 03:15 UTC   364d            ca                      no      
apiserver-kubelet-client   Apr 15, 2022 03:15 UTC   364d            ca                      no      
controller-manager.conf    Apr 15, 2022 03:15 UTC   364d                                    no      
front-proxy-client         Apr 15, 2022 03:15 UTC   364d            front-proxy-ca          no      
scheduler.conf             Apr 15, 2022 03:15 UTC   364d                                    no      

CERTIFICATE AUTHORITY   EXPIRES                  RESIDUAL TIME   EXTERNALLY MANAGED
ca                      Jan 13, 2031 12:05 UTC   9y              no      
front-proxy-ca          Jan 13, 2031 12:05 UTC   9y              no
```

/etc/kubernetes/admin.conf 证书

```
# cat /etc/kubernetes/admin.conf 
client-certificate-data: LS0tLS1CRUdJTiBDRVJUSU
......
dy9wCkdNNHAzRVBUNFQvN0E5b0sxWk1LRXZCTjRQYnRIV1FpRDN5K3R4TlQwa0RsNTNhUXFHcz0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=

base64解析
# echo LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUM4akNDQWRxZ0F3SUJBZ0lJS0tCaVB2WFpIN3N3RFFZSktvWklodmNOQV......
Re0E5b0sxWk1LRXZCTjRQYnRIV1FpRDN5K3R4TlQwa0RsNTNhUXFHcz0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo= | base64 -d
-----BEGIN CERTIFICATE-----
MIIC8jCCAdqgAwIBAgIIKKBiPvXZH7swDQYJKoZIhvcNAQELBQAwFTETMBEGA1UE
......
yLRf16OWsanlgH9yed/2pJcvpSRNBoBugOf8wScbR9QxZz9az6INQ95dTBi7zw/p
GM4p3EPT4T/7A9oK1ZMKEvBN4PbtHWQiD3y+txNT0kDl53aQqGs=
-----END CERTIFICATE-----
```

拿到ssl证书网站分析一下，也更新了

![d2d71ba3f4f77fa4f9ff353fb1ea0cfa357cb181.png](/img/d2d71ba3f4f77fa4f9ff353fb1ea0cfa357cb181.png)

各个节点的kubelet的证书到期后会自动更新，正好之前集群kubelet节点有一个到期了的，检查日志发现会自动更新，所以就不需要担心了。

```
certificate rotation detected, shutting down client connections to start using new credentials
```

# 参考资料

[https://kubernetes.io/zh/docs/reference/setup-tools/kubeadm/kubeadm-certs/](https://kubernetes.io/zh/docs/reference/setup-tools/kubeadm/kubeadm-certs/)

