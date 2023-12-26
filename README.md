### 系统环境
1. ubuntu20.04
2. ubuntu22.04

### 准备工作
1. 网络环境(所有节点)
```bash
root@master1:~/k8s_kubeadm_install# ping pkgs.k8s.io
PING redirect.k8s.io (34.107.204.206) 56(84) bytes of data.
64 bytes from 206.204.107.34.bc.googleusercontent.com (34.107.204.206): icmp_seq=1 ttl=128 time=159 ms
64 bytes from 206.204.107.34.bc.googleusercontent.com (34.107.204.206): icmp_seq=2 ttl=128 time=159 ms

root@master1:~/k8s_kubeadm_install# ping registry.aliyuncs.com
PING registry.aliyuncs.com (120.55.105.209) 56(84) bytes of data.
64 bytes from 120.55.105.209 (120.55.105.209): icmp_seq=1 ttl=128 time=30.8 ms
64 bytes from 120.55.105.209 (120.55.105.209): icmp_seq=2 ttl=128 time=30.2 ms
```

2. 更换系统apt源(有必要的话,所有节点)  
```bash
https://help.mirrors.cernet.edu.cn/ubuntu/


apt update
```

3. 安装ansible(操作节点)
```bash
# 在操作节点上安装 
apt install ansible
mkdir /etc/ansible
# 复制整个k8s_kubeadm_install到管理机
cd k8s_kubeadm_install
# 修改hosts节点名,分组不能修改(请注意: 主机名，ansible节点，/etc/hosts要保持一致)
vim files/ansible/hosts
cp -r files/ansible /etc/
```

4. 手动修改所有节点hostname(所有节点，请注意: 主机名，ansible节点，/etc/hosts要保持一致)
```bash 
sudo hostnamectl set-hostname master1
sudo hostnamectl set-hostname node1
sudo hostnamectl set-hostname node2
...
```

5. 手动修改所有节点/etc/hosts(操作节点，请注意: 主机名，ansible节点，/etc/hosts要保持一致)
```bash 
x.x.x.x master1
y.y.y.y node1
z.z.z.z node2
...
```

6. 配置免密(操作节点)
可以使用以下方式远程
```bash
ssh root@master1 
ssh root@node1
ssh root@node2
...
```

7. 使用ansible统一修改hosts和apt源(前提在操作机上修改后)
```bash
ansible k8s -m copy -a "src=/etc/hosts dest=/etc/hosts"
ansible k8s -m copy -a "src=/etc/apt/sources.list dest=/etc/apt/sources.list"
ansible k8s -m apt -a "update_cache=yes"
```

8. 修改files/vars.yaml(操作节点)  
<strong>指定版本，master节点ip等信息</strong>


### 说明
1. files/calico/custom-resources.yaml && files/calico/tigera-operator.yaml(calico)
```bash
# calico官网地址，查看对应版本
https://docs.tigera.io/calico/latest/about
https://docs.tigera.io/calico/latest/getting-started/kubernetes/requirements#kubernetes-requirements

# 快速部署
https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart

# 需要提前下载对应版本tigera-operator.yaml到files目录，其他操作不用做
https://raw.githubusercontent.com/projectcalico/calico/v3.26.4/manifests/tigera-operator.yaml

# 需要提前下载对应版本custom-resources.yaml到files目录，其他操作不用做
https://raw.githubusercontent.com/projectcalico/calico/v3.26.4/manifests/custom-resources.yaml

# 将下载的文件放至files/calico,并按"文件名_version"格式命名
```

2. files/ingress/deploy.yaml(Ingress)
```bash
# 查看支持版本
https://github.com/kubernetes/ingress-nginx/blob/main/README.md#readme

version=v1.9.4

# 快速部署
https://kubernetes.github.io/ingress-nginx/deploy/#quick-start

# 单独下载deploy.yaml文件
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-${version}/deploy/static/provider/baremetal/deploy.yaml


# deploy.yaml源码文件包下载

https://github.com/kubernetes/ingress-nginx/releases
wget https://github.com/kubernetes/ingress-nginx/archive/refs/tags/controller-${version}.tar.gz
tar -xf controller-${version}.tar.gz

vim ingress-nginx-controller-v1.9.4/deploy/static/provider/baremetal/deploy.yaml

# 需要提前下载对应版本deploy.yaml到files目录，手动修改镜像名称，有三处(v1.9.4,其余版本在这附近)
445        image: registry.cn-hangzhou.aliyuncs.com/google_containers/nginx-ingress-controller:v1.9.4
542        image: registry.cn-hangzhou.aliyuncs.com/google_containers/kube-webhook-certgen:v20231011-8b53cabe0
591        image: registry.cn-hangzhou.aliyuncs.com/google_containers/kube-webhook-certgen:v20231011-8b53cabe0 

把 kind: Deployment 改为 kind: DaemonSet 模式，这样每台node上都有 
ingress-nginx-controller pod 副本。 
使用hostNetwork: true，默认 ingress-nginx 随机提供 nodeport 端口，
开启 hostNetwork 启用80、443端口。
如果不关心 ingressClass 或者很多没有 ingressClass 配置的 ingress 对象， 
添加参数 --watch-ingress-without-class=true

# 位置为v1.9.4，其余版本行号在这附近
391 kind: DaemonSet
409  #strategy:
410    #rollingUpdate:
411      #maxUnavailable: 1
412    #type: RollingUpdate
422       hostNetwork: true
433         - --watch-ingress-without-class=true

# 将下载的文件放至files/ingress,并按"文件名_version"格式命名
```

3. 官方源目前的版本
```bash
root@master1:/etc/apt/sources.list.d# apt-cache madison kubeadm
   kubeadm | 1.29.0-1.1 | https://pkgs.k8s.io/core:/stable:/v1.29/deb  Packages
   kubeadm | 1.28.5-1.1 | https://pkgs.k8s.io/core:/stable:/v1.28/deb  Packages
   kubeadm | 1.28.4-1.1 | https://pkgs.k8s.io/core:/stable:/v1.28/deb  Packages
   kubeadm | 1.28.3-1.1 | https://pkgs.k8s.io/core:/stable:/v1.28/deb  Packages
   kubeadm | 1.28.2-1.1 | https://pkgs.k8s.io/core:/stable:/v1.28/deb  Packages
   kubeadm | 1.28.1-1.1 | https://pkgs.k8s.io/core:/stable:/v1.28/deb  Packages
   kubeadm | 1.28.0-1.1 | https://pkgs.k8s.io/core:/stable:/v1.28/deb  Packages
   kubeadm | 1.27.9-1.1 | https://pkgs.k8s.io/core:/stable:/v1.27/deb  Packages
   kubeadm | 1.27.8-1.1 | https://pkgs.k8s.io/core:/stable:/v1.27/deb  Packages
   kubeadm | 1.27.7-1.1 | https://pkgs.k8s.io/core:/stable:/v1.27/deb  Packages
   kubeadm | 1.27.6-1.1 | https://pkgs.k8s.io/core:/stable:/v1.27/deb  Packages
   kubeadm | 1.27.5-1.1 | https://pkgs.k8s.io/core:/stable:/v1.27/deb  Packages
   kubeadm | 1.27.4-1.1 | https://pkgs.k8s.io/core:/stable:/v1.27/deb  Packages
   kubeadm | 1.27.3-1.1 | https://pkgs.k8s.io/core:/stable:/v1.27/deb  Packages
   kubeadm | 1.27.2-1.1 | https://pkgs.k8s.io/core:/stable:/v1.27/deb  Packages
   kubeadm | 1.27.1-1.1 | https://pkgs.k8s.io/core:/stable:/v1.27/deb  Packages
   kubeadm | 1.27.0-2.1 | https://pkgs.k8s.io/core:/stable:/v1.27/deb  Packages
   kubeadm | 1.26.12-1.1 | https://pkgs.k8s.io/core:/stable:/v1.26/deb  Packages
   kubeadm | 1.26.11-1.1 | https://pkgs.k8s.io/core:/stable:/v1.26/deb  Packages
   kubeadm | 1.26.10-1.1 | https://pkgs.k8s.io/core:/stable:/v1.26/deb  Packages
   kubeadm | 1.26.9-1.1 | https://pkgs.k8s.io/core:/stable:/v1.26/deb  Packages
   kubeadm | 1.26.8-1.1 | https://pkgs.k8s.io/core:/stable:/v1.26/deb  Packages
   kubeadm | 1.26.7-1.1 | https://pkgs.k8s.io/core:/stable:/v1.26/deb  Packages
   kubeadm | 1.26.6-1.1 | https://pkgs.k8s.io/core:/stable:/v1.26/deb  Packages
   kubeadm | 1.26.5-1.1 | https://pkgs.k8s.io/core:/stable:/v1.26/deb  Packages
   kubeadm | 1.26.4-1.1 | https://pkgs.k8s.io/core:/stable:/v1.26/deb  Packages
   kubeadm | 1.26.3-1.1 | https://pkgs.k8s.io/core:/stable:/v1.26/deb  Packages
   kubeadm | 1.26.2-1.1 | https://pkgs.k8s.io/core:/stable:/v1.26/deb  Packages
   kubeadm | 1.26.1-1.1 | https://pkgs.k8s.io/core:/stable:/v1.26/deb  Packages
   kubeadm | 1.26.0-2.1 | https://pkgs.k8s.io/core:/stable:/v1.26/deb  Packages
   kubeadm | 1.25.16-1.1 | https://pkgs.k8s.io/core:/stable:/v1.25/deb  Packages
   kubeadm | 1.25.15-1.1 | https://pkgs.k8s.io/core:/stable:/v1.25/deb  Packages
   kubeadm | 1.25.14-1.1 | https://pkgs.k8s.io/core:/stable:/v1.25/deb  Packages
   kubeadm | 1.25.13-1.1 | https://pkgs.k8s.io/core:/stable:/v1.25/deb  Packages
   kubeadm | 1.25.12-1.1 | https://pkgs.k8s.io/core:/stable:/v1.25/deb  Packages
   kubeadm | 1.25.11-1.1 | https://pkgs.k8s.io/core:/stable:/v1.25/deb  Packages
   kubeadm | 1.25.10-1.1 | https://pkgs.k8s.io/core:/stable:/v1.25/deb  Packages
   kubeadm | 1.25.9-1.1 | https://pkgs.k8s.io/core:/stable:/v1.25/deb  Packages
   kubeadm | 1.25.8-1.1 | https://pkgs.k8s.io/core:/stable:/v1.25/deb  Packages
   kubeadm | 1.25.7-1.1 | https://pkgs.k8s.io/core:/stable:/v1.25/deb  Packages
   kubeadm | 1.25.6-1.1 | https://pkgs.k8s.io/core:/stable:/v1.25/deb  Packages
   kubeadm | 1.25.5-1.1 | https://pkgs.k8s.io/core:/stable:/v1.25/deb  Packages
   kubeadm | 1.25.4-1.1 | https://pkgs.k8s.io/core:/stable:/v1.25/deb  Packages
   kubeadm | 1.25.3-1.1 | https://pkgs.k8s.io/core:/stable:/v1.25/deb  Packages
   kubeadm | 1.25.2-1.1 | https://pkgs.k8s.io/core:/stable:/v1.25/deb  Packages
   kubeadm | 1.25.1-1.1 | https://pkgs.k8s.io/core:/stable:/v1.25/deb  Packages
   kubeadm | 1.25.0-2.1 | https://pkgs.k8s.io/core:/stable:/v1.25/deb  Packages
```

### Run
<strong>必须确认vars.yaml变量是否修改</strong>control_plane_endpoint需要与hosts文件格式一致

```bash
# 默认只安装集群基础功能
ansible-playbook playbooks/main.yaml
# 选装其他插件,harbor需要修改ansible/hosts中分组和/etc/hosts解析
ansible-playbook playbooks/main.yaml -t [dashboard | harbor | metrics | prometheus]
```
