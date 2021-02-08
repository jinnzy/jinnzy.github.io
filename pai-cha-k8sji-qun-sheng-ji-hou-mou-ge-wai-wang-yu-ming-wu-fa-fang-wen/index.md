# 排查k8s集群升级后某个外网域名无法访问


线下测试环境升级k8s集群过了几天后，研发同学报了一个问题，myhuaweicloud.com域名无法访问，之前是可以的。

进入到所在pod内，`ping myhuaweicloud.com`试了下无法解析域名，再`ping baidu.com` 和其他域名是正常的，猜测大概率问题出在了coredns上，先把coredns debug日至打开观察一下

```
.:53 {
     ...
      log
      debug
     ...
}
```

在容器内`pingmyhuaweicloud.com` ，查看日志发现search域 和绝对域名都解析不到，这就很奇怪了。

![9dac80d7ddb34991072ebce5b858e12170891fba.png](/img/9dac80d7ddb34991072ebce5b858e12170891fba.png)

然后查看了一下coredns配置，发现了问题，因为我们k8s内应用访问外网统一都是走的egress,有一些sdk支持代理就直接使用了，还有一些无法使用代理的，我们会在coredns把域名解析到egress的地址上，这样就会走egress代理了。 

问题就在于上次升级后把coredns也更新了，configmap添加的custom.db文件和deployment挂载的配置都没了，随后把这个添加上恢复正常了。

![cfb0c078bed5034877e5db964a40e473ce649f23.png](/img/cfb0c078bed5034877e5db964a40e473ce649f23.png)![184a82920cacbe03dd0440ac90e5d11344c06c23.png](/img/184a82920cacbe03dd0440ac90e5d11344c06c23.png)

后面要注意升级k8s后，这些系统组件的配置变化，避免踩坑。

