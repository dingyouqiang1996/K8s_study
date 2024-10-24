# kubeadm部署高可用kubernetes集群 1.30   

# 二、kubernetes 1.30.0 部署工具介绍

## What is Kubeadm ?

`Kubeadm is a tool built to provide best-practice "fast paths" for creating Kubernetes clusters. It performs the actions necessary to get a minimum viable, secure cluster up and running in a user friendly way. Kubeadm's scope is limited to the local node filesystem and the Kubernetes API, and it is intended to be a composable building block of higher level tools.`

Kubeadm是为创建Kubernetes集群提供最佳实践并能够“快速路径”构建kubernetes集群的工具。它能够帮助我们执行必要的操作，以获得最小可行的、安全的集群，并以用户友好的方式运行。



## Common Kubeadm cmdlets

1. **kubeadm init** to bootstrap the initial Kubernetes control-plane node. `初始化`
2. **kubeadm join** to bootstrap a Kubernetes worker node or an additional control plane node, and join it to the cluster. `添加工作节点到kubernetes集群`
3. **kubeadm upgrade** to upgrade a Kubernetes cluster to a newer version. ` 更新kubernetes版本`
4. **kubeadm reset** to revert any changes made to this host by kubeadm init or kubeadm join. ` 重置kubernetes集群`



# 三、kubernetes 1.30.0 部署环境准备

>3主2从

## 3.1 主机操作系统说明

| 序号 | 操作系统及版本  | 备注 |
| :--: | :-------------: | :--: |
|  1   | CentOS stream 9 |      |



## 3.2 主机硬件配置说明

| 需求 | CPU  | 内存 | 硬盘  | 角色         | 主机名   |
| ---- | ---- | ---- | ----- | ------------ | -------- |
| 值   | 2C   | 6G   | 120GB | master       | master01 |
| 值   | 2C   | 6G   | 120GB | master       | master02 |
| 值   | 2C   | 6G   | 120GB | master       | master03 |
| 值   | 2C   | 6G   | 120GB | worker(node) | worker01 |
| 值   | 2C   | 6G   | 120GB | worker(node) | worker02 |

| 序号 | 主机名   | IP地址          | 备注   |
| ---- | -------- | --------------- | ------ |
| 1    | master01 | 192.168.229.170 | master |
| 2    | master02 | 192.168.229.171 | master |
| 3    | master03 | 192.168.229.172 | master |
| 4    | worker01 | 192.168.229.173 | node   |
| 5    | worker02 | 192.168.229.174 | node   |
| 6    | master01 | 192.168.229.100 | vip    |



| 序号 | 主机名   | 功能                | 备注             |
| ---- | -------- | ------------------- | ---------------- |
| 1    | master01 | haproxy、keepalived | keepalived主节点 |
| 2    | master02 | haproxy、keepalived | keepalived从节点 |



## 3.3 主机配置

### 3.3.1 主机名配置

由于本次使用5台主机完成kubernetes集群部署，其中3台为master节点,名称为master01、master02、master03;其中2台为worker节点，名称分别为：worker01及worker02

~~~powershell
master节点,名称为master01
# hostnamectl set-hostname master01
~~~



~~~powershell
master节点,名称为master02
# hostnamectl set-hostname master02
~~~



~~~powershell
master节点,名称为master03
# hostnamectl set-hostname master03
~~~



~~~powershell
worker1节点,名称为worker01
# hostnamectl set-hostname worker01
~~~



~~~powershell
worker2节点,名称为worker02
# hostnamectl set-hostname worker02
~~~



### 3.3.2 主机IP地址配置

vi /etc/NetworkManager/system-connections/ens33.nmconnection

~~~powershell
master01节点IP地址为：192.168.229.170/24

[ipv4]
address1=192.168.229.170/24,192.168.229.2
dns=114.114.114.114
method=manual

~~~

```
nmcli c reload                       
nmcli c up ens33                    
```



~~~powershell
master02节点IP地址为：192.168.229.171/24


[ipv4]
address1=192.168.229.171/24,192.168.229.2
dns=114.114.114.114
method=manual
~~~



~~~powershell
master03节点IP地址为：192.168.229.172/24

[ipv4]
address1=192.168.229.172/24,192.168.229.2
dns=114.114.114.114
method=manual
~~~



~~~powershell
worker01节点IP地址为：192.168.229.173/24

[ipv4]
address1=192.168.229.173/24,192.168.229.2
dns=114.114.114.114
method=manual
~~~



~~~powershell
worker02节点IP地址为：192.168.229.174/24

[ipv4]
address1=192.168.229.174/24,192.168.229.2
dns=114.114.114.114
method=manual
~~~



### 3.3.3 主机名与IP地址解析



> 所有集群主机均需要进行配置。



~~~powershell
# cat /etc/hosts
......
192.168.229.170 master01
192.168.229.171 master02
192.168.229.172 master03
192.168.229.173 worker01
192.168.229.174 worker02
~~~



### 3.3.4 防火墙配置



> 所有主机均需要操作。



~~~powershell
关闭现有防火墙firewalld
systemctl disable firewalld
systemctl stop firewalld
firewall-cmd --state
not running
~~~



### 3.3.5 SELINUX配置



> 所有主机均需要操作。修改SELinux配置需要重启操作系统。



~~~powershell
sed -ri 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
~~~



### 3.3.6 时间同步配置



>所有主机均需要操作。最小化安装系统需要安装chronyd软件。



~~~powershell
#启用chronyd服务
systemctl enable chronyd

#重启chronyd服务
systemctl restart chronyd

#查看chronyd服务状态
systemctl status chronyd

~~~

```
vi /etc/chrony.conf

pool ntp1.aliyun.com iburst
pool ntp2.aliyun.com iburst
pool ntp3.aliyun.com iburst
pool ntp4.aliyun.com iburst
pool ntp5.aliyun.com iburst
pool ntp6.aliyun.com iburst

systemctl restart chronyd
```



### 3.3.7 多机互信

> 在master节点上生成证书，复制到其它节点即可。复制完成后，可以相互测试登录。



~~~powershell
# ssh-keygen
~~~



~~~powershell
# cd /root/.ssh
[root@master01 .ssh]# ls
id_rsa  id_rsa.pub  known_hosts
[root@master01 .ssh]# cp id_rsa.pub authorized_keys
~~~



~~~powershell
# for i in 171 172 173 174; do scp -r /root/.ssh 192.168.229.$i:/root/; done
~~~





### 3.3.8 升级操作系统内核

> 所有主机均需要操作。

内核不用升级

~~~powershell
[root@worker01 ~]# uname -r
5.14.0-437.el9.x86_64
~~~



### 3.3.9 配置内核转发及网桥过滤

>所有主机均需要操作。



~~~powershell
添加网桥过滤及内核转发配置文件
# cat /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
vm.swappiness = 0
~~~



~~~powershell
加载br_netfilter模块
# modprobe br_netfilter
~~~



~~~powershell
查看是否加载
# lsmod | grep br_netfilter
br_netfilter           22256  0
bridge                151336  1 br_netfilter
~~~

开机加载：

```
vi /etc/modules-load.d/net.conf

br_netfilter
bridge
```



~~~powershell
加载网桥过滤及内核转发配置文件
sysctl --system
~~~



### 3.3.10 安装ipset及ipvsadm

> 所有主机均需要操作。主要用于实现service转发。



~~~powershell
安装ipset及ipvsadm
# yum -y install ipset ipvsadm
~~~



~~~powershell
vi /etc/modules-load.d/ipvs.conf

ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
~~~





### 3.3.11 关闭SWAP分区

> 修改完成后需要重启操作系统，如不重启，可临时关闭，命令为swapoff -a



~~~powershell
永远关闭swap分区，需要重启操作系统
# cat /etc/fstab
......

# /dev/mapper/centos-swap swap                    swap    defaults        0 0

在上一行中行首添加#
~~~



## 3.4 Containerd安装

### 3.4.1  Containerd部署文件获取

![image-20230404104037517](D:/k8s云原生架构师/04k8s核心实战-v3/04集群部署/03_kubeadm_单master节点k8s集群/01-笔记/如何基于Ubuntu 24.04部署原生K8S 1.30.0集群.assets/image-20230404104037517.png)



![image-20230404104105753](D:/k8s云原生架构师/04k8s核心实战-v3/04集群部署/03_kubeadm_单master节点k8s集群/01-笔记/如何基于Ubuntu 24.04部署原生K8S 1.30.0集群.assets/image-20230404104105753.png)









~~~powershell
下载指定版本containerd
# wget https://github.com/containerd/containerd/releases/download/v1.7.16/cri-containerd-cni-1.7.16-linux-amd64.tar.gz
~~~



~~~powershell
解压安装
# tar vxf cri-containerd-cni-1.7.16-linux-amd64.tar.gz  -C /
~~~



### 3.4.2 Containerd配置文件生成并修改



~~~powershell
创建配置文件目录
# mkdir /etc/containerd
~~~



~~~powershell
生成配置文件
# containerd config default > /etc/containerd/config.toml
~~~



~~~powershell
修改第67行
# vim /etc/containerd/config.toml

sandbox_image = "registry.k8s.io/pause:3.9" 由3.8修改为3.9
~~~



~~~powershell
如果使用阿里云容器镜像仓库，也可以修改为：
sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.9" 由3.8修改为3.9
~~~



~~~powershell
修改第139行
# vim /etc/containerd/config.toml

SystemdCgroup = true 由false修改为true
~~~

**注意不要搞成systemd_cgroup = false了，改了这个会有问题**



### 3.4.3 Containerd启动及开机自启动

~~~powershell
设置开机自启动并现在启动
# systemctl enable --now containerd
~~~



~~~powershell
验证其版本
# containerd --version
~~~



# 四、HAProxy及Keepalived部署

## 4.1 HAProxy及keepalived安装

~~~powershell
[root@master01 ~]# yum -y install haproxy keepalived
~~~



~~~powershell
[root@master02 ~]# yum -y install haproxy keepalived
~~~





## 4.2 HAProxy配置及启动



~~~powershell
[root@master01 ~]# vim /etc/haproxy/haproxy.cfg
[root@master01 ~]# cat /etc/haproxy/haproxy.cfg
#---------------------------------------------------------------------
# Example configuration for a possible web application.  See the
# full configuration options online.
#
#
#---------------------------------------------------------------------

#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
  maxconn  2000
  ulimit-n  16384
  log  127.0.0.1 local0 err
  stats timeout 30s

defaults
  log global
  mode  http
  option  httplog
  timeout connect 5000
  timeout client  50000
  timeout server  50000
  timeout http-request 15s
  timeout http-keep-alive 15s

frontend monitor-in
  bind *:33305
  mode http
  option httplog
  monitor-uri /monitor

frontend k8s-master
  bind 0.0.0.0:16443
  bind 127.0.0.1:16443
  mode tcp
  option tcplog
  tcp-request inspect-delay 5s
  default_backend k8s-master

backend k8s-master
  mode tcp
  option tcplog
  option tcp-check
  balance roundrobin
  default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
  server master01   192.168.229.170:6443  check
  server master02   192.168.229.171:6443  check
  server master03   192.168.229.172:6443  check
~~~



~~~powershell
[root@master01 ~]# systemctl enable haproxy;systemctl start haproxy

[root@master01 ~]# systemctl status haproxy
~~~



![image-20220119174750138](kubeadm部署高可用kubernetes集群 1.21.0.assets/image-20220119174750138.png)





~~~powershell
[root@master01 ~]# scp /etc/haproxy/haproxy.cfg master02:/etc/haproxy/haproxy.cfg
~~~



~~~powershell
[root@master02 ~]# systemctl enable haproxy;systemctl start haproxy

[root@master02 ~]# systemctl status haproxy
~~~



![image-20220119175023889](kubeadm部署高可用kubernetes集群 1.21.0.assets/image-20220119175023889.png)



## 4.3 Keepalived配置及启动



~~~powershell
[root@master01 ~]# vim /etc/keepalived/keepalived.conf
[root@master01 ~]# cat /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
script_user root
    enable_script_security
}
vrrp_script chk_apiserver {
    script "/etc/keepalived/check_apiserver.sh" #此脚本需要多独定义，并要调用。
    interval 5
    weight -5
    fall 2
rise 1
}
vrrp_instance VI_1 {
    state MASTER
    interface ens33 # 修改为正在使用的网卡
    mcast_src_ip 192.168.229.170 #为本master主机对应的IP地址
    virtual_router_id 51
    priority 101
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass abc123
    }
    virtual_ipaddress {
        192.168.229.100 #为VIP地址
    }
    track_script {
       chk_apiserver # 执行上面检查apiserver脚本
    }
}
~~~



~~~powershell
[root@master01 ~]# vim /etc/keepalived/check_apiserver.sh
[root@master01 ~]# cat /etc/keepalived/check_apiserver.sh
#!/bin/bash

err=0
for k in $(seq 1 3)
do
    check_code=$(pgrep haproxy)
    if [[ $check_code == "" ]]; then
        err=$(expr $err + 1)
        sleep 1
        continue
    else
        err=0
        break
    fi
done

if [[ $err != "0" ]]; then
    echo "systemctl stop keepalived"
    /usr/bin/systemctl stop keepalived
    exit 1
else
    exit 0
fi
~~~



~~~powershell
[root@master01 ~]# chmod +x /etc/keepalived/check_apiserver.sh
~~~



~~~powershell
scp /etc/keepalived/keepalived.conf master02:/etc/keepalived/

scp /etc/keepalived/check_apiserver.sh master02:/etc/keepalived/
~~~



~~~powershell
[root@master02 ~]# vim /etc/keepalived/keepalived.conf
[root@master02 ~]# cat /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
script_user root
    enable_script_security
}
vrrp_script chk_apiserver {
    script "/etc/keepalived/check_apiserver.sh" #此脚本需要多独定义，并要调用。
    interval 5
    weight -5
    fall 2
rise 1
}
vrrp_instance VI_1 {
    state BACKUP
    interface ens33 # 修改为正在使用的网卡
    mcast_src_ip 192.168.229.171 #为本master主机对应的IP地址
    virtual_router_id 51
    priority 99 # 修改为99
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass abc123
    }
    virtual_ipaddress {
        192.168.10.100 #为VIP地址
    }
    track_script {
       chk_apiserver # 执行上面检查apiserver脚本
    }
}
~~~



~~~powershell
[root@master01 ~]# systemctl enable keepalived;systemctl start keepalived
~~~



~~~powershell
[root@master02 ~]# systemctl enable keepalived;systemctl start keepalived
~~~



## 4.4 验证高可用集群可用性



~~~powershell
[root@master01 ~]# ip a s ens33
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:50:f9:5f brd ff:ff:ff:ff:ff:ff
    inet 192.168.10.11/24 brd 192.168.10.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    inet 192.168.10.100/32 scope global ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::adf4:a8bc:a1c:a9f7/64 scope link tentative noprefixroute dadfailed
       valid_lft forever preferred_lft forever
    inet6 fe80::2b33:40ed:9311:8812/64 scope link tentative noprefixroute dadfailed
       valid_lft forever preferred_lft forever
    inet6 fe80::8508:20d8:7240:32b2/64 scope link tentative noprefixroute dadfailed
       valid_lft forever preferred_lft forever
~~~





~~~powershell
[root@master01 ~]# ss -anput | grep ":16443"
tcp    LISTEN     0      2000   127.0.0.1:16443                 *:*                   users:(("haproxy",pid=2983,fd=6))
tcp    LISTEN     0      2000      *:16443                 *:*                   users:(("haproxy",pid=2983,fd=5))
~~~



~~~powershell
[root@master02 ~]# ss -anput | grep ":16443"
tcp    LISTEN     0      2000   127.0.0.1:16443                 *:*                   users:(("haproxy",pid=2974,fd=6))
tcp    LISTEN     0      2000      *:16443                 *:*                   users:(("haproxy",pid=2974,fd=5))
~~~





# 五、kubernetes 1.30.0  集群部署

## 5.1 集群软件版本说明

|          | kubeadm                | kubelet                                       | kubectl                |
| -------- | ---------------------- | --------------------------------------------- | ---------------------- |
| 版本     | 1.30.0                 | 1.30.0                                        | 1.30.0                 |
| 安装位置 | 集群所有主机           | 集群所有主机                                  | 集群所有主机           |
| 作用     | 初始化集群、管理集群等 | 用于接收api-server指令，对pod生命周期进行管理 | 集群应用命令行管理工具 |



## 5.2 kubernetes YUM源准备

> 在/etc/yum.repos.d/目录中创建k8s.repo文件，把下面内容复制进去即可。

### 5.2.1 阿里云YUM源



~~~powershell
[root@k8s-master1 ~]# vi /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.30/rpm/
enabled=1
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.30/rpm/repodata/repomd.xml.key
~~~



## 5.3 集群软件安装



~~~powershell
查看指定版本
# yum list kubeadm.x86_64 --showduplicates | sort -r
# yum list kubelet.x86_64 --showduplicates | sort -r
# yum list kubectl.x86_64 --showduplicates | sort -r
~~~





~~~powershell
安装指定版本
# yum -y install --setopt=obsoletes=0 kubeadm-1.30.0  kubelet-1.30.0 kubectl-1.30.0
~~~





## 5.4 配置kubelet

>为了实现docker使用的cgroupdriver与kubelet使用的cgroup的一致性，建议修改如下文件内容。



~~~powershell
# vim /etc/sysconfig/kubelet
KUBELET_EXTRA_ARGS="--cgroup-driver=systemd"
~~~



~~~powershell
设置kubelet为开机自启动即可，由于没有生成配置文件，集群初始化后自动启动
# systemctl enable kubelet
~~~



## 5.5 集群镜像准备



~~~powershell
root@k8s-master01:~# kubeadm config images list

registry.k8s.io/kube-apiserver:v1.30.0
registry.k8s.io/kube-controller-manager:v1.30.0
registry.k8s.io/kube-scheduler:v1.30.0
registry.k8s.io/kube-proxy:v1.30.0
registry.k8s.io/coredns/coredns:v1.11.1
registry.k8s.io/pause:3.9
registry.k8s.io/etcd:3.5.12-0
~~~



~~~powershell
使用阿里云容器镜像仓库
# kubeadm config images list --image-repository registry.aliyuncs.com/google_containers
registry.aliyuncs.com/google_containers/kube-apiserver:v1.30.0
registry.aliyuncs.com/google_containers/kube-controller-manager:v1.30.0
registry.aliyuncs.com/google_containers/kube-scheduler:v1.30.0
registry.aliyuncs.com/google_containers/kube-proxy:v1.30.0
registry.aliyuncs.com/google_containers/coredns:v1.11.1
registry.aliyuncs.com/google_containers/pause:3.9
registry.aliyuncs.com/google_containers/etcd:3.5.12-0
~~~



~~~powershell
root@k8s-master01:~# kubeadm config images pull

[config/images] Pulled registry.k8s.io/kube-apiserver:v1.30.0
[config/images] Pulled registry.k8s.io/kube-controller-manager:v1.30.0
[config/images] Pulled registry.k8s.io/kube-scheduler:v1.30.0
[config/images] Pulled registry.k8s.io/kube-proxy:v1.30.0
[config/images] Pulled registry.k8s.io/coredns/coredns:v1.11.1
[config/images] Pulled registry.k8s.io/pause:3.9
[config/images] Pulled registry.k8s.io/etcd:3.5.12-0
~~~



~~~powershell
使用阿里云容器镜像仓库
# kubeadm config images pull --image-repository registry.aliyuncs.com/google_containers
[config/images] Pulled registry.aliyuncs.com/google_containers/kube-apiserver:v1.30.0
[config/images] Pulled registry.aliyuncs.com/google_containers/kube-controller-manager:v1.30.0
[config/images] Pulled registry.aliyuncs.com/google_containers/kube-scheduler:v1.30.0
[config/images] Pulled registry.aliyuncs.com/google_containers/kube-proxy:v1.30.0
[config/images] Pulled registry.aliyuncs.com/google_containers/coredns:v1.11.1
[config/images] Pulled registry.aliyuncs.com/google_containers/pause:3.9
[config/images] Pulled registry.aliyuncs.com/google_containers/etcd:3.5.12-0
~~~



## 5.6 集群初始化

kubeadm config print init-defaults > kubeadm-config.yaml



> 使用阿里云容器镜像仓库
>
> imageRepository: registry.aliyuncs.com/google_containerss



```
apiVersion: kubeadm.k8s.io/v1beta3
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: 7t2weq.bjbawausm0jaxury
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 192.168.229.170
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
  imagePullPolicy: IfNotPresent
  name: master01
  taints: null
---
apiServer:
  certSANs:
  - 192.168.229.100
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controlPlaneEndpoint: 192.168.229.100:16443
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: v1.30.0
networking:
  dnsDomain: cluster.local
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.96.0.0/12
scheduler: {}
```



~~~powershell
[root@master01 ~]# kubeadm init --config /root/kubeadm-config.yaml --upload-certs
~~~



~~~powershell
[init] Using Kubernetes version: v1.30.0
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local master01] and IPs [10.96.0.1 192.168.229.170]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [localhost master01] and IPs [192.168.229.170 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [localhost master01] and IPs [192.168.229.170 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "super-admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests"
[kubelet-check] Waiting for a healthy kubelet. This can take up to 4m0s
[kubelet-check] The kubelet is healthy after 502.827682ms
[api-check] Waiting for a healthy API server. This can take up to 4m0s
[api-check] The API server is healthy after 4.501331598s
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Storing the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
[upload-certs] Using certificate key:
3677962eba923f1b5097c8abab045721b75846ae712046c890c2d87c15982f1d
[mark-control-plane] Marking the node master01 as control-plane by adding the labels: [node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node master01 as control-plane by adding the taints [node-role.kubernetes.io/control-plane:NoSchedule]
[bootstrap-token] Using token: abcdef.0123456789abcdef
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.229.100:16443 --token 7t2weq.bjbawausm0jaxury \
        --discovery-token-ca-cert-hash sha256:d5a0f93fb5520543ce1a7df956805ccae9cc50e182d575b9aa4ccf2ab479b4c7 
~~~





## 5.7 集群应用客户端管理集群文件准备

~~~powershell
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
ls /root/.kube/
~~~



~~~powershell
[root@master01 ~]# export KUBECONFIG=/etc/kubernetes/admin.conf
~~~



## 5.8K8S集群网络插件calico部署

> calico访问链接：https://projectcalico.docs.tigera.io/about/about-calico



![image-20230404115348450](D:/k8s云原生架构师/04k8s核心实战-v3/04集群部署/03_kubeadm_单master节点k8s集群/01-笔记/如何基于Ubuntu 24.04部署原生K8S 1.30.0集群.assets/image-20230404115348450.png)







![image-20230907161355282](D:/k8s云原生架构师/04k8s核心实战-v3/04集群部署/03_kubeadm_单master节点k8s集群/01-笔记/如何基于Ubuntu 24.04部署原生K8S 1.30.0集群.assets/image-20230907161355282.png)





~~~powershell
root@k8s-master01:~# kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml
~~~



~~~powershell
输出内容：
namespace/tigera-operator created
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgpfilters.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/caliconodestatuses.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipreservations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/kubecontrollersconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/apiservers.operator.tigera.io created
customresourcedefinition.apiextensions.k8s.io/imagesets.operator.tigera.io created
customresourcedefinition.apiextensions.k8s.io/installations.operator.tigera.io created
customresourcedefinition.apiextensions.k8s.io/tigerastatuses.operator.tigera.io created
serviceaccount/tigera-operator created
clusterrole.rbac.authorization.k8s.io/tigera-operator created
clusterrolebinding.rbac.authorization.k8s.io/tigera-operator created
deployment.apps/tigera-operator created
~~~



~~~powershell
# wget https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/custom-resources.yaml
~~~

删除删不掉的名称空间

```
kubectl get namespace tigera-operator -o json > devtesting.json
打开devtesting.json把finalizers里的内容删掉
kubectl proxy --port 8080
curl -k -H "Content-Type: application/json" -X PUT --data-binary @devtesting.json http://127.0.0.1:8080/api/v1/namespaces/tigera-operator/finalize

kubectl get namespace calico-system -o json > devtesting.json
打开devtesting.json把finalizers里的内容删掉
kubectl proxy --port 8080
curl -k -H "Content-Type: application/json" -X PUT --data-binary @devtesting.json http://127.0.0.1:8080/api/v1/namespaces/calico-system/finalize
```



~~~powershell
# vim custom-resources.yaml

修改镜像仓库地址和cidr


# cat custom-resources.yaml


# This section includes base Calico installation configuration.
# For more information, see: https://projectcalico.docs.tigera.io/master/reference/installation/api#operator.tigera.io/v1.Installation
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  # Configures Calico networking.
  registry: registry.cn-hangzhou.aliyuncs.com/
  imagePath: hxpdocker
  calicoNetwork:
    # Note: The ipPools section cannot be modified post-install.
    ipPools:
    - blockSize: 26
      cidr: 10.244.0.0/16 修改此行内容为初始化时定义的pod network cidr
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()

---

# This section configures the Calico API server.
# For more information, see: https://projectcalico.docs.tigera.io/master/reference/installation/api#operator.tigera.io/v1.APIServer
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}
~~~



~~~powershell
# kubectl create -f custom-resources.yaml

installation.operator.tigera.io/default created
apiserver.operator.tigera.io/default created
~~~



~~~powershell
root@k8s-master01:~# kubectl get pods -n calico-system
NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-76bbb9b96b-rvdrd   1/1     Running   0          15m
calico-node-cp5xf                          1/1     Running   0          15m
calico-node-tv27t                          1/1     Running   0          15m
calico-node-x2c4p                          1/1     Running   0          15m
calico-typha-65c8d59447-ldfbp              1/1     Running   0          14m
calico-typha-65c8d59447-zlcjt              1/1     Running   0          15m
csi-node-driver-2zr9w                      2/2     Running   0          15m
csi-node-driver-5zvzq                      2/2     Running   0          15m
csi-node-driver-bs5b2                      2/2     Running   0          15m
~~~



安装calico客户端：

![image-20240501140402589](kubeadm部署高可用kubernetes集群 1.21.0.assets\image-20240501140402589.png)





## 5.9 集群其它Master节点加入集群

获取certificate-key

```
kubeadm init phase upload-certs --upload-certs
```



~~~powershell
[root@master02 ~]# kubeadm join 192.168.229.100:16443  --token 7t2weq.bjbawausm0jaxury \
        --discovery-token-ca-cert-hash sha256:d5a0f93fb5520543ce1a7df956805ccae9cc50e182d575b9aa4ccf2ab479b4c7    --control-plane --certificate-key 40e1f222c90394cbed97ea5cbeb343cb163c4b164cce9f764e26a950a016490e

~~~



~~~powershell
[root@master03 ~]# kubeadm join 192.168.229.100:16443 --token 7t2weq.bjbawausm0jaxury \
        --discovery-token-ca-cert-hash sha256:d5a0f93fb5520543ce1a7df956805ccae9cc50e182d575b9aa4ccf2ab479b4c7    --control-plane --certificate-key 40e1f222c90394cbed97ea5cbeb343cb163c4b164cce9f764e26a950a016490e
~~~







## 5.10 集群工作节点加入集群

> 因容器镜像下载较慢，可能会导致报错，主要错误为没有准备好cni（集群网络插件），如有网络，请耐心等待即可。



~~~powershell
kubeadm join 192.168.229.100:16443 --token 7t2weq.bjbawausm0jaxury \
        --discovery-token-ca-cert-hash sha256:d5a0f93fb5520543ce1a7df956805ccae9cc50e182d575b9aa4ccf2ab479b4c7 
~~~



~~~powershell
[root@worker02 ~]# kubeadm join 192.168.229.100:16443 --token 7t2weq.bjbawausm0jaxury \
        --discovery-token-ca-cert-hash sha256:d5a0f93fb5520543ce1a7df956805ccae9cc50e182d575b9aa4ccf2ab479b4c7 
~~~





## 5.11 验证集群可用性

~~~powershell
查看所有的节点
[root@master01 ~]# kubectl get nodes
NAME       STATUS   ROLES                  AGE     VERSION
master01   Ready    control-plane,master   13m     v1.21.0
master02   Ready    control-plane,master   2m25s   v1.21.0
master03   Ready    control-plane,master   87s     v1.21.0
worker01   Ready    <none>                 3m13s   v1.21.0
worker02   Ready    <none>                 2m50s   v1.21.0
~~~



~~~powershell
查看集群健康情况,理想状态
[root@master01 ~]# kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-0               Healthy   {"health":"true"}
~~~



~~~powershell
真实情况
# kubectl get cs
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS      MESSAGE                                                                                       ERROR
scheduler            Unhealthy   Get "http://127.0.0.1:10251/healthz": dial tcp 127.0.0.1:10251: connect: connection refused
controller-manager   Unhealthy   Get "http://127.0.0.1:10252/healthz": dial tcp 127.0.0.1:10252: connect: connection refused
etcd-0               Healthy     {"health":"true"}
~~~







~~~powershell
查看kubernetes集群pod运行情况
[root@master01 ~]# kubectl get pods -n kube-system
NAME                               READY   STATUS    RESTARTS   AGE
coredns-558bd4d5db-smp62           1/1     Running   0          13m
coredns-558bd4d5db-zcmp5           1/1     Running   0          13m
etcd-master01                      1/1     Running   0          14m
etcd-master02                      1/1     Running   0          3m10s
etcd-master03                      1/1     Running   0          115s
kube-apiserver-master01            1/1     Running   0          14m
kube-apiserver-master02            1/1     Running   0          3m13s
kube-apiserver-master03            1/1     Running   0          116s
kube-controller-manager-master01   1/1     Running   1          13m
kube-controller-manager-master02   1/1     Running   0          3m13s
kube-controller-manager-master03   1/1     Running   0          116s
kube-proxy-629zl                   1/1     Running   0          2m17s
kube-proxy-85qn8                   1/1     Running   0          3m15s
kube-proxy-fhqzt                   1/1     Running   0          13m
kube-proxy-jdxbd                   1/1     Running   0          3m40s
kube-proxy-ks97x                   1/1     Running   0          4m3s
kube-scheduler-master01            1/1     Running   1          13m
kube-scheduler-master02            1/1     Running   0          3m13s
kube-scheduler-master03            1/1     Running   0          115s

~~~



~~~powershell
再次查看calico-system命名空间中pod运行情况。
[root@master01 ~]# kubectl get pod -n calico-system
NAME                                      READY   STATUS    RESTARTS   AGE
calico-kube-controllers-666bb9949-4z77k   1/1     Running   0          10m
calico-node-b5wjv                         1/1     Running   0          10m
calico-node-d427l                         1/1     Running   0          4m45s
calico-node-jkq7f                         1/1     Running   0          2m59s
calico-node-wtjnm                         1/1     Running   0          4m22s
calico-node-xxh2p                         1/1     Running   0          3m57s
calico-typha-7cd9d6445b-5zcg5             1/1     Running   0          2m54s
calico-typha-7cd9d6445b-b5d4j             1/1     Running   0          10m
calico-typha-7cd9d6445b-z44kp             1/1     Running   1          4m17s
~~~



~~~powershell
在master节点上操作，查看网络节点是否添加
[root@master01 ~]# DATASTORE_TYPE=kubernetes KUBECONFIG=~/.kube/config calicoctl get nodes
NAME
master01
master02
master03
worker01
worker02
~~~

