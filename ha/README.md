# 使用cloud-config在CoreOS之上安装kubernetes
本目录下包含了使用cloud-config文件完成kubernetes集群的安装的模版文件：

|文件|说明|
|---|---|
|etcd2_cc.yaml|配置本机启动etcd2服务|
|kubernetes_master_cc.yaml|安装配置本机作为kubernetes master(flannel网络)|
|kubernetes_node_cc.yaml|安装配置本机作为kubernetes的worker节点|
|skydns.yaml|初始化完成集群后，添加内网的dns服务|

### 集群规划
下面的表格展示了一个样例集群的规划：

|主机|安装组件|
|---|---|
|kubernetes-master01|etcd2第一个节点, kubernetes master|
|kubernetes-master02|etcd2第二个节点, kubernetes master|
|kubernetes-master03|etcd2第三个节点, kubernetes master|
|kube-node-01|worker 1|
|kube-node-02|worder 2|


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
1. 执行```coreos-cloudinit --from-file kubernetes_master_cc.yaml```完成安装


### 配置TLS


### 配置worker节点
使用```kubernetes_node_cc.yaml```配置并启动worker节点。使用下面的步骤完成对worker节点的cloud-config的配置:


## TODO

