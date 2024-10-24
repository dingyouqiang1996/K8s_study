# 什么是kubeasz

**kubeasz** 致力于提供快速部署高可用`k8s`集群的工具, 同时也努力成为`k8s`实践、使用的参考书；基于二进制方式部署和利用`ansible-playbook`实现自动化；既提供一键安装脚本, 也可以根据`安装指南`分步执行安装各个组件。

**kubeasz** 从每一个单独部件组装到完整的集群，提供最灵活的配置能力，几乎可以设置任何组件的任何参数；同时又为集群创建预置一套运行良好的默认配置，甚至自动化创建适合大规模集群的[BGP Route Reflector网络模式](https://github.com/easzlab/kubeasz/blob/master/docs/setup/network-plugin/calico-bgp-rr.md)。

- **集群特性** [Master高可用](https://github.com/easzlab/kubeasz/blob/master/docs/setup/00-planning_and_overall_intro.md#ha-architecture)、[离线安装](https://github.com/easzlab/kubeasz/blob/master/docs/setup/offline_install.md)、[多架构支持(amd64/arm64)](https://github.com/easzlab/kubeasz/blob/master/docs/setup/multi_platform.md)
- **集群版本** kubernetes v1.24, v1.25, v1.26, v1.27, v1.28, v1.29, v1.30
- **运行时** [containerd](https://github.com/easzlab/kubeasz/blob/master/docs/setup/03-container_runtime.md) v1.6.x, v1.7.x
- **网络** [calico](https://github.com/easzlab/kubeasz/blob/master/docs/setup/network-plugin/calico.md), [cilium](https://github.com/easzlab/kubeasz/blob/master/docs/setup/network-plugin/cilium.md), [flannel](https://github.com/easzlab/kubeasz/blob/master/docs/setup/network-plugin/flannel.md), [kube-ovn](https://github.com/easzlab/kubeasz/blob/master/docs/setup/network-plugin/kube-ovn.md), [kube-router](https://github.com/easzlab/kubeasz/blob/master/docs/setup/network-plugin/kube-router.md)

**[news]** kubeasz 通过cncf一致性测试 [详情](https://github.com/easzlab/kubeasz/blob/master/docs/mixes/conformance.md)

推荐版本对照

| Kubernetes | 1.22  | 1.23  | 1.24  | 1.25  | 1.26  | 1.27  | 1.28  | 1.29  | 1.30  |
| ---------- | ----- | ----- | ----- | ----- | ----- | ----- | ----- | ----- | ----- |
| kubeasz    | 3.1.1 | 3.2.0 | 3.6.2 | 3.6.2 | 3.6.2 | 3.6.2 | 3.6.2 | 3.6.3 | 3.6.4 |

## 支持系统

- **Alibaba Linux** 2.1903, 3.2104([notes](https://github.com/easzlab/kubeasz/blob/master/docs/setup/multi_os.md#Alibaba))
- **Alma Linux** 8, 9
- **Anolis OS** 8.x RHCK, 8.x ANCK
- **CentOS/RHEL** 7, 8, 9
- **Debian** 10, 11([notes](https://github.com/easzlab/kubeasz/blob/master/docs/setup/multi_os.md#Debian))
- **Fedora** 34, 35, 36, 37
- **Kylin Linux Advanced Server V10** 麒麟V10 Tercel, Lance
- **openSUSE** Leap 15.x([notes](https://github.com/easzlab/kubeasz/blob/master/docs/setup/multi_os.md#openSUSE))
- **Rocky Linux** 8, 9
- **Ubuntu** 16.04, 18.04, 20.04, 22.04, 24.04



# 安装集群

## 设置ip



~~~powershell
k8s-master01节点IP地址为：192.168.229.170/24
root@k8s-master01:~# vim /etc/netplan/01-network-manager-all.yaml
root@k8s-master01:~# cat /etc/netplan/01-network-manager-all.yaml

cat > /etc/netplan/01-network-manager-all.yaml <<EOF
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:
      dhcp4: no
      addresses:
        - 192.168.229.170/24
      routes:
        - to: default
          via: 192.168.229.2
      nameservers:
        addresses: [119.29.29.29,114.114.114.114,8.8.8.8]
EOF
~~~



~~~powershell
systemctl restart systemd-networkd
netplan apply
~~~



~~~powershell
k8s-master02节点IP地址为：192.168.229.171/24
root@k8s-worker01:~# vim /etc/netplan/01-network-manager-all.yaml
root@k8s-worker01:~# cat /etc/netplan/01-network-manager-all.yaml

cat > /etc/netplan/01-network-manager-all.yaml <<EOF
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:
      dhcp4: no
      addresses:
        - 192.168.229.171/24
      routes:
        - to: default
          via: 192.168.229.2
      nameservers:
        addresses: [119.29.29.29,114.114.114.114,8.8.8.8]
EOF
~~~



~~~powershell
systemctl restart systemd-networkd
netplan apply
~~~





~~~powershell
k8s-worker01节点IP地址为：192.168.229.172/24
root@k8s-worker02:~# vim /etc/netplan/01-network-manager-all.yaml
root@k8s-worker02:~# cat /etc/netplan/01-network-manager-all.yaml

cat > /etc/netplan/01-network-manager-all.yaml <<EOF
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:
      dhcp4: no
      addresses:
        - 192.168.229.172/24
      routes:
        - to: default
          via: 192.168.229.2
      nameservers:
        addresses: [119.29.29.29,114.114.114.114,8.8.8.8]
EOF
~~~







~~~powershell
k8s-worker03节点IP地址为：192.168.229.173/24
root@k8s-worker02:~# vim /etc/netplan/01-network-manager-all.yaml
root@k8s-worker02:~# cat /etc/netplan/01-network-manager-all.yaml

cat > /etc/netplan/01-network-manager-all.yaml <<EOF
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:
      dhcp4: no
      addresses:
        - 192.168.229.173/24
      routes:
        - to: default
          via: 192.168.229.2
      nameservers:
        addresses: [119.29.29.29,114.114.114.114,8.8.8.8]
EOF
~~~







~~~powershell
kubeasz节点IP地址为：192.168.229.174/24
root@k8s-worker02:~# vim /etc/netplan/01-network-manager-all.yaml
root@k8s-worker02:~# cat /etc/netplan/01-network-manager-all.yaml

cat > /etc/netplan/01-network-manager-all.yaml <<EOF
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:
      dhcp4: no
      addresses:
        - 192.168.229.174/24
      routes:
        - to: default
          via: 192.168.229.2
      nameservers:
        addresses: [119.29.29.29,114.114.114.114,8.8.8.8]
EOF
~~~



~~~powershell
systemctl restart systemd-networkd
netplan apply
~~~



## 准备安装



```
# 国内环境
./ezdown -D
# 海外环境
#./ezdown -D -m standard

# 容器化运行kubeasz
./ezdown -S

# 创建新集群 k8s-01
docker exec -it kubeasz ezctl new k8s-01

然后根据提示配置'/etc/kubeasz/clusters/k8s-01/hosts' 和 '/etc/kubeasz/clusters/k8s-01/config.yml'：根据前面节点规划修改hosts 文件和其他集群层面的主要配置选项；其他集群组件等配置项可以在config.yml 文件中修改。


```

```
root@mark-VMware-Virtual-Platform:~/kubeasz-3.6.4# cat /etc/kubeasz/clusters/k8s-01/hosts 
# 'etcd' cluster should have odd member(s) (1,3,5,...)
[etcd]
192.168.229.170
192.168.229.171
192.168.229.172

# master node(s), set unique 'k8s_nodename' for each node
# CAUTION: 'k8s_nodename' must consist of lower case alphanumeric characters, '-' or '.',
# and must start and end with an alphanumeric character
[kube_master]
192.168.229.170 k8s_nodename='master-01'
192.168.229.171 k8s_nodename='master-02'

# work node(s), set unique 'k8s_nodename' for each node
# CAUTION: 'k8s_nodename' must consist of lower case alphanumeric characters, '-' or '.',
# and must start and end with an alphanumeric character
[kube_node]
192.168.229.172 k8s_nodename='worker-01'
192.168.229.173 k8s_nodename='worker-02'

# [optional] harbor server, a private docker registry
# 'NEW_INSTALL': 'true' to install a harbor server; 'false' to integrate with existed one
[harbor]
#192.168.1.8 NEW_INSTALL=false

# [optional] loadbalance for accessing k8s from outside
[ex_lb]
#192.168.1.6 LB_ROLE=backup EX_APISERVER_VIP=192.168.1.250 EX_APISERVER_PORT=8443
#192.168.1.7 LB_ROLE=master EX_APISERVER_VIP=192.168.1.250 EX_APISERVER_PORT=8443

# [optional] ntp server for the cluster
[chrony]
#192.168.1.1

[all:vars]
# --------- Main Variables ---------------
# Secure port for apiservers
SECURE_PORT="6443"

# Cluster container-runtime supported: docker, containerd
# if k8s version >= 1.24, docker is not supported
CONTAINER_RUNTIME="containerd"

# Network plugins supported: calico, flannel, kube-router, cilium, kube-ovn
CLUSTER_NETWORK="calico"

# Service proxy mode of kube-proxy: 'iptables' or 'ipvs'
PROXY_MODE="ipvs"

# K8S Service CIDR, not overlap with node(host) networking
SERVICE_CIDR="10.68.0.0/16"

# Cluster CIDR (Pod CIDR), not overlap with node(host) networking
CLUSTER_CIDR="172.20.0.0/16"

# NodePort Range
NODE_PORT_RANGE="30000-32767"

# Cluster DNS Domain
CLUSTER_DNS_DOMAIN="cluster.local"

# -------- Additional Variables (don't change the default value right now) ---
# Binaries Directory
bin_dir="/opt/kube/bin"

# Deploy Directory (kubeasz workspace)
base_dir="/etc/kubeasz"

# Directory for a specific cluster
cluster_dir="{{ base_dir }}/clusters/k8s-01"

# CA and other components cert/key Directory
ca_dir="/etc/kubernetes/ssl"

# Default 'k8s_nodename' is empty
k8s_nodename=''

# Default python interpreter
ansible_python_interpreter=/usr/bin/python3
```



## 安装

```
ssh-keygen
ssh-copy-id root@192.168.229.170
ssh-copy-id root@192.168.229.171
ssh-copy-id root@192.168.229.172
ssh-copy-id root@192.168.229.173
ssh-copy-id root@192.168.229.174


docker exec -it kubeasz ezctl setup k8s-01 all

scp kubectl.kubeconfig  root@192.168.229.170:/root/.kube/config
```





# 删除节点

```
docker exec -it kubeasz ezctl del-node k8s-01 192.168.229.173
```



# 添加节点

```
docker exec -it kubeasz ezctl add-node k8s-01 192.168.229.173 k8s_nodename='worker-02'
```



# 清理

```
docker exec -it kubeasz ezctl destroy k8s-01
```

