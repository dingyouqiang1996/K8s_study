# 什么是kubean

Kubean 是**一个基于Kubespray 构建的集群生命周期管理工具**。 简单易用：通过声明式API 实现Kubean 和K8s 集群强劲生命周期管理的部署。 支持离线：每个版本都会发布离线包（os-pkgs、镜像、二进制包）。 你不必担心如何收集所需的资源。



声明式 API：Kubean 的核心在于其简单易用的声明式 API，使得用户可以专注于描述他们想要的结果，而无需关心实现细节。

离线支持：每个版本都包含了完整的离线包，包括系统软件包、镜像和二进制文件，即使在网络受限的环境中也能保证正常运行。

跨平台兼容：支持 AMD 和 ARM 架构，以及包括常见 Linux 发行版和基于鲲鹏的麒麟操作系统的多种环境。

灵活扩展：Kubean 基于 Kubespray，这意味着你可以利用 Kubespray 的自定义功能来构建符合特定需求的集群。

https://kubean-io.github.io/kubean/zh/

# Kubean 的整体架构如下所示：

[![kubean-architecture](images\kubean-archit-new.png)](https://kubean-io.github.io/kubean/zh/assets/images/kubean-archit-new.png)

Kubean 需要运行在一个已存在的 Kubernetes 集群，通过应用 Kubean 提供的标准 CRD 资源和 Kubernetes 内建资源来控制和管理集群的生命周期（安装、卸载、升级、扩容、缩容等）。 Kubean 采用 Kubespray 作为底层技术依赖，一方面简化了集群部署的操作流程，降低了用户的使用门槛。另一方面在 Kubespray 能力基础上增加了集群操作记录、离线版本记录等诸多新特性。



[![kubean-components](images\kubean-components.png)](https://kubean-io.github.io/kubean/zh/assets/images/kubean-components.png)

Kubean 运行着多个控制器，这些控制器跟踪 Kubean CRD 对象的变化，并且与底层集群的 API 服务器进行通信来创建 Kubernetes原生资源对象。由以下四个组件构成：

1. Cluster Controller: 监视 `Cluster Objects`。唯一标识一个集群，拥有集群节点的访问信息、类型信息、部署参数信息，并且关联所有对此集群的操作（`ClusterOperation Objects`）；
2. ClusterOperation Controller: 监视 `ClusterOperation Objects`。当 `ClusterOperation Object` 被创建时，控制器会组装一个 [Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/) 去执行 CRD 对象里定义的操作；
3. Manifest Controller: 监视 `Manifest Objects`。用于记录和维护当前版本的 Kubean 使用和兼容的组件、包及版本；
4. LocalArtifactSet Controller：监视 `LocalArtifactSet Objects`。用于记录离线包支持的组件及版本信息。



# Kubean vs Kubespray

Kubespray 使用 Ansible 作为底层来配置和编排，可以运行在裸金属机、虚拟机、大多数云环境等。它支持众多 Kubernetes 版本和插件，可以完成集群从 0 到 1 的搭建和配置，也包含集群生命周期的维护，使用方式非常灵活。

Kubean 基于 Kubespray，拥有 Kubespray 所有优势。并且 Kubean 引用 Operator 概念以实现完全云原生化，原生以容器方式运行，提供 Helm Chart 包进行快速部署。

Kubespray 仅在参数级别上支持离线，并没有包含一个完成构建离线安装包的过程，所以对于有离线场景需求的使用者来说，直接使用 Kubespray 会变得非常繁琐，这通常会让他们失去耐心。

Kubean 不仅有一套完善的制作离线包的工作流，还适配国产信创环境，简化 Kubespray 的复杂配置，能够对集群生命周期以云原生的方式去管理。

# 安装kubean

```
helm repo add kubean-io https://kubean-io.github.io/kubean-helm-chart/
helm install kubean kubean-io/kubean --create-namespace -n kubean-system


root@k8s-master01:~# kubectl get pod -n kubean-system
NAME                                READY   STATUS    RESTARTS   AGE
kubean-7f7c58f867-fpf5h             1/1     Running   0          80s
kubean-admission-58d49d8687-zccdz   1/1     Running   0          80s
```



# clone仓库

```
apt install git

git clone https://github.com/kubean-io/kubean.git
```



# 安装

## 准备机子：

| 需求 | CPU  | 内存 | 硬盘  | 角色         | 主机名       |
| ---- | ---- | ---- | ----- | ------------ | ------------ |
| 值   | 2C   | 6G   | 100GB | master       | k8s-master01 |
| 值   | 2C   | 6G   | 100GB | master       | k8s-master02 |
| 值   | 2C   | 6G   | 100GB | master       | k8s-master03 |
| 值   | 2C   | 6G   | 100GB | worker(node) | k8s-worker02 |



## 主机IP地址配置



~~~powershell
k8s-master01节点IP地址为：192.168.229.180/24
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
        - 192.168.229.180/24
      routes:
        - to: default
          via: 192.168.229.2
      nameservers:
        addresses: [119.29.29.29,114.114.114.114,8.8.8.8]
EOF
~~~



~~~powershell
# netplan apply
~~~



~~~powershell
k8s-master02节点IP地址为：192.168.229.181/24
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
        - 192.168.229.181/24
      routes:
        - to: default
          via: 192.168.229.2
      nameservers:
        addresses: [119.29.29.29,114.114.114.114,8.8.8.8]
EOF
~~~



~~~powershell
# netplan apply
~~~





~~~powershell
k8s-master03节点IP地址为：192.168.229.182/24
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
        - 192.168.229.182/24
      routes:
        - to: default
          via: 192.168.229.2
      nameservers:
        addresses: [119.29.29.29,114.114.114.114,8.8.8.8]
EOF
~~~



~~~powershell
# netplan apply
~~~



~~~powershell
k8s-worker01节点IP地址为：192.168.229.183/24
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
        - 192.168.229.183/24
      routes:
        - to: default
          via: 192.168.229.2
      nameservers:
        addresses: [119.29.29.29,114.114.114.114,8.8.8.8]
EOF
~~~



~~~powershell
# netplan apply
~~~



## 准备crd文件



```
kubean/examples/install/2.mirror
```



```
root@k8s-master01:~/kubean/examples/install/2.mirror# cat Cluster.yml 
# Copyright 2023 Authors of kubean-io
# SPDX-License-Identifier: Apache-2.0

apiVersion: kubean.io/v1alpha1
kind: Cluster
metadata:
  name: cluster1-online
  labels:
    clusterName: cluster1-online
spec:
  hostsConfRef:
    namespace: kubean-system
    name: online-hosts-conf
  varsConfRef:
    namespace: kubean-system
    name: online-vars-conf
```



```
root@k8s-master01:~/kubean/examples/install/2.mirror# cat ClusterOperation.yml 

cat > ClusterOperation.yml <<EOF
# Copyright 2023 Authors of kubean-io
# SPDX-License-Identifier: Apache-2.0

apiVersion: kubean.io/v1alpha1
kind: ClusterOperation
metadata:
  name: cluster1-online-install-ops
spec:
  cluster: cluster1-online
  image: ghcr.m.daocloud.io/kubean-io/spray-job:v0.15.3 # Please replace <TAG> with the specified version, such as v0.4.9
  actionType: playbook
  action: cluster.yml
  preHook:
    - actionType: playbook
      action: ping.yml
    - actionType: playbook
      action: disable-firewalld.yml
  postHook:
    - actionType: playbook
      action: kubeconfig.yml
    - actionType: playbook
      action: cluster-info.yml
EOF
```



```
root@k8s-master01:~/kubean/examples/install/2.mirror# cat HostsConfCM.yml 

cat > HostsConfCM.yml <<EOF
# Copyright 2023 Authors of kubean-io
# SPDX-License-Identifier: Apache-2.0

apiVersion: v1
kind: ConfigMap
metadata:
  name: online-hosts-conf
  namespace: kubean-system
data:
  hosts.yml: |
    all:
      hosts:
        node1:
          ip: 192.168.229.180
          access_ip: 192.168.229.180
          ansible_host: 192.168.229.180
          ansible_connection: ssh
          ansible_user: root
          ansible_password: mark
        node2:
          ip: 192.168.229.181
          access_ip: 192.168.229.181
          ansible_host: 192.168.229.181
          ansible_connection: ssh
          ansible_user: root
          ansible_password: mark
        node3:
          ip: 192.168.229.182
          access_ip: 192.168.229.182
          ansible_host: 192.168.229.182
          ansible_connection: ssh
          ansible_user: root
          ansible_password: mark
        node4:
          ip: 192.168.229.183
          access_ip: 192.168.229.183
          ansible_host: 192.168.229.183
          ansible_connection: ssh
          ansible_user: root
          ansible_password: mark
      children:
        kube_control_plane:
          hosts:
            node1:
            node2:
            node3:
        kube_node:
          hosts:
            node1:
            node2:
            node3:
            node4:
        etcd:
          hosts:
            node1:
            node2:
            node3:
        k8s_cluster:
          children:
            kube_control_plane:
            kube_node:
        calico_rr:
          hosts: {}
EOF
```



```
root@k8s-master01:~/kubean/examples/install/2.mirror# cat VarsConfCM.yml 
# Copyright 2023 Authors of kubean-io
# SPDX-License-Identifier: Apache-2.0

apiVersion: v1
kind: ConfigMap
metadata:
  name: online-vars-conf
  namespace: kubean-system
data:
  group_vars.yml: |
    container_manager: containerd
    k8s_image_pull_policy: IfNotPresent
    kube_network_plugin: calico
    kube_network_plugin_multus: false
    kube_proxy_mode: iptables
    enable_nodelocaldns: false
    etcd_deployment_type: kubeadm
    override_system_hostname: false
    ntp_enabled: true

    download_run_once: true
    download_container: false
    download_force_cache: true
    download_localhost: true

    additional_sysctl:
    - { name: kernel.pid_max, value: 4194304 }

    calico_cni_name: calico
    calico_felix_premetheusmetricsenabled: true
    calico_feature_detect_override: "ChecksumOffloadBroken=true" # FIX https://github.com/kubernetes-sigs/kubespray/pull/9261

    ##https://github.com/kubernetes-sigs/kubespray/blob/master/docs/mirror.md
    gcr_image_repo: "gcr.m.daocloud.io"
    kube_image_repo: "k8s.m.daocloud.io"
    docker_image_repo: "docker.m.daocloud.io"
    quay_image_repo: "quay.m.daocloud.io"
    github_image_repo: "ghcr.m.daocloud.io"
    files_repo: "https://files.m.daocloud.io"

    kubeadm_download_url: "{{ files_repo }}/dl.k8s.io/release/{{ kubeadm_version }}/bin/linux/{{ image_arch }}/kubeadm"
    kubectl_download_url: "{{ files_repo }}/dl.k8s.io/release/{{ kube_version }}/bin/linux/{{ image_arch }}/kubectl"
    kubelet_download_url: "{{ files_repo }}/dl.k8s.io/release/{{ kube_version }}/bin/linux/{{ image_arch }}/kubelet"
    cni_download_url: "{{ files_repo }}/github.com/containernetworking/plugins/releases/download/{{ cni_version }}/cni-plugins-linux-{{ image_arch }}-{{ cni_version }}.tgz"
    crictl_download_url: "{{ files_repo }}/github.com/kubernetes-sigs/cri-tools/releases/download/{{ crictl_version }}/crictl-{{ crictl_version }}-{{ ansible_system | lower }}-{{ image_arch }}.tar.gz"
    etcd_download_url: "{{ files_repo }}/github.com/etcd-io/etcd/releases/download/{{ etcd_version }}/etcd-{{ etcd_version }}-linux-{{ image_arch }}.tar.gz"
    calicoctl_download_url: "{{ files_repo }}/github.com/projectcalico/calico/releases/download/{{ calico_ctl_version }}/calicoctl-linux-{{ image_arch }}"
    calicoctl_alternate_download_url: "{{ files_repo }}/github.com/projectcalico/calicoctl/releases/download/{{ calico_ctl_version }}/calicoctl-linux-{{ image_arch }}"
    calico_crds_download_url: "{{ files_repo }}/github.com/projectcalico/calico/archive/{{ calico_version }}.tar.gz"
    helm_download_url: "{{ files_repo }}/get.helm.sh/helm-{{ helm_version }}-linux-{{ image_arch }}.tar.gz"
    crun_download_url: "{{ files_repo }}/github.com/containers/crun/releases/download/{{ crun_version }}/crun-{{ crun_version }}-linux-{{ image_arch }}"
    kata_containers_download_url: "{{ files_repo }}/github.com/kata-containers/kata-containers/releases/download/{{ kata_containers_version }}/kata-static-{{ kata_containers_version }}-{{ ansible_architecture }}.tar.xz"
    runc_download_url: "{{ files_repo }}/github.com/opencontainers/runc/releases/download/{{ runc_version }}/runc.{{ image_arch }}"
    containerd_download_url: "{{ files_repo }}/github.com/containerd/containerd/releases/download/v{{ containerd_version }}/containerd-{{ containerd_version }}-linux-{{ image_arch }}.tar.gz"
    nerdctl_download_url: "{{ files_repo }}/github.com/containerd/nerdctl/releases/download/v{{ nerdctl_version }}/nerdctl-{{ nerdctl_version }}-{{ ansible_system | lower }}-{{ image_arch }}.tar.gz"
    cri_dockerd_download_url: "{{ files_repo }}/github.com/Mirantis/cri-dockerd/releases/download/v{{ cri_dockerd_version }}/cri-dockerd-{{ cri_dockerd_version }}.{{ image_arch }}.tgz"
```



```
kubectl apply -f examples/install/2.mirror
```

# 删除节点

```
cat > Cluster.yml <<EOF 
apiVersion: kubean.io/v1alpha1
kind: Cluster
metadata:
  name: cluster1-online
  labels:
    clusterName: cluster1-online
spec:
  hostsConfRef:
    namespace: kubean-system
    name: online-hosts-conf
  varsConfRef:
    namespace: kubean-system
    name: online-vars-conf
EOF
```



```
root@k8s-master01:~/kubean/examples/scale/2.delWorkNode# cat ClusterOperation.yml 

cat > ClusterOperation.yml <<EOF
---
apiVersion: kubean.io/v1alpha1
kind: ClusterOperation
metadata:
  name: cluster-mini-dwn-ops
spec:
  cluster: cluster1-online
  image: ghcr.m.daocloud.io/kubean-io/spray-job:v0.15.3 # Please replace <TAG> with the specified version, such as v0.4.9
  actionType: playbook
  action: remove-node.yml
  extraArgs: -e node=node4
EOF
```



```
root@k8s-master01:~/kubean/examples/scale/2.delWorkNode# cat HostsConfCM.yml 
# Copyright 2023 Authors of kubean-io
# SPDX-License-Identifier: Apache-2.0

cat > HostsConfCM.yml <<EOF 
apiVersion: v1
kind: ConfigMap
metadata:
  name: online-hosts-conf
  namespace: kubean-system
data:
  hosts.yml: |
    all:
      hosts:
        node1:
          ip: 192.168.229.180
          access_ip: 192.168.229.180
          ansible_host: 192.168.229.180
          ansible_connection: ssh
          ansible_user: root
          ansible_password: mark
        node2:
          ip: 192.168.229.181
          access_ip: 192.168.229.181
          ansible_host: 192.168.229.181
          ansible_connection: ssh
          ansible_user: root
          ansible_password: mark
        node3:
          ip: 192.168.229.182
          access_ip: 192.168.229.182
          ansible_host: 192.168.229.182
          ansible_connection: ssh
          ansible_user: root
          ansible_password: mark
        node4:
          ip: 192.168.229.183
          access_ip: 192.168.229.183
          ansible_host: 192.168.229.183
          ansible_connection: ssh
          ansible_user: root
          ansible_password: mark
      children:
        kube_control_plane:
          hosts:
            node1:
            node2:
            node3:
        kube_node:
          hosts:
            node1:
            node2:
            node3:
            node4:
        etcd:
          hosts:
            node1:
            node2:
            node3:
        k8s_cluster:
          children:
            kube_control_plane:
            kube_node:
        calico_rr:
          hosts: {}
EOF
```





# 添加节点

```
cat > Cluster.yml <<EOF 
apiVersion: kubean.io/v1alpha1
kind: Cluster
metadata:
  name: cluster1-online
  labels:
    clusterName: cluster1-online
spec:
  hostsConfRef:
    namespace: kubean-system
    name: online-hosts-conf
  varsConfRef:
    namespace: kubean-system
    name: online-vars-conf
EOF
```



```
root@k8s-master01:~/kubean/examples/scale/1.addWorkNode# cat ClusterOperation.yml 

cat > ClusterOperation.yml  <<EOF
---
apiVersion: kubean.io/v1alpha1
kind: ClusterOperation
metadata:
  name: cluster-mini-awn-ops
spec:
  cluster: cluster1-online
  image: ghcr.m.daocloud.io/kubean-io/spray-job:v0.15.3 # Please replace <TAG> with the specified version, such as v0.4.9
  actionType: playbook
  action: scale.yml
  extraArgs: --limit=node4
EOF
```



```
root@k8s-master01:~/kubean/examples/scale/1.addWorkNode# cat HostsConfCM.yml 
cat > HostsConfCM.yml <<EOF
# Copyright 2023 Authors of kubean-io
# SPDX-License-Identifier: Apache-2.0

apiVersion: v1
kind: ConfigMap
metadata:
  name: online-hosts-conf
  namespace: kubean-system
data:
  hosts.yml: |
    all:
      hosts:
        node1:
          ip: 192.168.229.180
          access_ip: 192.168.229.180
          ansible_host: 192.168.229.180
          ansible_connection: ssh
          ansible_user: root
          ansible_password: mark
        node2:
          ip: 192.168.229.181
          access_ip: 192.168.229.181
          ansible_host: 192.168.229.181
          ansible_connection: ssh
          ansible_user: root
          ansible_password: mark
        node3:
          ip: 192.168.229.182
          access_ip: 192.168.229.182
          ansible_host: 192.168.229.182
          ansible_connection: ssh
          ansible_user: root
          ansible_password: mark
        node4:
          ip: 192.168.229.183
          access_ip: 192.168.229.183
          ansible_host: 192.168.229.183
          ansible_connection: ssh
          ansible_user: root
          ansible_password: mark
      children:
        kube_control_plane:
          hosts:
            node1:
            node2:
            node3:
        kube_node:
          hosts:
            node1:
            node2:
            node3:
            node4:
        etcd:
          hosts:
            node1:
            node2:
            node3:
        k8s_cluster:
          children:
            kube_control_plane:
            kube_node:
        calico_rr:
          hosts: {}
EOF
```



# 升级

```
root@k8s-master01:~/kubean/examples/upgrade# cat ClusterOperation.yml 

cat > ClusterOperation.yml <<EOF
---
apiVersion: kubean.io/v1alpha1
kind: ClusterOperation
metadata:
  name: cluster-mini-upgrade-ops
spec:
  cluster: cluster1-online
  image: ghcr.m.daocloud.io/kubean-io/spray-job:v0.15.3 # Please replace <TAG> with the specified version, such as v0.4.9
  actionType: playbook
  action: upgrade-cluster.yml
EOF
```



```
root@k8s-master01:~/kubean/examples/upgrade# cat VarsConfCM.yml 

cat > VarsConfCM.yml <<EOF 
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: online-vars-conf
  namespace: kubean-system
data:
  group_vars.yml: |
    kube_version: v1.28.1
    container_manager: containerd
    k8s_image_pull_policy: IfNotPresent
    kube_network_plugin: calico
    kube_network_plugin_multus: false
    kube_proxy_mode: iptables
    enable_nodelocaldns: false
    etcd_deployment_type: kubeadm
    override_system_hostname: false
    ntp_enabled: true

    download_run_once: true
    download_container: false
    download_force_cache: true
    download_localhost: true

    additional_sysctl:
    - { name: kernel.pid_max, value: 4194304 }

    calico_cni_name: calico
    calico_felix_premetheusmetricsenabled: true
    calico_feature_detect_override: "ChecksumOffloadBroken=true" # FIX https://github.com/kubernetes-sigs/kubespray/pull/9261

    ##https://github.com/kubernetes-sigs/kubespray/blob/master/docs/mirror.md
    gcr_image_repo: "gcr.m.daocloud.io"
    kube_image_repo: "k8s.m.daocloud.io"
    docker_image_repo: "docker.m.daocloud.io"
    quay_image_repo: "quay.m.daocloud.io"
    github_image_repo: "ghcr.m.daocloud.io"
    files_repo: "https://files.m.daocloud.io"

    kubeadm_download_url: "{{ files_repo }}/dl.k8s.io/release/{{ kubeadm_version }}/bin/linux/{{ image_arch }}/kubeadm"
    kubectl_download_url: "{{ files_repo }}/dl.k8s.io/release/{{ kube_version }}/bin/linux/{{ image_arch }}/kubectl"
    kubelet_download_url: "{{ files_repo }}/dl.k8s.io/release/{{ kube_version }}/bin/linux/{{ image_arch }}/kubelet"
    cni_download_url: "{{ files_repo }}/github.com/containernetworking/plugins/releases/download/{{ cni_version }}/cni-plugins-linux-{{ image_arch }}-{{ cni_version }}.tgz"
    crictl_download_url: "{{ files_repo }}/github.com/kubernetes-sigs/cri-tools/releases/download/{{ crictl_version }}/crictl-{{ crictl_version }}-{{ ansible_system | lower }}-{{ image_arch }}.tar.gz"
    etcd_download_url: "{{ files_repo }}/github.com/etcd-io/etcd/releases/download/{{ etcd_version }}/etcd-{{ etcd_version }}-linux-{{ image_arch }}.tar.gz"
    calicoctl_download_url: "{{ files_repo }}/github.com/projectcalico/calico/releases/download/{{ calico_ctl_version }}/calicoctl-linux-{{ image_arch }}"
    calicoctl_alternate_download_url: "{{ files_repo }}/github.com/projectcalico/calicoctl/releases/download/{{ calico_ctl_version }}/calicoctl-linux-{{ image_arch }}"
    calico_crds_download_url: "{{ files_repo }}/github.com/projectcalico/calico/archive/{{ calico_version }}.tar.gz"
    helm_download_url: "{{ files_repo }}/get.helm.sh/helm-{{ helm_version }}-linux-{{ image_arch }}.tar.gz"
    crun_download_url: "{{ files_repo }}/github.com/containers/crun/releases/download/{{ crun_version }}/crun-{{ crun_version }}-linux-{{ image_arch }}"
    kata_containers_download_url: "{{ files_repo }}/github.com/kata-containers/kata-containers/releases/download/{{ kata_containers_version }}/kata-static-{{ kata_containers_version }}-{{ ansible_architecture }}.tar.xz"
    runc_download_url: "{{ files_repo }}/github.com/opencontainers/runc/releases/download/{{ runc_version }}/runc.{{ image_arch }}"
    containerd_download_url: "{{ files_repo }}/github.com/containerd/containerd/releases/download/v{{ containerd_version }}/containerd-{{ containerd_version }}-linux-{{ image_arch }}.tar.gz"
    nerdctl_download_url: "{{ files_repo }}/github.com/containerd/nerdctl/releases/download/v{{ nerdctl_version }}/nerdctl-{{ nerdctl_version }}-{{ ansible_system | lower }}-{{ image_arch }}.tar.gz"
    cri_dockerd_download_url: "{{ files_repo }}/github.com/Mirantis/cri-dockerd/releases/download/v{{ cri_dockerd_version }}/cri-dockerd-{{ cri_dockerd_version }}.{{ image_arch }}.tgz"
EOF
```



# 清理

```
root@k8s-master01:~/kubean/examples/uninstall# cat ClusterOperation.yml 

cat >ClusterOperation.yml <<EOF
---
apiVersion: kubean.io/v1alpha1
kind: ClusterOperation
metadata:
  name: cluster-mini-uninstall-ops
spec:
  cluster: cluster1-online
  image: ghcr.m.daocloud.io/kubean-io/spray-job:v0.15.3 # Please replace <TAG> with the specified version, such as v0.4.9
  actionType: playbook
  action: reset.yml
EOF
```

