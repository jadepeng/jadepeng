---
title: K8S集群安装
tags: ["jqpeng"]
categories: ["博客","jqpeng"]
date: 2018-11-12 09:18
---
文章作者:jqpeng
原文链接: [K8S集群安装](https://www.cnblogs.com/xiaoqi/p/9944706.html)

主要参考 [https://github.com/opsnull/follow-me-install-kubernetes-cluster](https://github.com/opsnull/follow-me-install-kubernetes-cluster)

## 01.系统初始化和全局变量

### 添加 k8s 和 docker 账户

在每台机器上添加 k8s 账户，可以无密码 sudo：


    $ sudo useradd -m k8s
    $ sudo visudo
    $ sudo grep '%wheel.*NOPASSWD: ALL' /etc/sudoers
    %wheel	ALL=(ALL)	NOPASSWD: ALL
    $ sudo gpasswd -a k8s wheel


在每台机器上添加 docker 账户，将 k8s 账户添加到 docker 组中，同时配置 dockerd 参数：


    $ sudo useradd -m docker
    $ sudo gpasswd -a k8s docker
    $ sudo mkdir -p  /etc/docker/
    $ cat /etc/docker/daemon.json
    {
        "registry-mirrors": ["https://hub-mirror.c.163.com", "https://docker.mirrors.ustc.edu.cn"],
        "max-concurrent-downloads": 20
    }


### 无密码 ssh 登录其它节点

ssh-copy-id root@docker86-18  
 ssh-copy-id root@docker86-21  
 ssh-copy-id root@docker86-91  
 ssh-copy-id root@docker86-9

ssh-copy-id k8s@docker86-155  
 ssh-copy-id k8s@docker86-18  
 ssh-copy-id root@docker86-21  
 ssh-copy-id root@docker86-91  
 ssh-copy-id root@docker86-9


    source ./environment.sh
    for node_ip in ${NODE_IPS[@]}
      do
        echo ">>> ${node_ip}"
        ssh root@${node_ip} "mkdir -p /opt/k8s/bin && chown -R k8s /opt/k8s && mkdir -p /etc/kubernetes/cert &&chown -R k8s /etc/kubernetes && mkdir -p /etc/etcd/cert && chown -R k8s /etc/etcd/cert &&  mkdir -p /var/lib/etcd && chown -R k8s /etc/etcd/cert"
        scp environment.sh k8s@${node_ip}:/opt/k8s/bin/
        ssh k8s@${node_ip} "chmod +x /opt/k8s/bin/*"
      done


### 定义全局变量


    cat <<EOF >environment.sh 
    #!/usr/bin/bash
    
    # 生成 EncryptionConfig 所需的加密 key
    ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
    
    # 最好使用 当前未用的网段 来定义服务网段和 Pod 网段
    
    # 服务网段，部署前路由不可达，部署后集群内路由可达(kube-proxy 和 ipvs 保证)
    SERVICE_CIDR="10.69.0.0/16"
    
    # Pod 网段，建议 /16 段地址，部署前路由不可达，部署后集群内路由可达(flanneld 保证)
    CLUSTER_CIDR="170.22.0.0/16"
    
    # 服务端口范围 (NodePort Range)
    export NODE_PORT_RANGE="10000-40000"
    
    # 集群各机器 IP 数组
    export NODE_IPS=(192.168.86.154 192.168.86.155 192.168.86.156 192.168.86.18 192.168.86.21 192.168.86.91 192.168.86.9)
    
    # etcd节点
    export ETCD_NODE_IPS=(192.168.86.154 192.168.86.155 192.168.86.156)
    
    # 集群各 IP 对应的 主机名数组
    export NODE_NAMES=(docker86-154 docker86-155 docker86-156 docker86-18 docker86-21 docker86-91 docker86-9)
    
    # kube-apiserver 的 VIP（HA 组件 keepalived 发布的 IP）
    export MASTER_VIP=192.168.86.214
    
    # kube-apiserver VIP 地址（HA 组件 haproxy 监听 8443 端口）
    export KUBE_APISERVER="https://${MASTER_VIP}:8443"
    
    # HA 节点，配置 VIP 的网络接口名称
    export VIP_IF="em1"
    
    # etcd 集群服务地址列表
    export ETCD_ENDPOINTS="https://192.168.86.154:2379,https://192.168.86.155:2379,https://192.168.86.156:2379"
    
    # etcd 集群间通信的 IP 和端口
    export ETCD_NODES="docker86-154=https://192.168.86.154:2380,docker86-155=https://192.168.86.155:2380,docker86-156=https://192.168.86.156:2380"
    
    # flanneld 网络配置前缀
    export FLANNEL_ETCD_PREFIX="/kubernetes/network"
    
    # kubernetes 服务 IP (一般是 SERVICE_CIDR 中第一个IP)
    export CLUSTER_KUBERNETES_SVC_IP="10.69.0.1"
    
    # 集群 DNS 服务 IP (从 SERVICE_CIDR 中预分配)
    export CLUSTER_DNS_SVC_IP="10.69.0.2"
    
    # 集群 DNS 域名
    export CLUSTER_DNS_DOMAIN="cluster.local."
    
    # 将二进制目录 /opt/k8s/bin 加到 PATH 中
    export PATH=/opt/k8s/bin:$PATH
    EOF


然后，把全局变量定义脚本拷贝到所有节点的 /opt/k8s/bin 目录：


    source ./environment.sh
    for node_ip in ${NODE_IPS[@]}
      do
        echo ">>> ${node_ip}"
        scp environment.sh k8s@${node_ip}:/opt/k8s/bin/
        ssh k8s@${node_ip} "chmod +x /opt/k8s/bin/*"
      done


### CA证书

配置文件：  
 17520h 2年，最大2年


    cat > ca-config.json <<EOF
    {
      "signing": {
        "default": {
          "expiry": "17520h"
        },
        "profiles": {
          "kubernetes": {
            "usages": [
                "signing",
                "key encipherment",
                "server auth",
                "client auth"
            ],
            "expiry": "87600h"
          }
        }
      }
    }
    EOF


ca证书签名请求


    cat > ca-csr.json <<EOF
    {
      "CN": "kubernetes",
      "key": {
        "algo": "rsa",
        "size": 2048
      },
      "names": [
        {
          "C": "CN",
          "ST": "BeiJing",
          "L": "BeiJing",
          "O": "k8s",
          "OU": "4Paradigm"
        }
      ]
    }
    EOF


- CN：`Common Name`，kube-apiserver 从证书中提取该字段作为请求的**用户名 (User Name)**，浏览器使用该字段验证网站是否合法；
- O：`Organization`，kube-apiserver 从证书中提取该字段作为请求用户所属的**组 (Group)**；
- kube-apiserver 将提取的 User、Group 作为 `RBAC` 授权的用户标识；


#### 生成 CA 证书和私钥


    cfssl gencert -initca ca-csr.json | cfssljson -bare ca
    ls ca*


将生成的 CA 证书、秘钥文件、配置文件拷贝到所有节点的 /etc/kubernetes/cert 目录下：


    source /opt/k8s/bin/environment.sh # 导入 NODE_IPS 环境变量
    for node_ip in ${NODE_IPS[@]}
      do
        echo ">>> ${node_ip}"
        ssh root@${node_ip} "mkdir -p /etc/kubernetes/cert && chown -R k8s /etc/kubernetes"
        scp ca*.pem ca-config.json k8s@${node_ip}:/etc/kubernetes/cert
      done


### 客户端安装

wget [https://dl.k8s.io/v1.12.1/kubernetes-client-linux-amd64.tar.gz](https://dl.k8s.io/v1.12.1/kubernetes-client-linux-amd64.tar.gz)  
 tar -xzvf kubernetes-client-linux-amd64.tar.gz

分发到所有使用 kubectl 的节点：


    source /opt/k8s/bin/environment.sh
    for node_ip in ${NODE_IPS[@]}
      do
        echo ">>> ${node_ip}"
        scp kubernetes/client/bin/kubectl k8s@${node_ip}:/opt/k8s/bin/
        ssh k8s@${node_ip} "chmod +x /opt/k8s/bin/*"
      done


### 创建 admin 证书和私钥

kubectl 与 apiserver https 安全端口通信，apiserver 对提供的证书进行认证和授权。

kubectl 作为集群的管理工具，需要被授予最高权限。这里创建具有最高权限的 admin 证书。

创建证书签名请求：


    cat > admin-csr.json <<EOF
    {
      "CN": "admin",
      "hosts": [],
      "key": {
        "algo": "rsa",
        "size": 2048
      },
      "names": [
        {
          "C": "CN",
          "ST": "BeiJing",
          "L": "BeiJing",
          "O": "system:masters",
          "OU": "4Paradigm"
        }
      ]
    }
    EOF


O 为 system:masters，kube-apiserver 收到该证书后将请求的 Group 设置为 system:masters；  
 预定义的 ClusterRoleBinding cluster-admin 将 Group system:masters 与 Role cluster-admin 绑定，该 Role 授予所有 API的权限；  
 该证书只会被 kubectl 当做 client 证书使用，所以 hosts 字段为空；

### 生成证书和私钥：


    cfssl gencert -ca=/etc/kubernetes/cert/ca.pem \
      -ca-key=/etc/kubernetes/cert/ca-key.pem \
      -config=/etc/kubernetes/cert/ca-config.json \
      -profile=kubernetes admin-csr.json | cfssljson -bare admin
    ls admin*


### 创建 kubeconfig 文件

kubeconfig 为 kubectl 的配置文件，包含访问 apiserver 的所有信息，如 apiserver 地址、CA 证书和自身使用的证书；


    source /opt/k8s/bin/environment.sh
    # 设置集群参数
    kubectl config set-cluster kubernetes \
      --certificate-authority=/etc/kubernetes/cert/ca.pem \
      --embed-certs=true \
      --server=${KUBE_APISERVER} \
      --kubeconfig=kubectl.kubeconfig
    
    # 设置客户端认证参数
    kubectl config set-credentials admin \
      --client-certificate=admin.pem \
      --client-key=admin-key.pem \
      --embed-certs=true \
      --kubeconfig=kubectl.kubeconfig
    
    # 设置上下文参数
    kubectl config set-context kubernetes \
      --cluster=kubernetes \
      --user=admin \
      --kubeconfig=kubectl.kubeconfig
      
    # 设置默认上下文
    kubectl config use-context kubernetes --kubeconfig=kubectl.kubeconfig
    --certificate-authority：验证 kube-apiserver 证书的根证书；
    --client-certificate、--client-key：刚生成的 admin 证书和私钥，连接 kube-apiserver 时使用；
    --embed-certs=true：将 ca.pem 和 admin.pem 证书内容嵌入到生成的 kubectl.kubeconfig 文件中(不加时，写入的是证书文件路径)；


### 分发 kubeconfig 文件

分发到所有使用 kubectl 命令的节点：


    source /opt/k8s/bin/environment.sh
    for node_ip in ${NODE_IPS[@]}
      do
        echo ">>> ${node_ip}"
        ssh k8s@${node_ip} "mkdir -p ~/.kube"
        scp kubectl.kubeconfig k8s@${node_ip}:~/.kube/config
        ssh root@${node_ip} "mkdir -p ~/.kube"
        scp kubectl.kubeconfig root@${node_ip}:~/.kube/config
      done


保存到用户的 ~/.kube/config 文件；

## etcd安装

到 [https://github.com/coreos/etcd/releases](https://github.com/coreos/etcd/releases) 页面下载最新版本的发布包：


    wget https://github.com/etcd-io/etcd/releases/download/v3.3.10/etcd-v3.3.10-linux-amd64.tar.gz
    tar -xvf etcd-v3.3.10-linux-amd64.tar.gz


### 分发二进制文件到集群所有节点：


    source /opt/k8s/bin/environment.sh
    for node_ip in ${NODE_IPS[@]}
      do
        echo ">>> ${node_ip}"
        scp etcd-v3.3.10-linux-amd64/etcd* k8s@${node_ip}:/opt/k8s/bin
        ssh k8s@${node_ip} "chmod +x /opt/k8s/bin/*"
      done


### 创建 etcd 证书和私钥

创建证书签名请求：


    cat > etcd-csr.json <<EOF
    {
      "CN": "etcd",
      "hosts": [
        "127.0.0.1",
        "192.168.86.156",
        "192.168.86.155",
        "192.168.86.154"
      ],
      "key": {
        "algo": "rsa",
        "size": 2048
      },
      "names": [
        {
          "C": "CN",
          "ST": "BeiJing",
          "L": "BeiJing",
          "O": "k8s",
          "OU": "4Paradigm"
        }
      ]
    }
    EOF


### 生成证书和私钥：


    cfssl gencert -ca=/etc/kubernetes/cert/ca.pem \
        -ca-key=/etc/kubernetes/cert/ca-key.pem \
        -config=/etc/kubernetes/cert/ca-config.json \
        -profile=kubernetes etcd-csr.json | cfssljson -bare etcd
    ls etcd*


### 分发生成的证书和私钥到各 etcd 节点：


    source /opt/k8s/bin/environment.sh
    for node_ip in ${NODE_IPS[@]}
      do
        echo ">>> ${node_ip}"
        ssh root@${node_ip} "mkdir -p /etc/etcd/cert && chown -R k8s /etc/etcd/cert"
        scp etcd*.pem k8s@${node_ip}:/etc/etcd/cert/
      done


ETCD\_NODE\_IPS

### 创建 etcd 的 systemd unit 模板文件

cat &gt; etcd.service.template &lt;&lt;EOF  
 [Unit]  
 Description=Etcd Server  
 After=network.target  
 After=network-online.target  
 Wants=network-online.target  
 Documentation=https://github.com/coreos

[Service]  
 User=k8s  
 Type=notify  
 WorkingDirectory=/var/lib/etcd/  
 ExecStart=/opt/k8s/bin/etcd \  
 --data-dir=/var/lib/etcd \  
 --name=##NODE\_NAME## \

--cert-file=/etc/etcd/cert/etcd.pem \  
 --key-file=/etc/etcd/cert/etcd-key.pem \  
 --trusted-ca-file=/etc/kubernetes/cert/ca.pem \  
 --peer-cert-file=/etc/etcd/cert/etcd.pem \  
 --peer-key-file=/etc/etcd/cert/etcd-key.pem \  
 --peer-trusted-ca-file=/etc/kubernetes/cert/ca.pem \  
 --peer-client-cert-auth \  
 --client-cert-auth \  
 --listen-peer-urls=https://##NODE\_IP##:2380 \  
 --initial-advertise-peer-urls=https://##NODE\_IP##:2380 \  
 --listen-client-urls=https://##NODE\_IP##:2379,[http://127.0.0.1:2379](http://127.0.0.1:2379) \  
 --advertise-client-urls=https://##NODE\_IP##:2379 \  
 --initial-cluster-token=etcd-cluster-0 \  
 --initial-cluster=${ETCD\_NODES} \  
 --initial-cluster-state=new  
 Restart=on-failure  
 RestartSec=5  
 LimitNOFILE=65536

[Install]  
 WantedBy=multi-user.target  
 EOF

User：指定以 k8s 账户运行；  
 WorkingDirectory、--data-dir：指定工作目录和数据目录为 /var/lib/etcd，需在启动服务前创建这个目录；  
 --name：指定节点名称，当 --initial-cluster-state 值为 new 时，--name 的参数值必须位于 --initial-cluster 列表中；  
 --cert-file、--key-file：etcd server 与 client 通信时使用的证书和私钥；  
 --trusted-ca-file：签名 client 证书的 CA 证书，用于验证 client 证书；  
 --peer-cert-file、--peer-key-file：etcd 与 peer 通信使用的证书和私钥；  
 --peer-trusted-ca-file：签名 peer 证书的 CA 证书，用于验证 peer 证书；

### 为各节点创建和分发 etcd systemd unit 文件


    source /opt/k8s/bin/environment.sh
    for (( i=0; i < 3; i++ ))
      do
        sed -e "s/##NODE_NAME##/${NODE_NAMES[i]}/" -e "s/##NODE_IP##/${NODE_IPS[i]}/" etcd.service.template > etcd-${NODE_IPS[i]}.service 
      done
    ls *.service


分发生成的 systemd unit 文件：

source /opt/k8s/bin/environment.sh  
 for node\_ip in ${ETCD\_NODE\_IPS[@]}  
 do  
 echo "&gt;&gt;&gt; ${node\_ip}"  
 ssh root@${node\_ip} "mkdir -p /var/lib/etcd && chown -R k8s /var/lib/etcd"  
 scp etcd-${node\_ip}.service root@${node\_ip}:/etc/systemd/system/etcd.service  
 done

### 启动 etcd 服务

source ./environment.sh  
 for node\_ip in ${ETCD\_NODE\_IPS[@]}  
 do  
 echo "&gt;&gt;&gt; ${node\_ip}"  
 ssh root@${node\_ip} "systemctl daemon-reload && systemctl enable etcd && systemctl restart etcd &"  
 done

etcd 进程首次启动时会等待其它节点的 etcd 加入集群，命令 systemctl start etcd 会卡住一段时间，为正常现象。

### 检查启动结果


    source ./environment.sh
    for node_ip in ${ETCD_NODE_IPS[@]}
      do
        echo ">>> ${node_ip}"
        ssh k8s@${node_ip} "systemctl status etcd|grep Active"
      done


确保状态为 active (running)，否则查看日志，确认原因：

$ journalctl -u etcd

### 验证服务状态

部署完 etcd 集群后，在任一 etc 节点上执行如下命令：

source ./environment.sh  
 for node\_ip in ${ETCD\_NODE\_IPS[@]}  
 do  
 echo "&gt;&gt;&gt; ${node\_ip}"  
 ETCDCTL\_API=3 /opt/k8s/bin/etcdctl   
 --endpoints=https://${node\_ip}:2379   
 --cacert=/etc/kubernetes/cert/ca.pem   
 --cert=/etc/etcd/cert/etcd.pem   
 --key=/etc/etcd/cert/etcd-key.pem endpoint health  
 done  
 预期输出：


> > > 192.168.86.154  
> > > [https://192.168.86.154:2379](https://192.168.86.154:2379) is healthy: successfully committed proposal: took = 2.197007ms
> 
> 
> 
> > > 192.168.86.155  
> > > [https://192.168.86.155:2379](https://192.168.86.155:2379) is healthy: successfully committed proposal: took = 2.299328ms
> 
> 
> 
> > > 192.168.86.156  
> > > [https://192.168.86.156:2379](https://192.168.86.156:2379) is healthy: successfully committed proposal: took = 2.014274ms


## 05.部署 flannel 网络

kubernetes 要求集群内各节点(包括 master 节点)能通过 Pod 网段互联互通。flannel 使用 vxlan 技术为各节点创建一个可以互通的 Pod 网络，使用的端口为 UDP 8472，需要开放该端口（如公有云 AWS 等）。

flannel 第一次启动时，从 etcd 获取 Pod 网段信息，为本节点分配一个未使用的 /24 段地址，然后创建 flannel.1（也可能是其它名称，如 flannel1 等） 接口。

flannel 将分配的 Pod 网段信息写入 /run/flannel/docker 文件，docker 后续使用这个文件中的环境变量设置 docker0 网桥。

### 下载和分发 flanneld 二进制文件

到 [https://github.com/coreos/flannel/releases](https://github.com/coreos/flannel/releases) 页面下载最新版本的发布包：


    mkdir flannel
    wget https://github.com/coreos/flannel/releases/download/v0.10.0/flannel-v0.10.0-linux-amd64.tar.gz
    tar -xzvf flannel-v0.10.0-linux-amd64.tar.gz -C flannel


### 创建证书签名请求：


    cat > flanneld-csr.json <<EOF
    {
      "CN": "flanneld",
      "hosts": [],
      "key": {
        "algo": "rsa",
        "size": 2048
      },
      "names": [
        {
          "C": "CN",
          "ST": "BeiJing",
          "L": "BeiJing",
          "O": "k8s",
          "OU": "4Paradigm"
        }
      ]
    }
    EOF


该证书只会被 kubectl 当做 client 证书使用，所以 hosts 字段为空；  
 生成证书和私钥：


    cfssl gencert -ca=/etc/kubernetes/cert/ca.pem \
      -ca-key=/etc/kubernetes/cert/ca-key.pem \
      -config=/etc/kubernetes/cert/ca-config.json \
      -profile=kubernetes flanneld-csr.json | cfssljson -bare flanneld
    ls flanneld*pem


分发 flanneld 二进制文件和flannel 证书、私钥 到集群所有节点：


    source /opt/k8s/bin/environment.sh
    for node_ip in ${NODE_IPS[@]}
      do
        echo ">>> ${node_ip}"
        scp  flannel/{flanneld,mk-docker-opts.sh} k8s@${node_ip}:/opt/k8s/bin/
        ssh k8s@${node_ip} "chmod +x /opt/k8s/bin/*"
        ssh root@${node_ip} "mkdir -p /etc/flanneld/cert && chown -R k8s /etc/flanneld"scp flanneld*.pem k8s@${node_ip}:/etc/flanneld/cert
      done


创建  
 flannel 从 etcd 集群存取网段分配信息，而 etcd 集群启用了双向 x509 证书认证，所以需要为 flanneld 生成证书和私钥。

向 etcd 写入集群 Pod 网段信息  
 注意：本步骤只需执行一次。


    source /opt/k8s/bin/environment.sh
    etcdctl \
      --endpoints=${ETCD_ENDPOINTS} \
      --ca-file=/etc/kubernetes/cert/ca.pem \
      --cert-file=/etc/flanneld/cert/flanneld.pem \
      --key-file=/etc/flanneld/cert/flanneld-key.pem \
      set ${FLANNEL_ETCD_PREFIX}/config '{"Network":"'${CLUSTER_CIDR}'", "SubnetLen": 24, "Backend": {"Type": "vxlan"}}'


flanneld 当前版本 (v0.10.0) 不支持 etcd v3，故使用 etcd v2 API 写入配置 key 和网段数据；  
 写入的 Pod 网段 ${CLUSTER\_CIDR} 必须是 /16 段地址，必须与 kube-controller-manager 的 --cluster-cidr 参数值一致；

### 创建 flanneld 的 systemd unit 文件


    source /opt/k8s/bin/environment.sh
    export IFACE=eno1 # 有的为em1，eth0
    cat > flanneld.service << EOF
    [Unit]
    Description=Flanneld overlay address etcd agent
    After=network.target
    After=network-online.target
    Wants=network-online.target
    After=etcd.service
    Before=docker.service
    
    [Service]
    Type=notify
    ExecStart=/opt/k8s/bin/flanneld \\
      -etcd-cafile=/etc/kubernetes/cert/ca.pem \\
      -etcd-certfile=/etc/flanneld/cert/flanneld.pem \\
      -etcd-keyfile=/etc/flanneld/cert/flanneld-key.pem \\
      -etcd-endpoints=${ETCD_ENDPOINTS} \\
      -etcd-prefix=${FLANNEL_ETCD_PREFIX} \\
      -iface=${IFACE}
    ExecStartPost=/opt/k8s/bin/mk-docker-opts.sh -k DOCKER_NETWORK_OPTIONS -d /run/flannel/docker
    Restart=on-failure
    
    [Install]
    WantedBy=multi-user.target
    RequiredBy=docker.service
    EOF


mk-docker-opts.sh 脚本将分配给 flanneld 的 Pod 子网网段信息写入 /run/flannel/docker 文件，后续 docker 启动时使用这个文件中的环境变量配置 docker0 网桥；  
 flanneld 使用系统缺省路由所在的接口与其它节点通信，对于有多个网络接口（如内网和公网）的节点，可以用 -iface 参数指定通信接口，如上面的 eth0 接口;  
 flanneld 运行时需要 root 权限；  
 完整 unit 见 flanneld.service

注意：  
 有的IFACE=eno1，有的为em1，eth，通过ifconfig查看

### 分发 flanneld systemd unit 文件到所有节点


    source /opt/k8s/bin/environment.sh
    for node_ip in ${NODE_IPS[@]}
      do
        echo ">>> ${node_ip}"
        scp flanneld.service root@${node_ip}:/etc/systemd/system/
      done


### 启动 flanneld 服务


    source /opt/k8s/bin/environment.sh
    for node_ip in ${NODE_IPS[@]}
      do
        echo ">>> ${node_ip}"
        ssh root@${node_ip} "systemctl daemon-reload && systemctl enable flanneld && systemctl restart flanneld"
      done


### 检查启动结果


    source /opt/k8s/bin/environment.sh
    for node_ip in ${NODE_IPS[@]}
      do
        echo ">>> ${node_ip}"
        ssh k8s@${node_ip} "systemctl status flanneld|grep Active"
      done


确保状态为 active (running)，否则查看日志，确认原因：

$ journalctl -u flanneld

### 检查分配给各 flanneld 的 Pod 网段信息

查看集群 Pod 网段(/16)：

source /opt/k8s/bin/environment.sh  
 etcdctl   
 --endpoints=${ETCD\_ENDPOINTS}   
 --ca-file=/etc/kubernetes/cert/ca.pem   
 --cert-file=/etc/flanneld/cert/flanneld.pem   
 --key-file=/etc/flanneld/cert/flanneld-key.pem   
 get ${FLANNEL\_ETCD\_PREFIX}/config  
 输出：

{"Network":"170.22.0.0/16", "SubnetLen": 24, "Backend": {"Type": "vxlan"}}

查看已分配的 Pod 子网段列表(/24):

source /opt/k8s/bin/environment.sh  
 etcdctl   
 --endpoints=${ETCD\_ENDPOINTS}   
 --ca-file=/etc/kubernetes/cert/ca.pem   
 --cert-file=/etc/flanneld/cert/flanneld.pem   
 --key-file=/etc/flanneld/cert/flanneld-key.pem   
 ls ${FLANNEL\_ETCD\_PREFIX}/subnets  
 输出：

/kubernetes/network/subnets/170.22.76.0-24  
 /kubernetes/network/subnets/170.22.84.0-24  
 /kubernetes/network/subnets/170.22.45.0-24  
 /kubernetes/network/subnets/170.22.7.0-24  
 /kubernetes/network/subnets/170.22.12.0-24  
 /kubernetes/network/subnets/170.22.78.0-24  
 /kubernetes/network/subnets/170.22.5.0-24

查看某一 Pod 网段对应的节点 IP 和 flannel 接口地址:

source /opt/k8s/bin/environment.sh  
 etcdctl   
 --endpoints=${ETCD\_ENDPOINTS}   
 --ca-file=/etc/kubernetes/cert/ca.pem   
 --cert-file=/etc/flanneld/cert/flanneld.pem   
 --key-file=/etc/flanneld/cert/flanneld-key.pem   
 get ${FLANNEL\_ETCD\_PREFIX}/subnets/170.22.76.0-24  
 输出：

{"PublicIP":"192.168.86.156","BackendType":"vxlan","BackendData":{"VtepMAC":"6a:aa:ca:8a:ac:ed"}}

验证各节点能通过 Pod 网段互通  
 在各节点上部署 flannel 后，检查是否创建了 flannel 接口(名称可能为 flannel0、flannel.0、flannel.1 等)：

source /opt/k8s/bin/environment.sh  
 for node\_ip in ${NODE\_IPS[@]}  
 do  
 echo "&gt;&gt;&gt; ${node\_ip}"  
 ssh ${node\_ip} "/usr/sbin/ip addr show flannel.1|grep -w inet"  
 done  
 输出：

inet 172.30.81.0/32 scope global flannel.1  
 inet 172.30.29.0/32 scope global flannel.1  
 inet 172.30.39.0/32 scope global flannel.1  
 在各节点上 ping 所有 flannel 接口 IP，确保能通：

source /opt/k8s/bin/environment.sh  
 for node\_ip in ${NODE\_IPS[@]}  
 do  
 echo "&gt;&gt;&gt; ${node\_ip}"  
 ssh ${node\_ip} "ping -c 1 172.30.81.0"  
 ssh ${node\_ip} "ping -c 1 172.30.29.0"  
 ssh ${node\_ip} "ping -c 1 172.30.39.0"  
 done

## 06-0.部署 master 节点

kubernetes master 节点运行如下组件：

kube-apiserver  
 kube-scheduler  
 kube-controller-manager  
 kube-scheduler 和 kube-controller-manager 可以以集群模式运行，通过 leader 选举产生一个工作进程，其它进程处于阻塞模式。

对于 kube-apiserver，可以运行多个实例（本文档是 3 实例），但对其它组件需要提供统一的访问地址，该地址需要高可用。本文档使用 keepalived 和 haproxy 实现 kube-apiserver VIP 高可用和负载均衡。

### 下载最新版本的二进制文件

从 CHANGELOG页面 下载 server tarball 文件（需要翻墙）


    wget https://dl.k8s.io/v1.12.1/kubernetes-server-linux-amd64.tar.gz
    tar -xzvf kubernetes-server-linux-amd64.tar.gz


将二进制文件拷贝到所有 所有节点：


    source /opt/k8s/bin/environment.sh
    for node_ip in ${NODE_IPS[@]}
      do
        echo ">>> ${node_ip}"
        scp kubernetes/server/bin/* k8s@${node_ip}:/opt/k8s/bin/
        ssh k8s@${node_ip} "chmod +x /opt/k8s/bin/*"
      done


如果有老版本运行，先停止：


    systemctl stop kubelet.service 
    systemctl stop kube-controller-manager.service 
    systemctl stop kube-apiserver.service 
    systemctl stop kube-proxy.service 
    systemctl stop kube-scheduler.service
    systemctl stop etcd
    systemctl stop 


## 06-1.部署高可用组件（keepalived+haproxy)

使用 keepalived 和 haproxy 实现 kube-apiserver 高可用的步骤：

- keepalived 提供 kube-apiserver 对外服务的 VIP；
- haproxy 监听 VIP，后端连接所有 kube-apiserver 实例，提供健康检查和负载均衡功能；
- 运行 keepalived 和 haproxy 的节点称为 LB 节点。由于 keepalived 是一主多备运行模式，故至少两个 LB 节点。


本文档复用 master 节点的三台机器，haproxy 监听的端口(8443) 需要与 kube-apiserver 的端口 6443 不同，避免冲突。

keepalived 在运行过程中周期检查本机的 haproxy 进程状态，如果检测到 haproxy 进程异常，则触发重新选主的过程，VIP 将飘移到新选出来的主节点，从而实现 VIP 的高可用。

所有组件（如 kubeclt、apiserver、controller-manager、scheduler 等）都通过 VIP 和 haproxy 监听的 8443 端口访问 kube-apiserver 服务。

### 安装软件包


    source /opt/k8s/bin/environment.sh
    for node_ip in ${NODE_IPS[@]}
      do
        echo ">>> ${node_ip}"
        ssh root@${node_ip} "yum install -y keepalived haproxy"
      done


ubuntu机器，apt-get install

### 配置和下发 haproxy 配置文件

haproxy 配置文件：


    cat > haproxy.cfg <<EOF
     global
         log /dev/log    local0
         log /dev/log    local1 notice
         chroot /var/lib/haproxy
         stats socket /var/run/haproxy-admin.sock mode 660 level admin
         stats timeout 30s
         user haproxy
         group haproxy
         daemon
         nbproc 1
     
     defaults
         log     global
         timeout connect 5000
         timeout client  10m
         timeout server  10m
     
     listen  admin_stats
         bind 0.0.0.0:10080
         mode http
         log 127.0.0.1 local0 err
         stats refresh 30s
         stats uri /status
         stats realm welcome login\ Haproxy
         stats auth admin:123456
         stats hide-version
         stats admin if TRUE
     
     listen kube-master
         bind 0.0.0.0:8443
         mode tcp
         option tcplog
         balance source
         server 192.168.86.154 192.168.86.154:6443 check inter 2000 fall 2 rise 2 weight 1
         server 192.168.86.155 192.168.86.155:6443 check inter 2000 fall 2 rise 2 weight 1
         server 192.168.86.156 192.168.86.156:6443 check inter 2000 fall 2 rise 2 weight 1
    EOF


- haproxy 在 10080 端口输出 status 信息；
- haproxy 监听所有接口的 8443 端口，该端口与环境变量 ${KUBE\_APISERVER} 指定的端口必须一致；
- server 字段列出所有 kube-apiserver 监听的 IP 和端口；


下发 haproxy.cfg 到所有 master 节点：


    source /opt/k8s/bin/environment.sh
    for node_ip in ${ETCD_NODE_IPS[@]}
      do
        echo ">>> ${node_ip}"
        scp haproxy.cfg root@${node_ip}:/etc/haproxy
      done


起 haproxy 服务


    source /opt/k8s/bin/environment.sh
    for node_ip in ${ETCD_NODE_IPS[@]}
      do
        echo ">>> ${node_ip}"
        ssh root@${node_ip} "systemctl restart haproxy"
      done


检查 haproxy 服务状态


    source /opt/k8s/bin/environment.sh
    for node_ip in ${ETCD_NODE_IPS[@]}
      do
        echo ">>> ${node_ip}"
        ssh root@${node_ip} "systemctl status haproxy|grep Active"
      done


确保状态为 active (running)，否则查看日志，确认原因：


> > > 192.168.86.154  
> > >  Active: active (running) since Tue 2018-11-06 10:48:13 CST; 5s ago
> 
> 
> 
> > > 192.168.86.155  
> > >  Active: active (running) since Tue 2018-11-06 10:48:14 CST; 5s ago
> 
> 
> 
> > > 192.168.86.156  
> > >  Active: active (running) since Tue 2018-11-06 10:48:13 CST; 5s ago


journalctl -u haproxy  
 检查 haproxy 是否监听 8443 端口：


    source /opt/k8s/bin/environment.sh
    for node_ip in ${ETCD_NODE_IPS[@]}
      do
        echo ">>> ${node_ip}"
        ssh root@${node_ip} "netstat -lnpt|grep haproxy"
      done


确保输出类似于:

tcp        0      0 0.0.0.0:8443            0.0.0.0:\*               LISTEN      45606/haproxy

### 配置和下发 keepalived 配置文件

keepalived 是一主（master）多备（backup）运行模式，故有两种类型的配置文件。master 配置文件只有一份，backup 配置文件视节点数目而定，对于本文档而言，规划如下：

master: 192.168.86.156  
 backup：192.168.86.155，192.168.86.154

master 配置文件：


    source /opt/k8s/bin/environment.sh
    cat  > keepalived-master.conf <<EOF
    global_defs {
        router_id lb-master-105
    }
    
    vrrp_script check-haproxy {
        script "killall -0 haproxy"
        interval 5
        weight -30
    }
    
    vrrp_instance VI-kube-master {
        state MASTER
        priority 120
        dont_track_primary
        interface ${VIP_IF}
        virtual_router_id 68
        advert_int 3
        track_script {
            check-haproxy
        }
        virtual_ipaddress {
            ${MASTER_VIP}
        }
    }
    EOF


VIP 所在的接口（interface ${VIP\_IF}）为 em1  
 使用 killall -0 haproxy 命令检查所在节点的 haproxy 进程是否正常。如果异常则将权重减少（-30）,从而触发重新选主过程；  
 router\_id、virtual\_router\_id 用于标识属于该 HA 的 keepalived 实例，如果有多套 keepalived HA，则必须各不相同；  
 backup 配置文件：

source /opt/k8s/bin/environment.sh  
 cat  &gt; keepalived-backup.conf &lt;&lt;EOF  
 global\_defs {  
 router\_id lb-backup-105  
 }

vrrp\_script check-haproxy {  
 script "killall -0 haproxy"  
 interval 5  
 weight -30  
 }

vrrp\_instance VI-kube-master {  
 state BACKUP  
 priority 110  
 dont\_track\_primary  
 interface ${VIP\_IF}  
 virtual\_router\_id 68  
 advert\_int 3  
 track\_script {  
 check-haproxy  
 }  
 virtual\_ipaddress {  
 ${MASTER\_VIP}  
 }  
 }  
 EOF

VIP 所在的接口（interface ${VIP\_IF}）为 em1  
 使用 killall -0 haproxy 命令检查所在节点的 haproxy 进程是否正常。如果异常则将权重减少（-30）,从而触发重新选主过程；  
 router\_id、virtual\_router\_id 用于标识属于该 HA 的 keepalived 实例，如果有多套 keepalived HA，则必须各不相同；  
 priority 的值必须小于 master 的值；

### 下发 keepalived 配置文件

下发 master 配置文件：


    scp keepalived-master.conf root@172.27.129.105:/etc/keepalived/keepalived.conf


下发 backup 配置文件：


    scp keepalived-backup.conf root@172.27.129.111:/etc/keepalived/keepalived.conf
    scp keepalived-backup.conf root@172.27.129.112:/etc/keepalived/keepalived.conf


起 keepalived 服务


    source /opt/k8s/bin/environment.sh
    for node_ip in ${ETCD_NODE_IPS[@]}
      do
        echo ">>> ${node_ip}"
        ssh root@${node_ip} "systemctl restart keepalived"
      done


检查 keepalived 服务  
 source /opt/k8s/bin/environment.sh  
 for node\_ip in ${ETCD\_NODE\_IPS[@]}  
 do  
 echo "&gt;&gt;&gt; ${node\_ip}"  
 ssh root@${node\_ip} "systemctl status keepalived|grep Active"  
 done  
 确保状态为 active (running)，否则查看日志（journalctl -u keepalived），确认原因：


> > > 192.168.86.154  
> > >  Active: active (running) since Tue 2018-11-06 10:54:01 CST; 17s ago
> 
> 
> 
> > > 192.168.86.155  
> > >  Active: active (running) since Tue 2018-11-06 10:54:03 CST; 18s ago
> 
> 
> 
> > > 192.168.86.156  
> > >  Active: active (running) since Tue 2018-11-06 10:54:03 CST; 17s ago


查看 VIP 所在的节点，确保可以 ping 通 VIP：

source /opt/k8s/bin/environment.sh  
 for node\_ip in ${NODE\_IPS[@]}  
 do  
 echo "&gt;&gt;&gt; ${node\_ip}"  
 ssh ${node\_ip} "/usr/sbin/ip addr show ${VIP\_IF}"  
 ssh ${node\_ip} "ping -c 1 ${MASTER\_VIP}"  
 done  
 查看 haproxy 状态页面  
 浏览器访问 ${MASTER\_VIP}:10080/status 地址，查看 haproxy 状态页面：

## 06-1.部署 kube-apiserver 组件

使用 keepalived 和 haproxy 部署一个 3 节点高可用 master 集群的步骤，对应的 LB VIP 为环境变量 ${MASTER\_VIP}。

### 创建 kubernetes 证书和私钥


    source /opt/k8s/bin/environment.sh
    cat > kubernetes-csr.json <<EOF
    {
      "CN": "kubernetes",
      "hosts": [
        "127.0.0.1",
        "192.168.86.156",
        "192.168.86.155",
        "192.168.86.154",
        "192.168.86.9",
        "${MASTER_VIP}",
        "${CLUSTER_KUBERNETES_SVC_IP}",
        "kubernetes",
        "kubernetes.default",
        "kubernetes.default.svc",
        "kubernetes.default.svc.local",
        "kubernetes.default.svc.local.com"
      ],
      "key": {
        "algo": "rsa",
        "size": 2048
      },
      "names": [
        {
          "C": "CN",
          "ST": "BeiJing",
          "L": "BeiJing",
          "O": "k8s",
          "OU": "4Paradigm"
        }
      ]
    }
    EOF


- hosts 字段指定授权使用该证书的 IP 或域名列表，这里列出了 VIP 、apiserver 节点 IP、kubernetes 服务 IP 和域名
- 域名最后字符不能是 .(如不能为 kubernetes.default.svc.cluster.local.)，否则解析时失败，提示： x509: cannot parse dnsName "kubernetes.default.svc.cluster.local."；
- 如果使用非 cluster.local 域名，如 opsnull.com，则需要修改域名列表中的最后两个域名为：kubernetes.default.svc.opsnull、kubernetes.default.svc.opsnull.com
- kubernetes 服务 IP 是 apiserver 自动创建的，一般是 --service-cluster-ip-range 参数指定的网段的第一个IP，后续可以通过如下命令获取：kubectl get svc kubernetes


生成证书和私钥：


    cfssl gencert -ca=/etc/kubernetes/cert/ca.pem \
      -ca-key=/etc/kubernetes/cert/ca-key.pem \
      -config=/etc/kubernetes/cert/ca-config.json \
      -profile=kubernetes kubernetes-csr.json | cfssljson -bare kubernetes
    ls kubernetes*pem


将生成的证书和私钥文件拷贝到 master 节点：


    source /opt/k8s/bin/environment.sh
    for node_ip in ${ETCD_NODE_IPS[@]}
      do
        echo ">>> ${node_ip}"
        ssh root@${node_ip} "mkdir -p /etc/kubernetes/cert/ && sudo chown -R k8s /etc/kubernetes/cert/"
        scp kubernetes*.pem k8s@${node_ip}:/etc/kubernetes/cert/
      done


k8s 账户可以读写 /etc/kubernetes/cert/ 目录；

### 创建加密配置文件


    source /opt/k8s/bin/environment.sh
    cat > encryption-config.yaml <<EOF
    kind: EncryptionConfig
    apiVersion: v1
    resources:
      - resources:
          - secrets
        providers:
          - aescbc:
              keys:
                - name: key1
                  secret: ${ENCRYPTION_KEY}
          - identity: {}
    EOF
    
    将加密配置文件拷贝到 master 节点的 /etc/kubernetes 目录下：
    
    source /opt/k8s/bin/environment.sh
    for node_ip in ${ETCD_NODE_IPS[@]}
      do
        echo ">>> ${node_ip}"
        scp encryption-config.yaml root@${node_ip}:/etc/kubernetes/
      done


### 创建 kube-apiserver systemd unit 模板文件


    source /opt/k8s/bin/environment.sh
    cat > kube-apiserver.service.template <<EOF
    [Unit]
    Description=Kubernetes API Server
    Documentation=https://github.com/GoogleCloudPlatform/kubernetes
    After=network.target
    
    [Service]
    ExecStart=/opt/k8s/bin/kube-apiserver \\
      --enable-admission-plugins=Initializers,NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
      --anonymous-auth=false \\
      --experimental-encryption-provider-config=/etc/kubernetes/encryption-config.yaml \\
      --advertise-address=##NODE_IP## \\
      --bind-address=##NODE_IP## \\
      --insecure-port=0 \\
      --authorization-mode=Node,RBAC \\
      --runtime-config=api/all \\
      --enable-bootstrap-token-auth \\
      --service-cluster-ip-range=${SERVICE_CIDR} \\
      --service-node-port-range=${NODE_PORT_RANGE} \\
      --tls-cert-file=/etc/kubernetes/cert/kubernetes.pem \\
      --tls-private-key-file=/etc/kubernetes/cert/kubernetes-key.pem \\
      --client-ca-file=/etc/kubernetes/cert/ca.pem \\
      --kubelet-client-certificate=/etc/kubernetes/cert/kubernetes.pem \\
      --kubelet-client-key=/etc/kubernetes/cert/kubernetes-key.pem \\
      --service-account-key-file=/etc/kubernetes/cert/ca-key.pem \\
      --etcd-cafile=/etc/kubernetes/cert/ca.pem \\
      --etcd-certfile=/etc/kubernetes/cert/kubernetes.pem \\
      --etcd-keyfile=/etc/kubernetes/cert/kubernetes-key.pem \\
      --etcd-servers=${ETCD_ENDPOINTS} \\
      --enable-swagger-ui=true \\
      --allow-privileged=true \\
      --apiserver-count=3 \\
      --audit-log-maxage=30 \\
      --audit-log-maxbackup=3 \\
      --audit-log-maxsize=100 \\
      --audit-log-path=/var/log/kube-apiserver-audit.log \\
      --event-ttl=1h \\
      --alsologtostderr=true \\
      --logtostderr=false \\
      --log-dir=/var/log/kubernetes \\
      --v=2
    Restart=on-failure
    RestartSec=5
    Type=notify
    User=k8s
    LimitNOFILE=65536
    
    [Install]
    WantedBy=multi-user.target
    EOF


- --experimental-encryption-provider-config：启用加密特性；
- --authorization-mode=Node,RBAC： 开启 Node 和 RBAC 授权模式，拒绝未授权的请求；
- --enable-admission-plugins：启用 ServiceAccount 和 NodeRestriction；
- --service-account-key-file：签名 ServiceAccount Token 的公钥文件，kube-controller-manager 的 --service-account-private-key-file 指定私钥文件，两者配对使用；
- --tls-\*-file：指定 apiserver 使用的证书、私钥和 CA 文件。--client-ca-file 用于验证 client (kue-controller-manager、kube-scheduler、kubelet、kube-proxy 等)请求所带的证书；
- --kubelet-client-certificate、--kubelet-client-key：如果指定，则使用 https 访问 kubelet APIs；需要为证书对应的用户(上面 kubernetes\*.pem 证书的用户为 kubernetes) 用户定义 RBAC 规则，否则访问 kubelet API \* 时提示未授权；
- --bind-address： 不能为 127.0.0.1，否则外界不能访问它的安全端口 6443；
- --insecure-port=0：关闭监听非安全端口(8080)；
- --service-cluster-ip-range： 指定 Service Cluster IP 地址段；
- --service-node-port-range： 指定 NodePort 的端口范围；
- --runtime-config=api/all=true： 启用所有版本的 APIs，如 autoscaling/v2alpha1；
- --enable-bootstrap-token-auth：启用 kubelet bootstrap 的 token 认证；
- --apiserver-count=3：指定集群运行模式，多台 kube-apiserver 会通过 leader 选举产生一个工作节点，其它节点处于阻塞状态；
- User=k8s：使用 k8s 账户运行；


### 为各节点创建和分发 kube-apiserver systemd unit 文件

替换模板文件中的变量，为各节点创建 systemd unit 文件：


    source /opt/k8s/bin/environment.sh
    for (( i=0; i < 3; i++ ))
      do
        sed -e "s/##NODE_NAME##/${NODE_NAMES[i]}/" -e "s/##NODE_IP##/${NODE_IPS[i]}/" kube-apiserver.service.template > kube-apiserver-${NODE_IPS[i]}.service 
      done
    ls kube-apiserver*.service


分发生成的 systemd unit 文件


    source /opt/k8s/bin/environment.sh
    for node_ip in ${ETCD_NODE_IPS[@]}
      do
        echo ">>> ${node_ip}"
        ssh root@${node_ip} "mkdir -p /var/log/kubernetes && chown -R k8s /var/log/kubernetes"
        scp kube-apiserver-${node_ip}.service root@${node_ip}:/etc/systemd/system/kube-apiserver.service
      done


### 启动 kube-apiserver 服务

source /opt/k8s/bin/environment.sh  
 for node\_ip in ${ETCD\_NODE\_IPS[@]}  
 do  
 echo "&gt;&gt;&gt; ${node\_ip}"  
 ssh root@${node\_ip} "systemctl daemon-reload && systemctl enable kube-apiserver && systemctl restart kube-apiserver"  
 done

### 检查 kube-apiserver 运行状态


    source /opt/k8s/bin/environment.sh
    for node_ip in ${ETCD_NODE_IPS[@]}
      do
        echo ">>> ${node_ip}"
        ssh root@${node_ip} "systemctl status kube-apiserver |grep 'Active:'"
      done


确保状态为 active (running)，否则到 master 节点查看日志，确认原因：

journalctl -u kube-apiserver

### 打印 kube-apiserver 写入 etcd 的数据

source /opt/k8s/bin/environment.sh  
 ETCDCTL\_API=3 etcdctl   
 --endpoints=${ETCD\_ENDPOINTS}   
 --cacert=/etc/kubernetes/cert/ca.pem   
 --cert=/etc/etcd/cert/etcd.pem   
 --key=/etc/etcd/cert/etcd-key.pem   
 get /registry/ --prefix --keys-only

检查集群信息


    kubectl cluster-info
    kubectl get all --all-namespaces
    kubectl get componentstatuses


检查 kube-apiserver 监听的端口  
 sudo netstat -lnpt|grep kube  
 tcp        0      0 172.27.129.105:6443     0.0.0.0:\*               LISTEN      13075/kube-apiserve

6443: 接收 https 请求的安全端口，对所有请求做认证和授权；  
 由于关闭了非安全端口，故没有监听 8080；

### 授予 kubernetes 证书访问 kubelet API 的权限

在执行 kubectl exec、run、logs 等命令时，apiserver 会转发到 kubelet。这里定义 RBAC 规则，授权 apiserver 调用 kubelet API。


    kubectl create clusterrolebinding kube-apiserver:kubelet-apis --clusterrole=system:kubelet-api-admin --user kubernetes


## 06-3.部署高可用 kube-controller-manager 集群

该集群包含 3 个节点，启动后将通过竞争选举机制产生一个 leader 节点，其它节点为阻塞状态。当 leader 节点不可用后，剩余节点将再次进行选举产生新的 leader 节点，从而保证服务的可用性。

为保证通信安全，本文档先生成 x509 证书和私钥，kube-controller-manager 在如下两种情况下使用该证书：

与 kube-apiserver 的安全端口通信时;  
 在安全端口(https，10252) 输出 prometheus 格式的 metrics；

### 创建 kube-controller-manager 证书和私钥

创建证书签名请求：


    cat > kube-controller-manager-csr.json <<EOF
    {
        "CN": "system:kube-controller-manager",
        "key": {
            "algo": "rsa",
            "size": 2048
        },
        "hosts": [
          "127.0.0.1",
          "192.168.86.156",  "192.168.86.155",  "192.168.86.154"
        ],
        "names": [
          {
            "C": "CN",
            "ST": "BeiJing",
            "L": "BeiJing",
            "O": "system:kube-controller-manager",
            "OU": "4Paradigm"
          }
        ]
    }
    EOF


hosts 列表包含所有 kube-controller-manager 节点 IP；  
 CN 为 system:kube-controller-manager、O 为 system:kube-controller-manager，kubernetes 内置的 ClusterRoleBindings system:kube-controller-manager 赋予 kube-controller-manager 工作所需的权限。  
 生成证书和私钥：


    cfssl gencert -ca=/etc/kubernetes/cert/ca.pem \
      -ca-key=/etc/kubernetes/cert/ca-key.pem \
      -config=/etc/kubernetes/cert/ca-config.json \
      -profile=kubernetes kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager


将生成的证书和私钥分发到所有 master 节点：


    source /opt/k8s/bin/environment.sh
    for node_ip in ${ETCD_NODE_IPS[@]}
      do
        echo ">>> ${node_ip}"
        scp kube-controller-manager*.pem k8s@${node_ip}:/etc/kubernetes/cert/
      done


### 创建和分发 kubeconfig 文件

kubeconfig 文件包含访问 apiserver 的所有信息，如 apiserver 地址、CA 证书和自身使用的证书；


    source /opt/k8s/bin/environment.sh
    kubectl config set-cluster kubernetes \
      --certificate-authority=/etc/kubernetes/cert/ca.pem \
      --embed-certs=true \
      --server=${KUBE_APISERVER} \
      --kubeconfig=kube-controller-manager.kubeconfig
    
    kubectl config set-credentials system:kube-controller-manager \
      --client-certificate=kube-controller-manager.pem \
      --client-key=kube-controller-manager-key.pem \
      --embed-certs=true \
      --kubeconfig=kube-controller-manager.kubeconfig
    
    kubectl config set-context system:kube-controller-manager \
      --cluster=kubernetes \
      --user=system:kube-controller-manager \
      --kubeconfig=kube-controller-manager.kubeconfig
    
    kubectl config use-context system:kube-controller-manager --kubeconfig=kube-controller-manager.kubeconfig


分发 kubeconfig 到所有 master 节点：


    source /opt/k8s/bin/environment.sh
    for node_ip in ${ETCD_NODE_IPS[@]}
      do
        echo ">>> ${node_ip}"
        scp kube-controller-manager.kubeconfig k8s@${node_ip}:/etc/kubernetes/
      done


### 创建和分发 kube-controller-manager systemd unit 文件


    source /opt/k8s/bin/environment.sh
    cat > kube-controller-manager.service <<EOF
    [Unit]
    Description=Kubernetes Controller Manager
    Documentation=https://github.com/GoogleCloudPlatform/kubernetes
    
    [Service]
    ExecStart=/opt/k8s/bin/kube-controller-manager \\
      --port=0 \\
      --secure-port=10252 \\
      --bind-address=127.0.0.1 \\
      --kubeconfig=/etc/kubernetes/kube-controller-manager.kubeconfig \\
      --service-cluster-ip-range=${SERVICE_CIDR} \\
      --cluster-name=kubernetes \\
      --cluster-signing-cert-file=/etc/kubernetes/cert/ca.pem \\
      --cluster-signing-key-file=/etc/kubernetes/cert/ca-key.pem \\
      --experimental-cluster-signing-duration=17520h \\
      --root-ca-file=/etc/kubernetes/cert/ca.pem \\
      --service-account-private-key-file=/etc/kubernetes/cert/ca-key.pem \\
      --leader-elect=true \\
      --feature-gates=RotateKubeletServerCertificate=true \\
      --controllers=*,bootstrapsigner,tokencleaner \\
      --horizontal-pod-autoscaler-use-rest-clients=true \\
      --horizontal-pod-autoscaler-sync-period=10s \\
      --tls-cert-file=/etc/kubernetes/cert/kube-controller-manager.pem \\
      --tls-private-key-file=/etc/kubernetes/cert/kube-controller-manager-key.pem \\
      --use-service-account-credentials=true \\
      --alsologtostderr=true \\
      --logtostderr=false \\
      --log-dir=/var/log/kubernetes \\
      --v=2
    Restart=on
    Restart=on-failure
    RestartSec=5
    User=k8s
    
    [Install]
    WantedBy=multi-user.target
    EOF


- --port=0：关闭监听 http /metrics 的请求，同时 --address 参数无效，--bind-address 参数有效；
- --secure-port=10252、--bind-address=0.0.0.0: 在所有网络接口监听 10252 端口的 https /metrics 请求；
- --kubeconfig：指定 kubeconfig 文件路径，kube-controller-manager 使用它连接和验证 kube-apiserver；
- --cluster-signing-\*-file：签名 TLS Bootstrap 创建的证书；
- --experimental-cluster-signing-duration：指定 TLS Bootstrap 证书的有效期；
- --root-ca-file：放置到容器 ServiceAccount 中的 CA 证书，用来对 kube-apiserver 的证书进行校验；
- --service-account-private-key-file：签名 ServiceAccount 中 Token 的私钥文件，必须和 kube-apiserver 的 --service-account-key-file 指定的公钥文件配对使用；
- --service-cluster-ip-range ：指定 Service Cluster IP 网段，必须和 kube-apiserver 中的同名参数一致；
- --leader-elect=true：集群运行模式，启用选举功能；被选为 leader 的节点负责处理工作，其它节点为阻塞状态；
- --feature-gates=RotateKubeletServerCertificate=true：开启 kublet server 证书的自动更新特性；
- --controllers=\*,bootstrapsigner,tokencleaner：启用的控制器列表，tokencleaner 用于自动清理过期的 Bootstrap token；
- --horizontal-pod-autoscaler-\*：custom metrics 相关参数，支持 autoscaling/v2alpha1；
- --tls-cert-file、--tls-private-key-file：使用 https 输出 metrics 时使用的 Server 证书和秘钥；
- --use-service-account-credentials=true:
- User=k8s：使用 k8s 账户运行；
- kube-controller-manager 不对请求 https metrics 的 Client 证书进行校验，故不需要指定 --tls-ca-file 参数，而且该参数已被淘汰。


分发 systemd unit 文件到所有 master 节点：


    source /opt/k8s/bin/environment.sh
    for node_ip in ${ETCD_NODE_IPS[@]}
      do
        echo ">>> ${node_ip}"
        scp kube-controller-manager.service root@${node_ip}:/etc/systemd/system/
      done


### kube-controller-manager 的权限

ClusteRole: system:kube-controller-manager 的权限很小，只能创建 secret、serviceaccount 等资源对象，各 controller 的权限分散到 ClusterRole system:controller:XXX 中。

需要在 kube-controller-manager 的启动参数中添加 --use-service-account-credentials=true 参数，这样 main controller 会为各 controller 创建对应的 ServiceAccount XXX-controller。

内置的 ClusterRoleBinding system:controller:XXX 将赋予各 XXX-controller ServiceAccount 对应的 ClusterRole system:controller:XXX 权限。

### 启动 kube-controller-manager 服务


    source /opt/k8s/bin/environment.sh
    for node_ip in ${ETCD_NODE_IPS[@]}
      do
        echo ">>> ${node_ip}"
        ssh root@${node_ip} "mkdir -p /var/log/kubernetes && chown -R k8s /var/log/kubernetes"
        ssh root@${node_ip} "systemctl daemon-reload && systemctl enable kube-controller-manager && systemctl restart kube-controller-manager"
      done


必须先创建日志目录；

### 检查服务运行状态

source /opt/k8s/bin/environment.sh  
 for node\_ip in ${ETCD\_NODE\_IPS[@]}  
 do  
 echo "&gt;&gt;&gt; ${node\_ip}"  
 ssh k8s@${node\_ip} "systemctl status kube-controller-manager|grep Active"  
 done  
 确保状态为 active (running)，否则查看日志，确认原因：


    $ journalctl -u kube-controller-manager


### 查看输出的 metric

注意：以下命令在 kube-controller-manager 节点上执行。

kube-controller-manager 监听 10252 端口，接收 https 请求：


    $ sudo netstat -lnpt|grep kube-controll
    tcp        0      0 127.0.0.1:10252         0.0.0.0:*               LISTEN      18377/kube-controll
    $ curl -s --cacert /etc/kubernetes/cert/ca.pem https://127.0.0.1:10252/metrics |head
    # HELP ClusterRoleAggregator_adds Total number of adds handled by workqueue: ClusterRoleAggregator
    # TYPE ClusterRoleAggregator_adds counter
    ClusterRoleAggregator_adds 3
    # HELP ClusterRoleAggregator_depth Current depth of workqueue: ClusterRoleAggregator
    # TYPE ClusterRoleAggregator_depth gauge
    ClusterRoleAggregator_depth 0
    # HELP ClusterRoleAggregator_queue_latency How long an item stays in workqueueClusterRoleAggregator before being requested.
    # TYPE ClusterRoleAggregator_queue_latency summary
    ClusterRoleAggregator_queue_latency{quantile="0.5"} 57018
    ClusterRoleAggregator_queue_latency{quantile="0.9"} 57268


curl --cacert CA 证书用来验证 kube-controller-manager https server 证书；  
 测试 kube-controller-manager 集群的高可用  
 停掉一个或两个节点的 kube-controller-manager 服务，观察其它节点的日志，看是否获取了 leader 权限。

### 查看当前的 leader


    $ kubectl get endpoints kube-controller-manager --namespace=kube-system  -o yaml
    apiVersion: v1
    kind: Endpoints
    metadata:
      annotations:
        control-plane.alpha.kubernetes.io/leader: '{"holderIdentity":"docker86-155_32dbaca9-e15f-11e8-87e7-e0db5521eb14","leaseDurationSeconds":15,"acquireTime":"2018-11-06T00:59:52Z","renewTime":"2018-11-06T01:34:01Z","leaderTransitions":39}'
      creationTimestamp: 2018-10-10T15:18:11Z
      name: kube-controller-manager
      namespace: kube-system
      resourceVersion: "6281708"
      selfLink: /api/v1/namespaces/kube-system/endpoints/kube-controller-manager
      uid: b38d3ea9-cc9f-11e8-9cde-d4ae52a3b675


可见，当前的 leader 为docker86-155 节点。

参考  
 关于 controller 权限和 use-service-account-credentials 参数：[https://github.com/kubernetes/kubernetes/issues/48208](https://github.com/kubernetes/kubernetes/issues/48208)  
 kublet 认证和授权：[https://kubernetes.io/docs/admin/kubelet-authentication-authorization/#kubelet-authorization](https://kubernetes.io/docs/admin/kubelet-authentication-authorization/#kubelet-authorization)

## 06-3.部署高可用 kube-scheduler 集群

该集群包含 3 个节点，启动后将通过竞争选举机制产生一个 leader 节点，其它节点为阻塞状态。当 leader 节点不可用后，剩余节点将再次进行选举产生新的 leader 节点，从而保证服务的可用性。

为保证通信安全，本文档先生成 x509 证书和私钥，kube-scheduler 在如下两种情况下使用该证书：

与 kube-apiserver 的安全端口通信;  
 在安全端口(https，10251) 输出 prometheus 格式的 metrics；

### 创建 kube-scheduler 证书和私钥


    cat > kube-scheduler-csr.json <<EOF
    {
        "CN": "system:kube-scheduler",
        "hosts": [
          "127.0.0.1",
          "192.168.86.156",  "192.168.86.155",  "192.168.86.154"
        ],
        "key": {
            "algo": "rsa",
            "size": 2048
        },
        "names": [
          {
            "C": "CN",
            "ST": "BeiJing",
            "L": "BeiJing",
            "O": "system:kube-scheduler",
            "OU": "4Paradigm"
          }
        ]
    }
    EOF


生成证书和私钥：


    cfssl gencert -ca=/etc/kubernetes/cert/ca.pem \
      -ca-key=/etc/kubernetes/cert/ca-key.pem \
      -config=/etc/kubernetes/cert/ca-config.json \
      -profile=kubernetes kube-scheduler-csr.json | cfssljson -bare kube-scheduler


### 创建和分发 kubeconfig 文件

kubeconfig 文件包含访问 apiserver 的所有信息，如 apiserver 地址、CA 证书和自身使用的证书；


    source /opt/k8s/bin/environment.sh
    kubectl config set-cluster kubernetes \
      --certificate-authority=/etc/kubernetes/cert/ca.pem \
      --embed-certs=true \
      --server=${KUBE_APISERVER} \
      --kubeconfig=kube-scheduler.kubeconfig
    
    kubectl config set-credentials system:kube-scheduler \
      --client-certificate=kube-scheduler.pem \
      --client-key=kube-scheduler-key.pem \
      --embed-certs=true \
      --kubeconfig=kube-scheduler.kubeconfig
    
    kubectl config set-context system:kube-scheduler \
      --cluster=kubernetes \
      --user=system:kube-scheduler \
      --kubeconfig=kube-scheduler.kubeconfig
    
    kubectl config use-context system:kube-scheduler --kubeconfig=kube-scheduler.kubeconfig


分发 kubeconfig 到所有 master 节点：


    source /opt/k8s/bin/environment.sh
    for node_ip in ${ETCD_NODE_IPS[@]}
      do
        echo ">>> ${node_ip}"
        scp kube-scheduler.kubeconfig k8s@${node_ip}:/etc/kubernetes/
      done


### 创建和分发 kube-scheduler systemd unit 文件


    cat > kube-scheduler.service <<EOF
    [Unit]
    Description=Kubernetes Scheduler
    Documentation=https://github.com/GoogleCloudPlatform/kubernetes
    
    [Service]
    ExecStart=/opt/k8s/bin/kube-scheduler \\
      --address=127.0.0.1 \\
      --kubeconfig=/etc/kubernetes/kube-scheduler.kubeconfig \\
      --leader-elect=true \\
      --alsologtostderr=true \\
      --logtostderr=false \\
      --log-dir=/var/log/kubernetes \\
      --v=2
    Restart=on-failure
    RestartSec=5
    User=k8s
    
    [Install]
    WantedBy=multi-user.target
    EOF


--address：在 127.0.0.1:10251 端口接收 http /metrics 请求；kube-scheduler 目前还不支持接收 https 请求；  
 --kubeconfig：指定 kubeconfig 文件路径，kube-scheduler 使用它连接和验证 kube-apiserver；  
 --leader-elect=true：集群运行模式，启用选举功能；被选为 leader 的节点负责处理工作，其它节点为阻塞状态；  
 User=k8s：使用 k8s 账户运行；

### 分发 systemd unit 文件到所有 master 节点：


    source /opt/k8s/bin/environment.sh
    for node_ip in ${ETCD_NODE_IPS[@]}
      do
        echo ">>> ${node_ip}"
        scp kube-scheduler.service root@${node_ip}:/etc/systemd/system/
      done


### 启动 kube-scheduler 服务


    source /opt/k8s/bin/environment.sh
    for node_ip in ${ETCD_NODE_IPS[@]}
      do
        echo ">>> ${node_ip}"
        ssh root@${node_ip} "mkdir -p /var/log/kubernetes && chown -R k8s /var/log/kubernetes"
        ssh root@${node_ip} "systemctl daemon-reload && systemctl enable kube-scheduler && systemctl restart kube-scheduler"
      done


必须先创建日志目录；

检查服务运行状态  
 source /opt/k8s/bin/environment.sh  
 for node\_ip in ${ETCD\_NODE\_IPS[@]}  
 do  
 echo "&gt;&gt;&gt; ${node\_ip}"  
 ssh k8s@${node\_ip} "systemctl status kube-scheduler|grep Active"  
 done

确保状态为 active (running)，否则查看日志，确认原因：


    journalctl -u kube-scheduler


### 查看输出的 metric

注意：以下命令在 kube-scheduler 节点上执行。

kube-scheduler 监听 10251 端口，接收 http 请求：

$ sudo netstat -lnpt|grep kube-sche  
 tcp        0      0 127.0.0.1:10251         0.0.0.0:\*               LISTEN      23783/kube-schedule  
 $ curl -s [http://127.0.0.1:10251/metrics](http://127.0.0.1:10251/metrics) |head

# HELP apiserver\_audit\_event\_total Counter of audit events generated and sent to the audit backend.

# TYPE apiserver\_audit\_event\_total counter

apiserver\_audit\_event\_total 0

# HELP go\_gc\_duration\_seconds A summary of the GC invocation durations.

# TYPE go\_gc\_duration\_seconds summary

go\_gc\_duration\_seconds{quantile="0"} 9.7715e-05  
 go\_gc\_duration\_seconds{quantile="0.25"} 0.000107676  
 go\_gc\_duration\_seconds{quantile="0.5"} 0.00017868  
 go\_gc\_duration\_seconds{quantile="0.75"} 0.000262444  
 go\_gc\_duration\_seconds{quantile="1"} 0.001205223

### 测试 kube-scheduler 集群的高可用

随便找一个或两个 master 节点，停掉 kube-scheduler 服务，看其它节点是否获取了 leader 权限（systemd 日志）。

查看当前的 leader  
 $ kubectl get endpoints kube-scheduler --namespace=kube-system  -o yaml  
 apiVersion: v1  
 kind: Endpoints  
 metadata:  
 annotations:  
 control-plane.alpha.kubernetes.io/leader: '{"holderIdentity":"kube-node3\_61f34593-6cc8-11e8-8af7-5254002f288e","leaseDurationSeconds":15,"acquireTime":"2018-06-10T16:09:56Z","renewTime":"2018-06-10T16:20:54Z","leaderTransitions":1}'  
 creationTimestamp: 2018-06-10T16:07:33Z  
 name: kube-scheduler  
 namespace: kube-system  
 resourceVersion: "4645"  
 selfLink: /api/v1/namespaces/kube-system/endpoints/kube-scheduler  
 uid: 62382d98-6cc8-11e8-96fa-525400ba84c6

## 07-1.部署 docker 组件

docker 是容器的运行环境，管理它的生命周期。kubelet 通过 Container Runtime Interface (CRI) 与 docker 进行交互。

### 安装依赖包

参考 [07-0.部署worker节点.md](07-0.%E9%83%A8%E7%BD%B2worker%E8%8A%82%E7%82%B9.md)

### 下载和分发 docker 二进制文件

到 [http://mirrors.ustc.edu.cn/docker-ce/linux/static/stable/x86\_64/](http://mirrors.ustc.edu.cn/docker-ce/linux/static/stable/x86_64/) 页面下载最新发布包：


    wget http://mirrors.ustc.edu.cn/docker-ce/linux/static/stable/x86_64/docker-18.06.1-ce.tgz
    tar -xvf docker-18.06.1-ce.tgz


分发二进制文件到所有 worker 节点：


    source /opt/k8s/bin/environment.sh
    for node_ip in ${NODE_IPS[@]}
      do
        echo ">>> ${node_ip}"
        scp docker/docker*  k8s@${node_ip}:/opt/k8s/bin/
        ssh k8s@${node_ip} "chmod +x /opt/k8s/bin/*"
      done


### 创建和分发 systemd unit 文件


    cat > docker.service <<"EOF"
    [Unit]
    Description=Docker Application Container Engine
    Documentation=http://docs.docker.io
    
    [Service]
    Environment="PATH=/opt/k8s/bin:/bin:/sbin:/usr/bin:/usr/sbin"
    EnvironmentFile=-/run/flannel/docker
    ExecStart=/opt/k8s/bin/dockerd --log-level=error $DOCKER_NETWORK_OPTIONS
    ExecReload=/bin/kill -s HUP $MAINPID
    Restart=on-failure
    RestartSec=5
    LimitNOFILE=infinity
    LimitNPROC=infinity
    LimitCORE=infinity
    Delegate=yes
    KillMode=process
    
    [Install]
    WantedBy=multi-user.target
    EOF


- EOF 前后有双引号，这样 bash 不会替换文档中的变量，如 $DOCKER\_NETWORK\_OPTIONS；
- dockerd 运行时会调用其它 docker 命令，如 docker-proxy，所以需要将 docker 命令所在的目录加到 PATH 环境变量中；
- flanneld 启动时将网络配置写入 `/run/flannel/docker` 文件中，dockerd 启动前读取该文件中的环境变量 `DOCKER_NETWORK_OPTIONS` ，然后设置 docker0 网桥网段；
- 如果指定了多个 `EnvironmentFile` 选项，则必须将 `/run/flannel/docker` 放在最后(确保 docker0 使用 flanneld 生成的 bip 参数)；
- docker 需要以 root 用于运行；
- docker 从 1.13 版本开始，可能将 **iptables FORWARD chain的默认策略设置为DROP**，从而导致 ping 其它 Node 上的 Pod IP 失败，遇到这种情况时，需要手动设置策略为 `ACCEPT`：


        $ sudo iptables -P FORWARD ACCEPT

    并且把以下命令写入 `/etc/rc.local` 文件中，防止节点重启**iptables FORWARD chain的默认策略又还原为DROP**


        /sbin/iptables -P FORWARD ACCEPT


完整 unit 见 [docker.service](https://github.com/opsnull/follow-me-install-kubernetes-cluster/blob/master/systemd/docker.service)

分发 systemd unit 文件到所有 worker 机器:


    source /opt/k8s/bin/environment.sh
    for node_ip in ${NODE_IPS[@]}
      do
        echo ">>> ${node_ip}"
        scp docker.service root@${node_ip}:/etc/systemd/system/
      done


### 配置和分发 docker 配置文件

使用国内的仓库镜像服务器以加快 pull image 的速度，同时增加下载的并发数 (需要重启 dockerd 生效)：


    cat > docker-daemon.json <<EOF
    {"insecure-registries":["192.168.86.8:5000","registry.xxx.com"],
        "registry-mirrors": ["https://jk4bb75a.mirror.aliyuncs.com", "https://docker.mirrors.ustc.edu.cn"],
        "max-concurrent-downloads": 20
    }
    EOF


分发 docker 配置文件到所有 work 节点：


    source /opt/k8s/bin/environment.sh
    for node_ip in ${NODE_IPS[@]}
      do
        echo ">>> ${node_ip}"
        ssh root@${node_ip} "mkdir -p  /etc/docker/"
        scp docker-daemon.json root@${node_ip}:/etc/docker/daemon.json
      done


### 启动 docker 服务


    source /opt/k8s/bin/environment.sh
    for node_ip in ${NODE_IPS[@]}
      do
        echo ">>> ${node_ip}"
        ssh root@${node_ip} "systemctl stop firewalld && systemctl disable firewalld"
        ssh root@${node_ip} "/usr/sbin/iptables -F && /usr/sbin/iptables -X && /usr/sbin/iptables -F -t nat && /usr/sbin/iptables -X -t nat"
        ssh root@${node_ip} "/usr/sbin/iptables -P FORWARD ACCEPT"
        ssh root@${node_ip} "systemctl daemon-reload && systemctl enable docker && systemctl restart docker"
        ssh root@${node_ip} 'for intf in /sys/devices/virtual/net/docker0/brif/*; do echo 1 > $intf/hairpin_mode; done'
        ssh root@${node_ip} "sudo sysctl -p /etc/sysctl.d/kubernetes.conf"
      done



    source /opt/k8s/bin/environment.sh
    for node_ip in ${NODE_IPS[@]}
      do
        echo ">>> ${node_ip}"
         ssh root@${node_ip} "systemctl restart docker"
      done


- 关闭 firewalld(centos7)/ufw(ubuntu16.04)，否则可能会重复创建 iptables 规则；
- 清理旧的 iptables rules 和 chains 规则；
- 开启 docker0 网桥下虚拟网卡的 hairpin 模式;


### 检查服务运行状态


    source /opt/k8s/bin/environment.sh
    for node_ip in ${NODE_IPS[@]}
      do
        echo ">>> ${node_ip}"
        ssh k8s@${node_ip} "systemctl status docker|grep Active"
      done


确保状态为 `active (running)`，否则查看日志，确认原因：


    $ journalctl -u docker


#### 检查 docker0 网桥


    source /opt/k8s/bin/environment.sh
    for node_ip in ${NODE_IPS[@]}
      do
        echo ">>> ${node_ip}"
        ssh k8s@${node_ip} "/usr/sbin/ip addr show flannel.1 && /usr/sbin/ip addr show docker0"
      done


确认各 work 节点的 docker0 网桥和 flannel.1 接口的 IP 处于同一个网段中(如下 172.30.39.0 和 172.30.39.1)：


    3: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default
        link/ether ce:2f:d6:53:e5:f3 brd ff:ff:ff:ff:ff:ff
        inet 172.30.39.0/32 scope global flannel.1
          valid_lft forever preferred_lft forever
        inet6 fe80::cc2f:d6ff:fe53:e5f3/64 scope link
          valid_lft forever preferred_lft forever
    4: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
        link/ether 02:42:bf:65:16:5c brd ff:ff:ff:ff:ff:ff
        inet 172.30.39.1/24 brd 172.30.39.255 scope global docker0
          valid_lft forever preferred_lft forever


## 07-2.部署 kubelet 组件

kublet 运行在每个 worker 节点上，接收 kube-apiserver 发送的请求，管理 Pod 容器，执行交互式命令，如 exec、run、logs 等。

kublet 启动时自动向 kube-apiserver 注册节点信息，内置的 cadvisor 统计和监控节点的资源使用情况。

为确保安全，本文档只开启接收 https 请求的安全端口，对请求进行认证和授权，拒绝未授权的访问(如 apiserver、heapster)。

### 创建 kubelet bootstrap kubeconfig 文件


    source /opt/k8s/bin/environment.sh
    for node_name in ${NODE_NAMES[@]}
      do
        echo ">>> ${node_name}"
    
        # 创建 token
        export BOOTSTRAP_TOKEN=$(kubeadm token create \
          --description kubelet-bootstrap-token \
          --groups system:bootstrappers:${node_name} \
          --kubeconfig ~/.kube/config)
    
        # 设置集群参数
        kubectl config set-cluster kubernetes \
          --certificate-authority=/etc/kubernetes/cert/ca.pem \
          --embed-certs=true \
          --server=${KUBE_APISERVER} \
          --kubeconfig=kubelet-bootstrap-${node_name}.kubeconfig
    
        # 设置客户端认证参数
        kubectl config set-credentials kubelet-bootstrap \
          --token=${BOOTSTRAP_TOKEN} \
          --kubeconfig=kubelet-bootstrap-${node_name}.kubeconfig
    
        # 设置上下文参数
        kubectl config set-context default \
          --cluster=kubernetes \
          --user=kubelet-bootstrap \
          --kubeconfig=kubelet-bootstrap-${node_name}.kubeconfig
    
        # 设置默认上下文
        kubectl config use-context default --kubeconfig=kubelet-bootstrap-${node_name}.kubeconfig
      done


- 证书中写入 Token 而非证书，证书后续由 controller-manager 创建。


查看 kubeadm 为各节点创建的 token：


    $ kubeadm token list --kubeconfig ~/.kube/config
    TOKEN                     TTL       EXPIRES                     USAGES                   DESCRIPTION               EXTRA GROUPS
    k0s2bj.7nvw1zi1nalyz4gz   23h       2018-06-14T15:14:31+08:00   authentication,signing   kubelet-bootstrap-token   system:bootstrappers:kube-node1
    mkus5s.vilnjk3kutei600l   23h       2018-06-14T15:14:32+08:00   authentication,signing   kubelet-bootstrap-token   system:bootstrappers:kube-node3
    zkiem5.0m4xhw0jc8r466nk   23h       2018-06-14T15:14:32+08:00   authentication,signing   kubelet-bootstrap-token   system:bootstrappers:kube-node2


- 创建的 token 有效期为 1 天，超期后将不能再被使用，且会被 kube-controller-manager 的 tokencleaner 清理(如果启用该 controller 的话)；
- kube-apiserver 接收 kubelet 的 bootstrap token 后，将请求的 user 设置为 system:bootstrap:<token id="">，group 设置为 system:bootstrappers；</token>


各 token 关联的 Secret：


    $ kubectl get secrets  -n kube-system
    NAME                     TYPE                                  DATA      AGE
    bootstrap-token-k0s2bj   bootstrap.kubernetes.io/token         7         1m
    bootstrap-token-mkus5s   bootstrap.kubernetes.io/token         7         1m
    bootstrap-token-zkiem5   bootstrap.kubernetes.io/token         7         1m
    default-token-99st7      kubernetes.io/service-account-token   3         2d


## 分发 bootstrap kubeconfig 文件到所有 worker 节点


    source /opt/k8s/bin/environment.sh
    for node_name in ${NODE_NAMES[@]}
      do
        echo ">>> ${node_name}"
        scp kubelet-bootstrap-${node_name}.kubeconfig k8s@${node_name}:/etc/kubernetes/kubelet-bootstrap.kubeconfig
      done


## 创建和分发 kubelet 参数配置文件

从 v1.10 开始，kubelet **部分参数**需在配置文件中配置，`kubelet --help` 会提示：


    DEPRECATED: This parameter should be set via the config file specified by the Kubelet's --config flag


创建 kubelet 参数配置模板文件：


    source /opt/k8s/bin/environment.sh
    cat > kubelet.config.json.template <<EOF
    {
      "kind": "KubeletConfiguration",
      "apiVersion": "kubelet.config.k8s.io/v1beta1",
      "authentication": {
        "x509": {
          "clientCAFile": "/etc/kubernetes/cert/ca.pem"
        },
        "webhook": {
          "enabled": true,
          "cacheTTL": "2m0s"
        },
        "anonymous": {
          "enabled": false
        }
      },
      "authorization": {
        "mode": "Webhook",
        "webhook": {
          "cacheAuthorizedTTL": "5m0s",
          "cacheUnauthorizedTTL": "30s"
        }
      },
      "address": "##NODE_IP##",
      "port": 10250,
      "readOnlyPort": 0,
      "cgroupDriver": "cgroupfs",
      "hairpinMode": "promiscuous-bridge",
      "serializeImagePulls": false,
      "featureGates": {
        "RotateKubeletClientCertificate": true,
        "RotateKubeletServerCertificate": true
      },
      "clusterDomain": "${CLUSTER_DNS_DOMAIN}",
      "clusterDNS": ["${CLUSTER_DNS_SVC_IP}"]
    }
    EOF


- address：API 监听地址，不能为 127.0.0.1，否则 kube-apiserver、heapster 等不能调用 kubelet 的 API；
- readOnlyPort=0：关闭只读端口(默认 10255)，等效为未指定；
- authentication.anonymous.enabled：设置为 false，不允许匿名访问 10250 端口；
- authentication.x509.clientCAFile：指定签名客户端证书的 CA 证书，开启 HTTP 证书认证；
- authentication.webhook.enabled=true：开启 HTTPs bearer token 认证；
- 对于未通过 x509 证书和 webhook 认证的请求(kube-apiserver 或其他客户端)，将被拒绝，提示 Unauthorized；
- authroization.mode=Webhook：kubelet 使用 SubjectAccessReview API 查询 kube-apiserver 某 user、group 是否具有操作资源的权限(RBAC)；
- featureGates.RotateKubeletClientCertificate、featureGates.RotateKubeletServerCertificate：自动 rotate 证书，证书的有效期取决于 kube-controller-manager 的 --experimental-cluster-signing-duration 参数；
- 需要 root 账户运行；


为各节点创建和分发 kubelet 配置文件：


    source /opt/k8s/bin/environment.sh
    for node_ip in ${NODE_IPS[@]}
      do 
        echo ">>> ${node_ip}"
        sed -e "s/##NODE_IP##/${node_ip}/" kubelet.config.json.template > kubelet.config-${node_ip}.json
        scp kubelet.config-${node_ip}.json root@${node_ip}:/etc/kubernetes/kubelet.config.json
      done


替换后的 kubelet.config.json 文件： [kubelet.config.json](https://github.com/opsnull/follow-me-install-kubernetes-cluster/blob/master/systemd/kubelet.config.json)

## 创建和分发 kubelet systemd unit 文件

创建 kubelet systemd unit 文件模板：


    cat > kubelet.service.template <<EOF
    [Unit]
    Description=Kubernetes Kubelet
    Documentation=https://github.com/GoogleCloudPlatform/kubernetes
    After=docker.service
    Requires=docker.service
    
    [Service]
    WorkingDirectory=/var/lib/kubelet
    ExecStart=/opt/k8s/bin/kubelet \\
      --bootstrap-kubeconfig=/etc/kubernetes/kubelet-bootstrap.kubeconfig \\
      --cert-dir=/etc/kubernetes/cert \\
      --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \\
      --config=/etc/kubernetes/kubelet.config.json \\
      --hostname-override=##NODE_NAME## \\
      --pod-infra-container-image=registry.access.redhat.com/rhel7/pod-infrastructure:latest \\
      --allow-privileged=true \\
      --alsologtostderr=true \\
      --logtostderr=false \\
      --log-dir=/var/log/kubernetes \\
      --v=2
    Restart=on-failure
    RestartSec=5
    
    [Install]
    WantedBy=multi-user.target
    EOF


- 如果设置了 `--hostname-override` 选项，则 `kube-proxy` 也需要设置该选项，否则会出现找不到 Node 的情况；
- `--bootstrap-kubeconfig`：指向 bootstrap kubeconfig 文件，kubelet 使用该文件中的用户名和 token 向 kube-apiserver 发送 TLS Bootstrapping 请求；
- K8S approve kubelet 的 csr 请求后，在 `--cert-dir` 目录创建证书和私钥文件，然后写入 `--kubeconfig` 文件；


替换后的 unit 文件：[kubelet.service](https://github.com/opsnull/follow-me-install-kubernetes-cluster/blob/master/systemd/kubelet.service)

为各节点创建和分发 kubelet systemd unit 文件：


    source /opt/k8s/bin/environment.sh
    for node_name in ${NODE_NAMES[@]}
      do 
        echo ">>> ${node_name}"
        sed -e "s/##NODE_NAME##/${node_name}/" kubelet.service.template > kubelet-${node_name}.service
        scp kubelet-${node_name}.service root@${node_name}:/etc/systemd/system/kubelet.service
      done


## Bootstrap Token Auth 和授予权限

kublet 启动时查找配置的 --kubeletconfig 文件是否存在，如果不存在则使用 --bootstrap-kubeconfig 向 kube-apiserver 发送证书签名请求 (CSR)。

kube-apiserver 收到 CSR 请求后，对其中的 Token 进行认证（事先使用 kubeadm 创建的 token），认证通过后将请求的 user 设置为 system:bootstrap:<token id="">，group 设置为 system:bootstrappers，这一过程称为 Bootstrap Token Auth。</token>

默认情况下，这个 user 和 group 没有创建 CSR 的权限:q，kubelet 启动失败，错误日志如下：


    $ sudo journalctl -u kubelet -a |grep -A 2 'certificatesigningrequests'
    May 06 06:42:36 kube-node1 kubelet[26986]: F0506 06:42:36.314378   26986 server.go:233] failed to run Kubelet: cannot create certificate signing request: certificatesigningrequests.certificates.k8s.io is forbidden: User "system:bootstrap:lemy40" cannot create certificatesigningrequests.certificates.k8s.io at the cluster scope
    May 06 06:42:36 kube-node1 systemd[1]: kubelet.service: Main process exited, code=exited, status=255/n/a
    May 06 06:42:36 kube-node1 systemd[1]: kubelet.service: Failed with result 'exit-code'.


解决办法是：创建一个 clusterrolebinding，将 group system:bootstrappers 和 clusterrole system:node-bootstrapper 绑定：


    $ kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --group=system:bootstrappers


## 启动 kubelet 服务


    source /opt/k8s/bin/environment.sh
    for node_ip in ${ETCD_NODE_IPS[@]}
      do
        echo ">>> ${node_ip}"
        ssh root@${node_ip} "mkdir -p /var/lib/kubelet"
        ssh root@${node_ip} "/usr/sbin/swapoff -a"
        ssh root@${node_ip} "mkdir -p /var/log/kubernetes && chown -R k8s /var/log/kubernetes"
        ssh root@${node_ip} "systemctl daemon-reload && systemctl enable kubelet && systemctl restart kubelet"
      done


- 关闭 swap 分区，否则 kubelet 会启动失败；
- 必须先创建工作和日志目录；


source /opt/k8s/bin/environment.sh  
 for node\_ip in ${NODE\_IPS[@]}  
 do  
 echo "&gt;&gt;&gt; ${node\_ip}"  
 ssh root@${node\_ip} "systemctl restart kubelet && systemctl status kubelet|grep Active:"  
 done


    $ journalctl -u kubelet |tail
    Jun 13 16:05:40 kube-node2 kubelet[22343]: I0613 16:05:40.388242   22343 feature_gate.go:226] feature gates: &{{} map[RotateKubeletServerCertificate:true RotateKubeletClientCertificate:true]}
    Jun 13 16:05:40 kube-node2 kubelet[22343]: I0613 16:05:40.394342   22343 mount_linux.go:211] Detected OS with systemd
    Jun 13 16:05:40 kube-node2 kubelet[22343]: W0613 16:05:40.394494   22343 cni.go:171] Unable to update cni config: No networks found in /etc/cni/net.d
    Jun 13 16:05:40 kube-node2 kubelet[22343]: I0613 16:05:40.399508   22343 server.go:376] Version: v1.10.4
    Jun 13 16:05:40 kube-node2 kubelet[22343]: I0613 16:05:40.399583   22343 feature_gate.go:226] feature gates: &{{} map[RotateKubeletServerCertificate:true RotateKubeletClientCertificate:true]}
    Jun 13 16:05:40 kube-node2 kubelet[22343]: I0613 16:05:40.399736   22343 plugins.go:89] No cloud provider specified.
    Jun 13 16:05:40 kube-node2 kubelet[22343]: I0613 16:05:40.399752   22343 server.go:492] No cloud provider specified: "" from the config file: ""
    Jun 13 16:05:40 kube-node2 kubelet[22343]: I0613 16:05:40.399777   22343 bootstrap.go:58] Using bootstrap kubeconfig to generate TLS client cert, key and kubeconfig file
    Jun 13 16:05:40 kube-node2 kubelet[22343]: I0613 16:05:40.446068   22343 csr.go:105] csr for this node already exists, reusing
    Jun 13 16:05:40 kube-node2 kubelet[22343]: I0613 16:05:40.453761   22343 csr.go:113] csr for this node is still valid


kubelet 启动后使用 --bootstrap-kubeconfig 向 kube-apiserver 发送 CSR 请求，当这个 CSR 被 approve 后，kube-controller-manager 为 kubelet 创建 TLS 客户端证书、私钥和 --kubeletconfig 文件。

注意：kube-controller-manager 需要配置 `--cluster-signing-cert-file` 和 `--cluster-signing-key-file` 参数，才会为 TLS Bootstrap 创建证书和私钥。


    $ kubectl get csr
    NAME                                                   AGE       REQUESTOR                 CONDITION
    node-csr-QzuuQiuUfcSdp3j5W4B2UOuvQ_n9aTNHAlrLzVFiqrk   43s       system:bootstrap:zkiem5   Pending
    node-csr-oVbPmU-ikVknpynwu0Ckz_MvkAO_F1j0hmbcDa__sGA   27s       system:bootstrap:mkus5s   Pending
    node-csr-u0E1-ugxgotO_9FiGXo8DkD6a7-ew8sX2qPE6KPS2IY   13m       system:bootstrap:k0s2bj   Pending
    
    $ kubectl get nodes
    No resources found.


- 三个 work 节点的 csr 均处于 pending 状态；


## approve kubelet CSR 请求

可以手动或自动 approve CSR 请求。推荐使用自动的方式，因为从 v1.8 版本开始，可以自动轮转approve csr 后生成的证书。

### 手动 approve CSR 请求

查看 CSR 列表：


    $ kubectl get csr
    NAME                                                   AGE       REQUESTOR                 CONDITION
    node-csr-QzuuQiuUfcSdp3j5W4B2UOuvQ_n9aTNHAlrLzVFiqrk   43s       system:bootstrap:zkiem5   Pending
    node-csr-oVbPmU-ikVknpynwu0Ckz_MvkAO_F1j0hmbcDa__sGA   27s       system:bootstrap:mkus5s   Pending
    node-csr-u0E1-ugxgotO_9FiGXo8DkD6a7-ew8sX2qPE6KPS2IY   13m       system:bootstrap:k0s2bj   Pending


approve CSR：


    $ kubectl certificate approve node-csr-QzuuQiuUfcSdp3j5W4B2UOuvQ_n9aTNHAlrLzVFiqrk
    certificatesigningrequest.certificates.k8s.io "node-csr-QzuuQiuUfcSdp3j5W4B2UOuvQ_n9aTNHAlrLzVFiqrk" approved


查看 Approve 结果：


    $ kubectl describe  csr node-csr-QzuuQiuUfcSdp3j5W4B2UOuvQ_n9aTNHAlrLzVFiqrk
    Name:               node-csr-QzuuQiuUfcSdp3j5W4B2UOuvQ_n9aTNHAlrLzVFiqrk
    Labels:             <none>
    Annotations:        <none>
    CreationTimestamp:  Wed, 13 Jun 2018 16:05:04 +0800
    Requesting User:    system:bootstrap:zkiem5
    Status:             Approved
    Subject:
             Common Name:    system:node:kube-node2
             Serial Number:
             Organization:   system:nodes
    Events:  <none>


- `Requesting User`：请求 CSR 的用户，kube-apiserver 对它进行认证和授权；
- `Subject`：请求签名的证书信息；
- 证书的 CN 是 system:node:kube-node2， Organization 是 system:nodes，kube-apiserver 的 Node 授权模式会授予该证书的相关权限；


### 自动 approve CSR 请求

创建三个 ClusterRoleBinding，分别用于自动 approve client、renew client、renew server 证书：


    cat > csr-crb.yaml <<EOF
     # Approve all CSRs for the group "system:bootstrappers"
     kind: ClusterRoleBinding
     apiVersion: rbac.authorization.k8s.io/v1
     metadata:
       name: auto-approve-csrs-for-group
     subjects:
     - kind: Group
       name: system:bootstrappers
       apiGroup: rbac.authorization.k8s.io
     roleRef:
       kind: ClusterRole
       name: system:certificates.k8s.io:certificatesigningrequests:nodeclient
       apiGroup: rbac.authorization.k8s.io
    ---
     # To let a node of the group "system:nodes" renew its own credentials
     kind: ClusterRoleBinding
     apiVersion: rbac.authorization.k8s.io/v1
     metadata:
       name: node-client-cert-renewal
     subjects:
     - kind: Group
       name: system:nodes
       apiGroup: rbac.authorization.k8s.io
     roleRef:
       kind: ClusterRole
       name: system:certificates.k8s.io:certificatesigningrequests:selfnodeclient
       apiGroup: rbac.authorization.k8s.io
    ---
    # A ClusterRole which instructs the CSR approver to approve a node requesting a
    # serving cert matching its client cert.
    kind: ClusterRole
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: approve-node-server-renewal-csr
    rules:
    - apiGroups: ["certificates.k8s.io"]
      resources: ["certificatesigningrequests/selfnodeserver"]
      verbs: ["create"]
    ---
     # To let a node of the group "system:nodes" renew its own server credentials
     kind: ClusterRoleBinding
     apiVersion: rbac.authorization.k8s.io/v1
     metadata:
       name: node-server-cert-renewal
     subjects:
     - kind: Group
       name: system:nodes
       apiGroup: rbac.authorization.k8s.io
     roleRef:
       kind: ClusterRole
       name: approve-node-server-renewal-csr
       apiGroup: rbac.authorization.k8s.io
    EOF


- auto-approve-csrs-for-group：自动 approve node 的第一次 CSR； 注意第一次 CSR 时，请求的 Group 为 system:bootstrappers；
- node-client-cert-renewal：自动 approve node 后续过期的 client 证书，自动生成的证书 Group 为 system:nodes;
- node-server-cert-renewal：自动 approve node 后续过期的 server 证书，自动生成的证书 Group 为 system:nodes;


生效配置：


    $ kubectl apply -f csr-crb.yaml


## 查看 kublet 的情况

等待一段时间(1-10 分钟)，三个节点的 CSR 都被自动 approve：


    $ kubectl get csr
    NAME                                                   AGE       REQUESTOR                 CONDITION
    csr-98h25                                              6m        system:node:kube-node2    Approved,Issued
    csr-lb5c9                                              7m        system:node:kube-node3    Approved,Issued
    csr-m2hn4                                              14m       system:node:kube-node1    Approved,Issued平时
    node-csr-7q7i0q4MF_K2TSEJj16At4CJFLlJkHIqei6nMIAaJCU   28m       system:bootstrap:k0s2bj   Approved,Issued
    node-csr-ND77wk2P8k2lHBtgBaObiyYw0uz1Um7g2pRvveMF-c4   35m       system:bootstrap:mkus5s   Approved,Issued
    node-csr-Nysmrw55nnM48NKwEJuiuCGmZoxouK4N8jiEHBtLQso   6m        system:bootstrap:zkiem5   Approved,Issued
    node-csr-QzuuQiuUfcSdp3j5W4B2UOuvQ_n9aTNHAlrLzVFiqrk   1h        system:bootstrap:zkiem5   Approved,Issued
    node-csr-oVbPmU-ikVknpynwu0Ckz_MvkAO_F1j0hmbcDa__sGA   1h        system:bootstrap:mkus5s   Approved,Issued
    node-csr-u0E1-ugxgotO_9FiGXo8DkD6a7-ew8sX2qPE6KPS2IY   1h        system:bootstrap:k0s2bj   Approved,Issued


所有节点均 ready：


    $ kubectl get nodes
    NAME         STATUS    ROLES     AGE       VERSION
    kube-node1   Ready     <none>    18m       v1.10.4
    kube-node2   Ready     <none>    10m       v1.10.4
    kube-node3   Ready     <none>    11m       v1.10.4


kube-controller-manager 为各 node 生成了 kubeconfig 文件和公私钥：


    $ ls -l /etc/kubernetes/kubelet.kubeconfig
    -rw------- 1 root root 2293 Jun 13 17:07 /etc/kubernetes/kubelet.kubeconfig
    
    $ ls -l /etc/kubernetes/cert/|grep kubelet
    -rw-r--r-- 1 root root 1046 Jun 13 17:07 kubelet-client.crt
    -rw------- 1 root root  227 Jun 13 17:07 kubelet-client.key
    -rw------- 1 root root 1334 Jun 13 17:07 kubelet-server-2018-06-13-17-07-45.pem
    lrwxrwxrwx 1 root root   58 Jun 13 17:07 kubelet-server-current.pem -> /etc/kubernetes/cert/kubelet-server-2018-06-13-17-07-45.pem


- kubelet-server 证书会周期轮转；


## kubelet 提供的 API 接口

kublet 启动后监听多个端口，用于接收 kube-apiserver 或其它组件发送的请求：


    $ sudo netstat -lnpt|grep kubelet
    tcp        0      0 172.27.129.111:4194     0.0.0.0:*               LISTEN      2490/kubelet
    tcp        0      0 127.0.0.1:10248         0.0.0.0:*               LISTEN      2490/kubelet
    tcp        0      0 172.27.129.111:10250    0.0.0.0:*               LISTEN      2490/kubelet


- 4194: cadvisor http 服务；
- 10248: healthz http 服务；
- 10250: https API 服务；注意：未开启只读端口 10255；


例如执行 `kubectl ec -it nginx-ds-5rmws -- sh` 命令时，kube-apiserver 会向 kubelet 发送如下请求：


    POST /exec/default/nginx-ds-5rmws/my-nginx?command=sh&input=1&output=1&tty=1


kubelet 接收 10250 端口的 https 请求：

- /pods、/runningpods
- /metrics、/metrics/cadvisor、/metrics/probes
- /spec
- /stats、/stats/container
- /logs
- /run/、"/exec/", "/attach/", "/portForward/", "/containerLogs/" 等管理；


详情参考：[https://github.com/kubernetes/kubernetes/blob/master/pkg/kubelet/server/server.go#L434:3](https://github.com/kubernetes/kubernetes/blob/master/pkg/kubelet/server/server.go#L434:3)

由于关闭了匿名认证，同时开启了 webhook 授权，所有访问 10250 端口 https API 的请求都需要被认证和授权。

预定义的 ClusterRole system:kubelet-api-admin 授予访问 kubelet 所有 API 的权限：


    $ kubectl describe clusterrole system:kubelet-api-admin
    Name:         system:kubelet-api-admin
    Labels:       kubernetes.io/bootstrapping=rbac-defaults
    Annotations:  rbac.authorization.kubernetes.io/autoupdate=true
    PolicyRule:
      Resources      Non-Resource URLs  Resource Names  Verbs
      ---------      -----------------  --------------  -----
      nodes          []                 []              [get list watch proxy]
      nodes/log      []                 []              [*]
      nodes/metrics  []                 []              [*]
      nodes/proxy    []                 []              [*]
      nodes/spec     []                 []              [*]
      nodes/stats    []                 []              [*]


## kublet api 认证和授权

kublet 配置了如下认证参数：

- authentication.anonymous.enabled：设置为 false，不允许匿名访问 10250 端口；
- authentication.x509.clientCAFile：指定签名客户端证书的 CA 证书，开启 HTTPs 证书认证；
- authentication.webhook.enabled=true：开启 HTTPs bearer token 认证；


同时配置了如下授权参数：

- authroization.mode=Webhook：开启 RBAC 授权；


kubelet 收到请求后，使用 clientCAFile 对证书签名进行认证，或者查询 bearer token 是否有效。如果两者都没通过，则拒绝请求，提示 Unauthorized：


    $ curl -s --cacert /etc/kubernetes/cert/ca.pem https://192.168.86.156:10250/metrics
    Unauthorized
    
    $ curl -s --cacert /etc/kubernetes/cert/ca.pem -H "Authorization: Bearer 123456" https://172.27.129.111:10250/metrics
    Unauthorized


通过认证后，kubelet 使用 SubjectAccessReview API 向 kube-apiserver 发送请求，查询证书或 token 对应的 user、group 是否有操作资源的权限(RBAC)；

证书认证和授权：


    $ # 权限不足的证书；
    $ curl -s --cacert /etc/kubernetes/cert/ca.pem --cert /etc/kubernetes/cert/kube-controller-manager.pem --key /etc/kubernetes/cert/kube-controller-manager-key.pem https://172.27.129.111:10250/metrics
    Forbidden (user=system:kube-controller-manager, verb=get, resource=nodes, subresource=metrics)
    
    $ # 使用部署 kubectl 命令行工具时创建的、具有最高权限的 admin 证书；
    $ curl -s --cacert /etc/kubernetes/cert/ca.pem --cert ./admin.pem --key ./admin-key.pem https://192.168.86.156:10250/metrics|head
    # HELP apiserver_client_certificate_expiration_seconds Distribution of the remaining lifetime on the certificate used to authenticate a request.
    # TYPE apiserver_client_certificate_expiration_seconds histogram
    apiserver_client_certificate_expiration_seconds_bucket{le="0"} 0
    apiserver_client_certificate_expiration_seconds_bucket{le="21600"} 0
    apiserver_client_certificate_expiration_seconds_bucket{le="43200"} 0
    apiserver_client_certificate_expiration_seconds_bucket{le="86400"} 0
    apiserver_client_certificate_expiration_seconds_bucket{le="172800"} 0
    apiserver_client_certificate_expiration_seconds_bucket{le="345600"} 0
    apiserver_client_certificate_expiration_seconds_bucket{le="604800"} 0
    apiserver_client_certificate_expiration_seconds_bucket{le="2.592e+06"} 0


- `--cacert`、`--cert`、`--key` 的参数值必须是文件路径，如上面的 `./admin.pem` 不能省略 `./`，否则返回 `401 Unauthorized`；


bear token 认证和授权：

创建一个 ServiceAccount，将它和 ClusterRole system:kubelet-api-admin 绑定，从而具有调用 kubelet API 的权限：


    kubectl create sa kubelet-api-test
    kubectl create clusterrolebinding kubelet-api-test --clusterrole=system:kubelet-api-admin --serviceaccount=default:kubelet-api-test
    SECRET=$(kubectl get secrets | grep kubelet-api-test | awk '{print $1}')
    TOKEN=$(kubectl describe secret ${SECRET} | grep -E '^token' | awk '{print $2}')
    echo ${TOKEN}
    
    $ curl -s --cacert /etc/kubernetes/cert/ca.pem -H "Authorization: Bearer ${TOKEN}" https://172.27.129.111:10250/metrics|head
    # HELP apiserver_client_certificate_expiration_seconds Distribution of the remaining lifetime on the certificate used to authenticate a request.
    # TYPE apiserver_client_certificate_expiration_seconds histogram
    apiserver_client_certificate_expiration_seconds_bucket{le="0"} 0
    apiserver_client_certificate_expiration_seconds_bucket{le="21600"} 0
    apiserver_client_certificate_expiration_seconds_bucket{le="43200"} 0
    apiserver_client_certificate_expiration_seconds_bucket{le="86400"} 0
    apiserver_client_certificate_expiration_seconds_bucket{le="172800"} 0
    apiserver_client_certificate_expiration_seconds_bucket{le="345600"} 0
    apiserver_client_certificate_expiration_seconds_bucket{le="604800"} 0
    apiserver_client_certificate_expiration_seconds_bucket{le="2.592e+06"} 0


### cadvisor 和 metrics

cadvisor 统计所在节点各容器的资源(CPU、内存、磁盘、网卡)使用情况，分别在自己的 http web 页面(4194 端口)和 10250 以 promehteus metrics 的形式输出。

浏览器访问 [http://172.27.129.105:4194/containers/](http://172.27.129.105:4194/containers/) 可以查看到 cadvisor 的监控页面：

![cadvisor-home](images/cadvisor-home.png)

浏览器访问 [https://172.27.129.80:10250/metrics](https://172.27.129.80:10250/metrics) 和 [https://172.27.129.80:10250/metrics/cadvisor](https://172.27.129.80:10250/metrics/cadvisor) 分别返回 kublet 和 cadvisor 的 metrics。

![cadvisor-metrics](images/cadvisor-metrics.png)

注意：

- kublet.config.json 设置 authentication.anonymous.enabled 为 false，不允许匿名证书访问 10250 的 https 服务；
- 参考[A.浏览器访问kube-apiserver安全端口.md](A.%E6%B5%8F%E8%A7%88%E5%99%A8%E8%AE%BF%E9%97%AEkube-apiserver%E5%AE%89%E5%85%A8%E7%AB%AF%E5%8F%A3.md)，创建和导入相关证书，然后访问上面的 10250 端口；


## 获取 kublet 的配置

从 kube-apiserver 获取各 node 的配置：

curl -sSL --cacert /etc/kubernetes/cert/ca.pem --cert ./admin.pem --key ./admin-key.pem [https://192.168.86.214:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy](https://192.168.86.214:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy)


    $ source /opt/k8s/bin/environment.sh
    $ # 使用部署 kubectl 命令行工具时创建的、具有最高权限的 admin 证书；
    $ curl -sSL --cacert /etc/kubernetes/cert/ca.pem --cert ./admin.pem --key ./admin-key.pem ${KUBE_APISERVER}/api/v1/nodes/docker86-155/proxy/configz | jq \
      '.kubeletconfig|.kind="KubeletConfiguration"|.apiVersion="kubelet.config.k8s.io/v1beta1"'
    {
      "syncFrequency": "1m0s",
      "fileCheckFrequency": "20s",
      "httpCheckFrequency": "20s",
      "address": "172.27.129.80",
      "port": 10250,
      "readOnlyPort": 10255,
      "authentication": {
        "x509": {},
        "webhook": {
          "enabled": false,
          "cacheTTL": "2m0s"
        },
        "anonymous": {
          "enabled": true
        }
      },
      "authorization": {
        "mode": "AlwaysAllow",
        "webhook": {
          "cacheAuthorizedTTL": "5m0s",
          "cacheUnauthorizedTTL": "30s"
        }
      },
      "registryPullQPS": 5,
      "registryBurst": 10,
      "eventRecordQPS": 5,
      "eventBurst": 10,
      "enableDebuggingHandlers": true,
      "healthzPort": 10248,
      "healthzBindAddress": "127.0.0.1",
      "oomScoreAdj": -999,
      "clusterDomain": "cluster.local.",
      "clusterDNS": [
        "10.254.0.2"
      ],
      "streamingConnectionIdleTimeout": "4h0m0s",
      "nodeStatusUpdateFrequency": "10s",
      "imageMinimumGCAge": "2m0s",
      "imageGCHighThresholdPercent": 85,
      "imageGCLowThresholdPercent": 80,
      "volumeStatsAggPeriod": "1m0s",
      "cgroupsPerQOS": true,
      "cgroupDriver": "cgroupfs",
      "cpuManagerPolicy": "none",
      "cpuManagerReconcilePeriod": "10s",
      "runtimeRequestTimeout": "2m0s",
      "hairpinMode": "promiscuous-bridge",
      "maxPods": 110,
      "podPidsLimit": -1,
      "resolvConf": "/etc/resolv.conf",
      "cpuCFSQuota": true,
      "maxOpenFiles": 1000000,
      "contentType": "application/vnd.kubernetes.protobuf",
      "kubeAPIQPS": 5,
      "kubeAPIBurst": 10,
      "serializeImagePulls": false,
      "evictionHard": {
        "imagefs.available": "15%",
        "memory.available": "100Mi",
        "nodefs.available": "10%",
        "nodefs.inodesFree": "5%"
      },
      "evictionPressureTransitionPeriod": "5m0s",
      "enableControllerAttachDetach": true,
      "makeIPTablesUtilChains": true,
      "iptablesMasqueradeBit": 14,
      "iptablesDropBit": 15,
      "featureGates": {
        "RotateKubeletClientCertificate": true,
        "RotateKubeletServerCertificate": true
      },
      "failSwapOn": true,
      "containerLogMaxSize": "10Mi",
      "containerLogMaxFiles": 5,
      "enforceNodeAllocatable": [
        "pods"
      ],
      "kind": "KubeletConfiguration",
      "apiVersion": "kubelet.config.k8s.io/v1beta1"
    }


或者参考代码中的注释：[https://github.com/kubernetes/kubernetes/blob/master/pkg/kubelet/apis/kubeletconfig/v1beta1/types.go](https://github.com/kubernetes/kubernetes/blob/master/pkg/kubelet/apis/kubeletconfig/v1beta1/types.go)

## 参考

1. kubelet 认证和授权：[https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet-authentication-authorization/](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet-authentication-authorization/)


source /opt/k8s/bin/environment.sh  
 for node\_ip in ${NODE\_IPS[@]}  
 do  
 echo "&gt;&gt;&gt; ${node\_ip}"  
 ssh root@${node\_ip} "systemctl daemon-reload && systemctl enable kubelet && systemctl restart kubelet"  
 done

source /opt/k8s/bin/environment.sh

for node\_ip in ${ETCD\_NODE\_IPS[@]}  
 do  
 echo "&gt;&gt;&gt; ${node\_ip}"  
 ssh root@${node\_ip} "mkdir -p /var/lib/kube-proxy"  
 ssh root@${node\_ip} "mkdir -p /var/log/kubernetes && chown -R k8s /var/log/kubernetes"  
 ssh root@${node\_ip} "systemctl daemon-reload && systemctl enable kube-proxy && systemctl restart kube-proxy"  
 done

source /opt/k8s/bin/environment.sh

for node\_ip in ${NODE\_IPS[@]}  
 do  
 echo "&gt;&gt;&gt; ${node\_ip}"  
 scp /usr/local/bin/pull-google-container root@${node\_ip}:/usr/local/bin/  
 ssh root@${node\_ip} "/usr/local/bin/pull-google-container k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.0"  
 done

192.168.86.18 192.168.86.21 192.168.86.91 192.168.86.9

cat &lt;&lt;EOF | kubectl apply -f -  
 kind: ClusterRoleBinding  
 apiVersion: rbac.authorization.k8s.io/v1beta1  
 metadata:  
 name: heapster-kubelet-api  
 roleRef:  
 apiGroup: rbac.authorization.k8s.io  
 kind: ClusterRole  
 name: system:kubelet-api-admin  
 subjects:

- kind: ServiceAccount  

name: heapster  

namespace: kube-system  

EOF


## 07-3.部署 kube-proxy 组件

kube-proxy 运行在所有 worker 节点上，，它监听 apiserver 中 service 和 Endpoint 的变化情况，创建路由规则来进行服务负载均衡。

本文档讲解部署 kube-proxy 的部署，使用 ipvs 模式。

### 下载和分发 kube-proxy 二进制文件

参考 [06-0.部署master节点.md](06-0.%E9%83%A8%E7%BD%B2master%E8%8A%82%E7%82%B9.md)

### 安装依赖包

各节点需要安装 `ipvsadm` 和 `ipset` 命令，加载 `ip_vs` 内核模块。

参考 [07-0.部署worker节点.md](07-0.%E9%83%A8%E7%BD%B2worker%E8%8A%82%E7%82%B9.md)

### 创建 kube-proxy 证书

创建证书签名请求：


    cat > kube-proxy-csr.json <<EOF
    {
      "CN": "system:kube-proxy",
      "key": {
        "algo": "rsa",
        "size": 2048
      },
      "names": [
        {
          "C": "CN",
          "ST": "BeiJing",
          "L": "BeiJing",
          "O": "k8s",
          "OU": "4Paradigm"
        }
      ]
    }
    EOF


- CN：指定该证书的 User 为 `system:kube-proxy`；
- 预定义的 RoleBinding `system:node-proxier` 将User `system:kube-proxy` 与 Role `system:node-proxier` 绑定，该 Role 授予了调用 `kube-apiserver` Proxy 相关 API 的权限；
- 该证书只会被 kube-proxy 当做 client 证书使用，所以 hosts 字段为空；


生成证书和私钥：


    cfssl gencert -ca=/etc/kubernetes/cert/ca.pem \
      -ca-key=/etc/kubernetes/cert/ca-key.pem \
      -config=/etc/kubernetes/cert/ca-config.json \
      -profile=kubernetes  kube-proxy-csr.json | cfssljson -bare kube-proxy


## 创建和分发 kubeconfig 文件


    source /opt/k8s/bin/environment.sh
    kubectl config set-cluster kubernetes \
      --certificate-authority=/etc/kubernetes/cert/ca.pem \
      --embed-certs=true \
      --server=${KUBE_APISERVER} \
      --kubeconfig=kube-proxy.kubeconfig
    
    kubectl config set-credentials kube-proxy \
      --client-certificate=kube-proxy.pem \
      --client-key=kube-proxy-key.pem \
      --embed-certs=true \
      --kubeconfig=kube-proxy.kubeconfig
    
    kubectl config set-context default \
      --cluster=kubernetes \
      --user=kube-proxy \
      --kubeconfig=kube-proxy.kubeconfig
    
    kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig


- `--embed-certs=true`：将 ca.pem 和 admin.pem 证书内容嵌入到生成的 kubectl-proxy.kubeconfig 文件中(不加时，写入的是证书文件路径)；


分发 kubeconfig 文件：


    source /opt/k8s/bin/environment.sh
    for node_name in ${NODE_NAMES[@]}
      do
        echo ">>> ${node_name}"
        scp kube-proxy.kubeconfig k8s@${node_name}:/etc/kubernetes/
      done


## 创建 kube-proxy 配置文件

从 v1.10 开始，kube-proxy **部分参数**可以配置文件中配置。可以使用 `--write-config-to` 选项生成该配置文件，或者参考 kubeproxyconfig 的类型定义源文件 ：[https://github.com/kubernetes/kubernetes/blob/master/pkg/proxy/apis/kubeproxyconfig/types.go](https://github.com/kubernetes/kubernetes/blob/master/pkg/proxy/apis/kubeproxyconfig/types.go)

创建 kube-proxy config 文件模板：


    cat >kube-proxy.config.yaml.template <<EOF
    apiVersion: kubeproxy.config.k8s.io/v1alpha1
    bindAddress: ##NODE_IP##
    clientConnection:
      kubeconfig: /etc/kubernetes/kube-proxy.kubeconfig
    clusterCIDR: ${CLUSTER_CIDR}
    healthzBindAddress: ##NODE_IP##:10256
    hostnameOverride: ##NODE_NAME##
    kind: KubeProxyConfiguration
    metricsBindAddress: ##NODE_IP##:10249
    mode: "ipvs"
    EOF


- `bindAddress`: 监听地址；
- `clientConnection.kubeconfig`: 连接 apiserver 的 kubeconfig 文件；
- `clusterCIDR`: kube-proxy 根据 `--cluster-cidr` 判断集群内部和外部流量，指定 `--cluster-cidr` 或 `--masquerade-all` 选项后 kube-proxy 才会对访问 Service IP 的请求做 SNAT；
- `hostnameOverride`: 参数值必须与 kubelet 的值一致，否则 kube-proxy 启动后会找不到该 Node，从而不会创建任何 ipvs 规则；
- `mode`: 使用 ipvs 模式；


为各节点创建和分发 kube-proxy 配置文件：


    source /opt/k8s/bin/environment.sh
    for (( i=0; i < 7; i++ ))
      do 
        echo ">>> ${NODE_NAMES[i]}"
        sed -e "s/##NODE_NAME##/${NODE_NAMES[i]}/" -e "s/##NODE_IP##/${NODE_IPS[i]}/" kube-proxy.config.yaml.template > kube-proxy-${NODE_NAMES[i]}.config.yaml
        scp kube-proxy-${NODE_NAMES[i]}.config.yaml root@${NODE_NAMES[i]}:/etc/kubernetes/kube-proxy.config.yaml
      done


替换后的 kube-proxy.config.yaml 文件：[kube-proxy.config.yaml](https://github.com/opsnull/follow-me-install-kubernetes-cluster/blob/master/systemd/kube-proxy.config.yaml)

## 创建和分发 kube-proxy systemd unit 文件


    source /opt/k8s/bin/environment.sh
    cat > kube-proxy.service <<EOF
    [Unit]
    Description=Kubernetes Kube-Proxy Server
    Documentation=https://github.com/GoogleCloudPlatform/kubernetes
    After=network.target
    
    [Service]
    WorkingDirectory=/var/lib/kube-proxy
    ExecStart=/opt/k8s/bin/kube-proxy \\
      --config=/etc/kubernetes/kube-proxy.config.yaml \\
      --alsologtostderr=true \\
      --logtostderr=false \\
      --log-dir=/var/log/kubernetes \\
      --v=2
    Restart=on-failure
    RestartSec=5
    LimitNOFILE=65536
    
    [Install]
    WantedBy=multi-user.target
    EOF


替换后的 unit 文件：[kube-proxy.service](https://github.com/opsnull/follow-me-install-kubernetes-cluster/blob/master/systemd/kube-proxy.service)

分发 kube-proxy systemd unit 文件：


    source /opt/k8s/bin/environment.sh
    for node_name in ${NODE_NAMES[@]}
      do 
        echo ">>> ${node_name}"
        scp kube-proxy.service root@${node_name}:/etc/systemd/system/
      done


### 启动 kube-proxy 服务


    source /opt/k8s/bin/environment.sh
    for node_ip in ${NODE_IPS[@]}
      do
        echo ">>> ${node_ip}"
        ssh root@${node_ip} "mkdir -p /var/lib/kube-proxy"
        ssh root@${node_ip} "mkdir -p /var/log/kubernetes && chown -R k8s /var/log/kubernetes"
        ssh root@${node_ip} "systemctl daemon-reload && systemctl enable kube-proxy && systemctl restart kube-proxy"
      done


- 必须先创建工作和日志目录；


### 检查启动结果


    source /opt/k8s/bin/environment.sh
    for node_ip in ${NODE_IPS[@]}
      do
        echo ">>> ${node_ip}"
        ssh k8s@${node_ip} "systemctl status kube-proxy|grep Active"
      done


确保状态为 `active (running)`，否则查看日志，确认原因：


    journalctl -u kube-proxy


## 查看监听端口和 metrics


    [k8s@kube-node1 ~]$ sudo netstat -lnpt|grep kube-prox
    tcp        0      0 172.27.129.105:10249    0.0.0.0:*               LISTEN      16847/kube-proxy
    tcp        0      0 172.27.129.105:10256    0.0.0.0:*               LISTEN      16847/kube-proxy


- 10249：http prometheus metrics port;
- 10256：http healthz port;


## 查看 ipvs 路由规则


    source /opt/k8s/bin/environment.sh
    for node_ip in ${NODE_IPS[@]}
      do
        echo ">>> ${node_ip}"
        ssh root@${node_ip} "/usr/sbin/ipvsadm -ln"
      done


预期输出：


    >>> 172.27.129.105
    IP Virtual Server version 1.2.1 (size=4096)
    Prot LocalAddress:Port Scheduler Flags
      -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
    TCP  10.254.0.1:443 rr persistent 10800
      -> 172.27.129.105:6443          Masq    1      0          0
    >>> 172.27.129.111
    IP Virtual Server version 1.2.1 (size=4096)
    Prot LocalAddress:Port Scheduler Flags
      -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
    TCP  10.254.0.1:443 rr persistent 10800
      -> 172.27.129.105:6443          Masq    1      0          0
    >>> 172.27.129.112
    IP Virtual Server version 1.2.1 (size=4096)
    Prot LocalAddress:Port Scheduler Flags
      -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
    TCP  10.254.0.1:443 rr persistent 10800
      -> 172.27.129.105:6443          Masq    1      0          0


可见将所有到 kubernetes cluster ip 443 端口的请求都转发到 kube-apiserver 的 6443 端口；

