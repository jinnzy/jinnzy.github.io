# 使用mitogen加速ansible


# 概述

前几天使用ansible通过公网方式建立的vpn部署k8s时遇到了几个问题，特别慢，时不时总是断开连接，导致playbook执行不下去，使用mitogen可以解决这些问题。

根据官方文档所述，可以提升1.25-7倍的速度，降低2倍cpu使用。

# 安装

注意：目前mitogen-0.2.9最新版本仅支持2.9.x的ansible，使用时请注意版本

我是在自己整理的playbook目录下操作的

```
wget https://networkgenomics.com/try/mitogen-0.2.9.tar.gz
```

![3cd815428d7279129bc5666ca80873c1a1092712.png](/img/3cd815428d7279129bc5666ca80873c1a1092712.png)

编辑当前目录下的ansible.cfg，添加以下配置，就可体验飞一般的速度了。

```
[defaults]
strategy_plugins = mitogen-0.2.9/ansible_mitogen/plugins/strategy
strategy = mitogen_linear
```

