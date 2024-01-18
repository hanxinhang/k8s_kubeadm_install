# Ansible使用kubeadm方式一键安装k8s
项目地址  
https://github.com/AYYQ127/k8s_kubeadm_install

## 示例

### 系统环境
ubuntu20.04

### 集群规划
主机名可以不用改强制修改为这样，只需要主机名，/etc/hosts和/etc/ansible/hosts都一致即可
主机名 | IP | 用途 
------------|------------|------------
master1  | 192.168.152.200 | 集群主节点1
master2  | 192.168.152.201 | 集群主节点2
node1    | 192.168.152.210 | 工作节点1
node2    | 192.168.152.211 | 工作节点2

### 检查网络环境
四个节点都检查一遍，确保网络没有问题，涉及到后面拉取镜像
```bash
ada@master1:~$ ping pkgs.k8s.io
PING redirect.k8s.io (34.107.204.206) 56(84) bytes of data.
64 bytes from 206.204.107.34.bc.googleusercontent.com (34.107.204.206): icmp_seq=1 ttl=128 time=163 ms

ada@master1:~$ ping registry.aliyuncs.com
PING registry.aliyuncs.com (120.55.105.209) 56(84) bytes of data.
64 bytes from 120.55.105.209 (120.55.105.209): icmp_seq=1 ttl=128 time=30.6 ms
```

### 手动修改所有节点hostname(请注意: 主机名，ansible节点，/etc/hosts要保持一致)
```bash 
ada@master1:~$ sudo hostnamectl set-hostname master1
ada@master2:~$ sudo hostnamectl set-hostname master2
ada@node1:~$ sudo hostnamectl set-hostname node1
ada@node2:~$ sudo hostnamectl set-hostname node2
```

### 手动修改主节点1/etc/hosts(请注意: 主机名，ansible节点，/etc/hosts要保持一致)
```bash 
root@master1:~$ cat /etc/hosts
127.0.0.1 localhost
127.0.1.1 base

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters


192.168.152.200 master1
192.168.152.201 master2
192.168.152.210 node1
192.168.152.211 node2
```

### 配置免密(主节点1)
生成密钥
```bash
ada@master1:~$ sudo su -
root@master1:~# ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /root/.ssh/id_rsa
Your public key has been saved in /root/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:ZYvD8SqDla9v10b843N/xMMWWS750HEF6m3HbXVAScs root@master1
The key's randomart image is:
+---[RSA 3072]----+
|             o=oo|
|             o.+o|
|        . o . E+B|
|       o * o .++*|
|      o S o.. +=*|
|     o . o  o. *+|
|    . o o  o ....|
|       +. . o + o|
|      .o.. . ..++|
+----[SHA256]-----+
```

暂时开启root密码登录 (四个节点都要操作)
```bash
root@master1:~# echo "PermitRootLogin yes" >> /etc/ssh/sshd_config
root@master1:~# systemctl restart sshd.service
root@master1:~# passwd root
New password:
Retype new password:
passwd: password updated successfully

root@master2:~# echo "PermitRootLogin yes" >> /etc/ssh/sshd_config
root@master2:~# systemctl restart sshd.service
root@master2:~# passwd root
New password:
Retype new password:
passwd: password updated successfully

root@node1:~# echo "PermitRootLogin yes" >> /etc/ssh/sshd_config
root@node1:~# systemctl restart sshd.service
root@node1:~# passwd root
New password:
Retype new password:
passwd: password updated successfully

root@node2:~# echo "PermitRootLogin yes" >> /etc/ssh/sshd_config
root@node2:~# systemctl restart sshd.service
root@node2:~# passwd root
New password:
Retype new password:
passwd: password updated successfully

```

复制密钥到各个节点，包括自己 
循环四次依次输入yes和密码
```bash
root@master1:~# for i in {master1,master2,node1,node2}
> do
> ssh-copy-id root@$i
> done
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/root/.ssh/id_rsa.pub"
The authenticity of host 'master1 (192.168.152.200)' can't be established.
ECDSA key fingerprint is SHA256:1QncUYX+qzfiSSNgIiU7NQtEBZEuv6+sHOwb7gGdseY.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
root@master1's password:

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'root@master1'"
and check to make sure that only the key(s) you wanted were added.
...其他三次省略
```
确认可以使用以下方式远程到四台机
```bash
root@master1:~# ssh root@master1
root@master1:~# ssh root@master2
root@master1:~# ssh root@node1
root@master1:~# ssh root@node2

```

再关闭root密码登录(所有节点都要操作)
```bash
root@master1:~# sed -i '$d' /etc/ssh/sshd_config
root@master1:~# systemctl restart sshd.service

root@master2:~# sed -i '$d' /etc/ssh/sshd_config
root@master2:~# systemctl restart sshd.service

root@node1:~# sed -i '$d' /etc/ssh/sshd_config
root@node1:~# systemctl restart sshd.service

root@node2:~# sed -i '$d' /etc/ssh/sshd_config
root@node2:~# systemctl restart sshd.service
```


### 更换主节点1系统源
https://help.mirrors.cernet.edu.cn/ubuntu/   
选择系统版本20.04  
```bash
ada@master1:~$ sudo su -
root@master1:~# cat <<'EOF' > /etc/apt/sources.list
# 默认注释了源码镜像以提高 apt update 速度，如有需要可自行取消注释
deb https://mirrors.cernet.edu.cn/ubuntu/ focal main restricted universe multiverse
# deb-src https://mirrors.cernet.edu.cn/ubuntu/ focal main restricted universe multiverse
deb https://mirrors.cernet.edu.cn/ubuntu/ focal-updates main restricted universe multiverse
# deb-src https://mirrors.cernet.edu.cn/ubuntu/ focal-updates main restricted universe multiverse
deb https://mirrors.cernet.edu.cn/ubuntu/ focal-backports main restricted universe multiverse
# deb-src https://mirrors.cernet.edu.cn/ubuntu/ focal-backports main restricted universe multiverse

# deb https://mirrors.cernet.edu.cn/ubuntu/ focal-security main restricted universe multiverse
# # deb-src https://mirrors.cernet.edu.cn/ubuntu/ focal-security main restricted universe multiverse

deb http://security.ubuntu.com/ubuntu/ focal-security main restricted universe multiverse
# deb-src http://security.ubuntu.com/ubuntu/ focal-security main restricted universe multiverse

# 预发布软件源，不建议启用
# deb https://mirrors.cernet.edu.cn/ubuntu/ focal-proposed main restricted universe multiverse
# # deb-src https://mirrors.cernet.edu.cn/ubuntu/ focal-proposed main restricted universe multiverse
EOF
root@master1:~# apt update
# 其他节点类似
```

### 安装ansible(在主节点安装)
```bash
root@master1:~# apt install ansible -y
root@master1:~# mkdir -p /etc/ansible/
```

### 复制整个k8s_kubeadm_install到主节点任意位置
https://github.com/AYYQ127/k8s_kubeadm_install
```bash
root@master1:~/k8s_kubeadm_install# tree
.
├── files
│   ├── ansible
│   │   ├── ansible.cfg
│   │   └── hosts
│   ├── calico
│   │   ├── custom-resources_v3.26.4.yaml
│   │   ├── custom-resources_v3.27.0.yaml
│   │   ├── tigera-operator_v3.26.4.yaml
│   │   └── tigera-operator_v3.27.0.yaml
│   ├── ingress
│   │   ├── deploy_v1.9.4.yaml
│   │   └── deploy_v1.9.5.yaml
│   ├── k8s_pkgs
│   │   ├── kubernetes-apt-keyring.gpg
│   │   └── source.list
│   ├── metrics
│   │   └── components.yaml
│   ├── rancher
│   ├── test-ingress.yaml
│   └── vars.yaml
├── How_to_run.md
├── playbooks
│   ├── dashboard_install.yaml
│   ├── harbor_install.yaml
│   ├── main.yaml
│   ├── metrics_server_install.yaml
│   └── prometheus_install.yaml
└── README.md
```

### 在主节点1准备ansible环境
```bash 
root@master1:~/k8s_kubeadm_install# vim files/ansible/hosts
root@master1:~/k8s_kubeadm_install# cat files/ansible/hosts
# 修改hosts节点名,分组不能修改,只加/etc/hosts中对应主机名

# 执行安装的节点,第一台master
[manage_node]
master1

# 其他主节点在此添加,不要再加manage_node
[other_masters]
master2

# 工作节点在此添加
[nodes]
node1
node2


# ****************以下内容不要修改*****************

# 除了操作节点的所有节点
[except_manage_node:children]
other_masters
nodes

# 所有主节点(请勿修改)
[masters:children]
manage_node
other_masters

# 所有节点分组(请勿修改)
[k8s:children]
manage_node
other_masters
nodes

```
```bash
# 修改hosts节点名,分组不能修改(请注意: 主机名，ansible节点，/etc/hosts要保持一致)
root@master1:~/k8s_kubeadm_install# cp -r files/ansible /etc/
```


### 使用ansible统一修改hosts和apt源
```bash
root@master1:~/k8s_kubeadm_install# ansible k8s -m copy -a "src=/etc/hosts dest=/etc/hosts"
root@master1:~/k8s_kubeadm_install# ansible k8s -m copy -a "src=/etc/apt/sources.list dest=/etc/apt/sources.list"
root@master1:~/k8s_kubeadm_install# ansible k8s -m apt -a "update_cache=yes"
```

### 修改files/vars.yaml(主节点1)  
<strong>指定版本，master节点ip等信息</strong>

```bash
root@master1:~/k8s_kubeadm_install# cat files/vars.yaml
# 时间服务器
NTP: 192.168.152.200

# 控制面板主机名，不需要修改，固定为[manage_node]
control_plane_endpoint: master1

# kubeadm init使用，大版本号，不需要后缀
kubernetes_version: v1.28.4

# 通常官方源的版本后缀为-1.1.阿里云的为-00,详细版本请看README.md
kube_3tools_version: 1.28.4-1.1

# 定义pod的cidr网络，kubeadm init使用
pod_network_cidr: 10.244.0.0/16
# kubeadm init使用
apiserver_advertise_address: 192.168.152.200

# 修改calico使用custom-resources.yaml使用
pod_network: 10.244.0.0

# pause所使用镜像版本，需要替换阿里源，官方源国内无法拉取
sandbox_image: pause:3.9

# 有版本依赖，请参考README.md指引更换版本
calico_version: v3.26.4

# 有版本依赖，请参考README.md指引更换版本
ingress_version: v1.9.4

```


### Run
<strong>必须确认vars.yaml变量是否修改</strong>control_plane_endpoint需要与hosts文件格式一致

安装过程分为三步，第一步会重启所有节点，重启后再次进入主节点1目录运行相同命令(总共执行两次)
```bash
# 默认只安装集群基础功能
ada@master1:~$ sudo su - 
root@master1:~$ cd k8s_kubeadm_install
root@master1:~/k8s_kubeadm_install# ansible-playbook playbooks/main.yaml

PLAY [第一步初始化系统] ***********************************************************************************************************************************************************************************************
...
...

PLAY [第二步安装kubeadmin] ***************************************************************************************************************************************************************************************
...
...
PLAY [第三步初始化集群,添加工作节点] ***************************************************************************************************************************************************************************************
...
PLAY RECAP ****************************************************************************************************************************************************************************************************
localhost                  : ok=26   changed=18   unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
master1                    : ok=20   changed=16   unreachable=0    failed=0    skipped=17   rescued=0    ignored=0
master2                    : ok=20   changed=15   unreachable=0    failed=0    skipped=17   rescued=0    ignored=0
node1                      : ok=20   changed=15   unreachable=0    failed=0    skipped=17   rescued=0    ignored=0
node2                      : ok=20   changed=16   unreachable=0    failed=0    skipped=17   rescued=0    ignored=0

```

```bash
# 等4-5分钟执行以下命令，集群安装完毕
root@master1:~/k8s_kubeadm_install# kubectl get nodes
NAME      STATUS   ROLES           AGE     VERSION
master1   Ready    control-plane   6m39s   v1.28.4
master2   Ready    control-plane   5m3s    v1.28.4
node1     Ready    node            6m16s   v1.28.4
node2     Ready    node            6m13s   v1.28.4
```

```bash
# 选装其他插件,harbor需要修改ansible/hosts中分组和/etc/hosts解析（暂时只支持metrics）
root@master1:~/k8s_kubeadm_install# ansible-playbook playbooks/main.yaml -t [metrics | harbor | dashboard | prometheus]
```