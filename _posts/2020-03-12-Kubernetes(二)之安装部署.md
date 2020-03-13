---
layout:     post
title:      Kubernetes(二)之安装部署
subtitle:   Kubernetes 安装部署
date:       2020-03-13
author:     Frederick
header-img: img/dushu.jpg
catalog: true
tags:
    - Kubernetes
---


# Kubernetes 安装部署
## 1.环境准备
### 1.1 部署机器准备

|  机器IP   |主机名  |角色|系统版本|备注|
|  ----  | ----  | ----| ---- | ---- |
| 192.168.3.120  | kube-master | master  | CentOS  7.4.1708    | 内存2G   |
| 192.168.3.122  | kube-node01 |  node  |  CentOS   7.4.1708   |  内存2G |
| 192.168.3.123  | kube-node02 |  node  |  CentOS   7.4.1708  | 内存2G   |

### 1.2 基础配置安装(每台机器都需执行)

#### 1.2.1 配置节点信息

- **修改对应节点主机hostname**

        hostnamectl set-hostname kube-master

- **配置DNS**

        cat <<EOF >>/etc/hosts
        192.168.3.120 kube-master
        192.168.3.123 kube-node01
        192.168.3.122 kube-node02
        EOF

#### 1.2.2 配置国内yum源

    mkdir /etc/yum.repos.d/bak && mv /etc/yum.repos.d/*.repo /etc/yum.repos.d/bak
    
    wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
    
    yum makecache
    
    yum -y update

#### 1.2.3 关闭防火墙、selinux和swap

- **1 关闭防火墙**

        systemctl stop firewalld & systemctl disable firewalld
- **2 关闭selinux**

        setenforce 0
        sed -i 's/^SELINUX=enforcing$/SELINUX=disabled/' /etc/selinux/config
- **3 关闭swap**

        swapoff -a
        yes | cp /etc/fstab /etc/fstab_bak
        cat /etc/fstab_bak |grep -v swap > /etc/fstab
        echo vm.swappiness=0 >>/etc/sysctl.conf
        sysctl -p
- **4 设置路由**

        yum install -y bridge-utils.x86_64
        
        cat << EOF >  /etc/sysctl.d/k8s.conf
        net.bridge.bridge-nf-call-ip6tables = 1
        net.bridge.bridge-nf-call-iptables = 1
        EOF

        sysctl --system  # 重新加载所有配置文件

#### 1.2.4 安装docker

- **配置docker源**

        yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
        yum makecache
- **安装对应版本docker**

        本次安装笔者选用:V18.09.9
        yum -y install docker-ce-18.09.9

- **安装版本查看如图所示**

    ![docker_version.jpg](https://upload-images.jianshu.io/upload_images/17904159-ef16886398457ffd.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- **启动docker服务并设置服务开机自启动**

        systemctl start docker & systemctl enable docker

- **修改docker cgroup驱动，与k8s一致**

        # 修改docker cgroup驱动：native.cgroupdriver=systemd
        cat > /etc/docker/daemon.json <<EOF
        {
        "exec-opts": ["native.cgroupdriver=systemd"],
        "log-driver": "json-file",
        "log-opts": {
            "max-size": "100m"
        },
        "storage-driver": "overlay2",
        "storage-opts": [
            "overlay2.override_kernel_check=true"
        ]
        }
        EOF

        # 重启使配置生效
        systemctl restart docker  
#### 1.2.4 k8s组件

- **配置k8s yum源**

        cat <<EOF > /etc/yum.repos.d/kubernetes.repo
        [kubernetes]
        name=Kubernetes
        baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
        enabled=1
        gpgcheck=0
        repo_gpgcheck=0
        gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
            http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
        EOF
        #更新yum
        yum makecache

- **master节点安装kubelet kubeadm kubectl**

        #指定版本安装V1-.17.3
        yum install -y kubelet-1.17.3 kubeadm-1.17.3 kubectl-1.17.3 --disableexcludes=kubernetes
- **安装结果如图所示**
![k8sversion.jpg](https://upload-images.jianshu.io/upload_images/17904159-27ee25ccef4b200a.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- **设置开机自启动kubelet**

        systemctl enable --now kubelet 

## 2.部署集群

### 2.1 master节点安装

#### 2.1.1 master节点k8s集群初始化

- **安装相应版本的k8s:V1.17.3**
    
        kubeadm init --kubernetes-version=1.17.3 --apiserver-advertise-address=192.168.3.120 --image-repository registry.aliyuncs.com/google_containers --service-cidr=192.1.0.0/16 --pod-network-cidr=192.244.0.0/16

- **安装结果如图所示：**
![1584073519(1).jpg](https://upload-images.jianshu.io/upload_images/17904159-fcc89e9d2054dd6d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- **记录安装终端末尾显示的内容，此内容需要在其它节点加入k8s集群时执行。**

        kubeadm join 192.168.3.120:6443 --token 76d8po.jgo5pniao8pomjwc \
            --discovery-token-ca-cert-hash sha256:69f401a7aabcc841b5dd857a66b753a6df4133f280bd7560903f2433144a99ec
- **配置kubectl工具**

        #此文件用来连接集群
        mkdir -p $HOME/.kube
        cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
        chown $(id -u):$(id -g) $HOME/.kube/config

- **设置master节点为工作节点，可调度**

        kubectl taint nodes --all node-role.kubernetes.io/master-
- **master节点部署flannel网络**

        kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
- **查看节点状态**

#### 2.1.2 node节点加入

- **执行加入集群命令**

        kubeadm join 192.168.3.120:6443 --token 76d8po.jgo5pniao8pomjwc \
            --discovery-token-ca-cert-hash sha256:69f401a7aabcc841b5dd857a66b753a6df4133f280bd7560903f2433144a99ec
- **执行结果如图所示**
![node_join.jpg](https://upload-images.jianshu.io/upload_images/17904159-6931e2d38cc70c98.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 3.集群状态查看
### 3.1 集群节点状态如图所示

![1584081461(1).jpg](https://upload-images.jianshu.io/upload_images/17904159-ab25997231ce2d94.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

注：状态为Ready表示节点健康，并且正常加入。

**博客著作权归本作者所有，任何形式的转载都请联系作者获得授权并注明出处。**