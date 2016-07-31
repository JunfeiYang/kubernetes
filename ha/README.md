# 通过service实现kubernetes的ha

### 集群规划
下面的表格展示了一个样例集群的规划：

|主机|安装组件|
|---|---|
|kubernetes-master01|etcd2第一个节点, kubernetes master|
|kubernetes-master02|etcd2第二个节点, kubernetes master|
|kubernetes-master03|etcd2第三个节点, kubernetes master|
|kubernetes-worker-01|kubernetes worker 1|
|kubernetes-worker-02|kubernetes worder 2|

### kubernetes架构图


### kubernete集群高可用方案
kubernetes作为容器的管理中心，通过对pod的数量进行监控，并根据主机或者容器失效的状态将新的pod调度到其他node上，实现了应用层面的高可用性。针对Kubernetes集群，高可用性还应包含一下两个层面的考虑：etcd数据存储的高可用性和kubernetes master组件的高可用性。
####etcd的高可用性方案
etcd在整个kubernetes集群中处于中心数据库的地位，为保证kubernetes集群的高可用性，一方面，etcd需要以集群的方式进行部署，以实现etcd数据存储的冗余，备份与高可用性，另一方面,etcd存储的数据本身也应考虑使用可靠的存储设备。对于etcd中需要保存的数据可靠性，可以考虑使用RAID磁盘阵列，高性能存储设备，NFS网络文件系统，或者使用云服务提供商的网盘系统等实现。
####kubernetes组件的高可用性方案
kubernetes master节点扮演者总控制中心的角色，主要的三个服务kube-apiserver,kube-controller-manage和kube-sheduler通过不断的与工作节点的kubelet和kube-proxy进行通信维护整个集群的健康状态。如果master的服务无法访问某个node,则会将该node标记为不可用，不再向其调度创建新的pod，但对master自身则需要进行额外的监控，使用master不在成为集群的单故障点，所以对master服务也需要进行高可用的方式部署。

###　部署kubernetes集群环境
#####etcd集群的部署
参考https://github.com/k8sp/bigdata/tree/master/install/cloud-config
部署etcd的方案，将etcd分别部署到kubernetes-master01，kubernetes-master02，kubernetes-master03上，可以使用static方式也可以使用基于token的服务发现的方式。
#####kubernetes master集群的部署
kubernetes master 会启动几个服务kube-apiserver,kube-controller-manage和kube-sheduler，master在集群环境下，多台master中的api server是同时工作的，而对集群状态存在修改动作的kube-controller-manage和kube-sheduler，同时只有一个实例在运行。为实现master节点的高可用性，首先需要一个load balancer，将请求转发给各个master节点，如果某一个master节点down机，如果转发规则没有及时更新，就会存在无效请求，导致服务不可用的状态，所以还需要对master节点存在健康检查的操作。
参考https://github.com/k8sp/bigdata/tree/master/install/cloud-config
部署master的方案。在部署前注意需要修改的地方主要有一下几点：
１.将kubernetes_master.manifest中
```
metadata: 
name: kube-controller
```
修改为
```
metadata: 
  name: kube-controller
  labels:
service: k8s
```
每个master节点会启动一个pod，该pod里面包含四个容器，分别为kube-apiserver,kube-controller-manage和kube-sheduler,kube-proxy,并打上labels为service: k8s的标签，后面定义的service会根据该labels，通过selector选取pod实现master负载均衡
2.多master集群部署和单master部署方式一样，需要修改kubernetes_master_cc.yaml，修改<MASTER_IP>为本机的IP地址，及修改<MY_ETCD_ENDPOINTS>为etcd集群的endpoints串
#####创建service做为load balancer
```
apiVersion: v1
kind: Service
metadata:
  name: k8s
  labels:
    service: allk8s
spec:
  ports:
    - name: client
      port: 443
  selector:
service: k8s
```
由于kube-controller-manager已开启livenessProbe探针
```
- name: kube-controller-manager
      image: typhoon1986/hyperkube-amd64:v1.2.0
      command:
      - /hyperkube
      - controller-manager
      - --master=http://127.0.0.1:8080
      - --service-account-private-key-file=/etc/kubernetes/ssl/apiserver-key.pem
      - --root-ca-file=/etc/kubernetes/ssl/ca.pem
      livenessProbe:
        httpGet:
          host: 127.0.0.1
          path: /healthz
          port: 10252s
        initialDelaySeconds: 15
timeoutSeconds: 1
```
kube-scheduler已开启livenessProbe探针
```
  - name: kube-scheduler
      image: typhoon1986/hyperkube-amd64:v1.2.0
      command:
      - /hyperkube
      - scheduler
      - --master=http://127.0.0.1:8080
      livenessProbe:
        httpGet:
          host: 127.0.0.1
          path: /healthz
          port: 10251
        initialDelaySeconds: 15
timeoutSeconds: 1
```
详细见kubernetes_master.manifest文件。所以该service,通过selector选取labels为service:k8s为的pod时，默认会根据pod的健康状态，更新该service对应的endpoints列表
#####kubernetes　worker集群的部署
参考https://github.com/k8sp/bigdata/tree/master/install/cloud-config
部署worker的方案，不同的是<KUBERNETES_MASTER>的地址不是设置为kubernetes　master的ip地址，而是load balancer的cluster ip地址
#####配置TLS
master节点和worker的通信以及和client的通信都需要基于TLS相互信任。
参考https://github.com/k8sp/bigdata/tree/master/install/cloud-config
部署TLS. 
## TODO
如果master节点down机，而定义的service对应的endpoints列表更新大概需要20-30秒，仍然存在无效请求的状况。基于service实现的load balancer底层主要基于iptables中nat转换的rule实现，需要对另外一种方案haproxy+keepalive实现的ha方案做一个性能方面的对比。

