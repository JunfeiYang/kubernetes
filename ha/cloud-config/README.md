# 使用cloud-config在CoreOS之上安装kubernetes
本目录下包含了使用cloud-config文件完成kubernetes集群的安装的模版文件：

|文件|说明|
|---|---|
|etcd2_cc.yaml|配置本机启动etcd2服务|
|kubernetes_master_cc.yaml|安装配置本机作为kubernetes master(flannel网络)|
|kubernetes_master_cc_calico.yaml|使用calico作为默认网络方案的kubernetes master配置|
|kubernetes_node_cc.yaml|安装配置本机作为kubernetes的worker节点|
|kubernetes_node_cc_calico.yaml|calico方式的worker节点配置|
|skydns.yaml|初始化完成集群后，添加内网的dns服务|

## 操作步骤
首先确定要安装的集群选用的网络方案，是使用calico还是flannel。使用calico，则以下文档中使用的cloud-config需要使用后缀为```_calico.yaml```的文件
### 集群规划
下面的表格展示了一个样例集群的规划：

|主机|安装组件|
|---|---|
|kubernetes-master|etcd2第一个节点, kubernetes master|
|etcd2|etcd2的第二个节点|
|etcd3|etcd2的第三个节点|
|kube-node-01|worker 1|
|kube-node-02|worder 2|

### 确定目前使用的节点的自动执行coreos-cloudinit的配置方法
由于使用不同的主机（包括AWS虚拟机，Openstack虚拟机，物理机等）会导致coreos自动执行的cloud-config位置不一样，所以需要确定以下的cloud-config文件需要存放的位置，并配置让系统启动的时候自动找到我们配置好的cloud-config

### 配置kubernetes master
1. **基于Discovery server部署etcd2** 使用```etcd2_cc.yaml```配置作为etcd2的集群节点。这里默认是使用discovery方式配置集群(将注释掉的discovery一行加上，并根据说明生成token)
	* 修改```<node_name>```字段为当前节点的名称
1. **基于Static Mode部署etcd2** 使用```etcd2_cc_static.yaml```配置作为etcd2的集群节点。对以下变量做出修改:
	* 修改```<node_name>```为当前节点的名称，分别为infra0,infra1,infra2
	* 修改```<ipv4_ip>```为当前节点的IP地址
	* 修改```<infra0_ip>,<infra1_ip>,<infra2_ip>```为三个节点的IP地址
1. ```kubernetes_master_cc.yaml```，修改```<SSH_PUBLIC_KEY>```为你本机的ssh公钥，参考[这里](https://linuxconfig.org/passwordless-ssh)
1. ```kubernetes_master_cc.yaml```，修改```<MY_ETCD_ENDPOINTS>```为etcd集群的endpoints串，对于此例的集群规划，可以配置```http:\/\/kubernetes-master:2379,http:\/\/etcd2:2379,http:\/\/etcd3:2379```。***注意：版本较低的skydns和nginx-ingress-controller不支持配置ETCD_ENDPOINTS，只能配置一个etcd的地址***
1. ```kubernetes_master_cc.yaml```，修改```<MASTER_IP>```为本机的IP地址
1. 修改```${ETCD_ENDPOINTS}```为ETCD集群的地址不需要转义:```http://kubernetes-master:2379,http://etcd2:2379,http://etcd3:2379```
1. 如果是calico模式，在```kubernetes_master_cc_calico.yaml```修改```<ETCD_AUTHORITY>```为etcd集群的一台机器的ip端口: ```kubernetes-master:2379```
1. 执行```coreos-cloudinit --from-file kubernetes_master_cc.yaml```完成安装


### 配置TLS
master节点和worker的通信以及和client的通信都需要基于[TLS](https://github.com/k8sp/tls)相互信任。master需要CA证书```ca.pem```，其自身的证书```apiserver.pem```和公私钥对```apiserver-key.pem```。CoreOS的[这个](https://coreos.com/kubernetes/docs/latest/openssl.html)文档解释如何生成这些文件。

1. 根据[证书生成说明](../tls/README.md)执行脚本生成master证书

1. 重启kubelet，确保master正确读取这些key，如果在配置master之前完成这一步，可以无需重启
  ```
  sudo systemctl restart kubelet
  ```

### 配置worker节点
使用```kubernetes_node_cc.yaml```配置并启动worker节点。使用下面的步骤完成对worker节点的cloud-config的配置:

1. ```<HOSTNAME>```: 本机的Hostname(e.g. kube-node1, kube-node2)
1. ```<SSH_PUBLIC_KEY>```: 你本机的ssh公钥
1. ```<KUBERNETES_MASTER>```: 前面步骤配置的master节点的IP地址
1. ```<ETCD_AUTHORITY>```: etcd的一个节点的IP端口，如上
1. ```<MY_ETCD_ENDPOINTS>```: 配置etcd集群访问的链接串，同上配置即可
1. ```<CA_CERT>```,粘贴ca.pem的内容,比如:

	```
	-----BEGIN CERTIFICATE-----
	MIIC9zCCA
	......
	sRj2yQ==
	-----END CERTIFICATE-----
	```

1. ```<CA_KEY_CERT>```: 粘贴ca-key.pem的内容
1. ```<IFACE>```: 集群内部通信的网卡

## 使用L2网络方式的部署master和node
参考链接:https://github.com/k8sp/kubernetes/blob/master/networking/performance.md

## TODO
* 为了能完成全自动的kubernetes worker的部署和启动，需要有以下功能
 * 一个内网key生成和管理的服务，用来自动的为每个新启动的worker配置worker的key
 * 自动从一个中心的地方获得每个worker对应的cloud-config文件
* ```<ETCD_AUTHORITY>```，应该使用高版本的ETCD_ENDPOINTS替换
