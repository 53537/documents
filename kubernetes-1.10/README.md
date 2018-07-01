# Kubernetes 1.10

## 简介

- 本文档基于二进制文件全新安装 Kubernetes 1.8 及以上版本，包括但不限于 1.10.5+

- **SELinux**
- **IPVS**

## Environment

### 1. OS
- CentOS 7.4 minimal x86_64 
- CentOS 7.5 minimal x86_64 

### 2. Firewalld
- 由于 iptables 会被 kube-proxy 接管，因此需 **禁用** Firewalld

      systemctl disable firewalld && systemctl stop firewalld

### 3. SELinux
- ##### enforcing

### 4. Server

| 节点 | IP | 角色 | 配置 | 备注 |
| :---: | :--: | :--: | :--: | :--: |
| 50-55 | 192.168.50.55 | Etcd / API Server | 4 CPU, 4G MEM, 30G DISK | - |
| 50-56 | 192.168.50.56 | Etcd / Node | 4 CPU, 4G MEM, 30G DISK | - |
| 50-57 | 192.168.50.57 | Etcd / Node | 4 CPU, 4G MEM, 30G DISK | - |

### 5. swap
- ##### Disabled

### 6. Network
- #### Flannel
  - ##### vxlan

- #### Subnet
  - ##### Service Network
    - ##### 10.0.0.0/12
      - ###### 1,048,576
  - ##### Pod Network
    - ##### 10.64.0.0/10
      - ###### 4,194,304

### 7. Docker
- #### Docker-CE 17.12 或更新版本
- #### 安装方式
  - 安装 Docker-CE Repo
    
        curl https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/centos/docker-ce.repo \
        -o /etc/yum.repos.d/docker-ce.repo

  - 更改为**清华镜像源**

        sed -i 's#download.docker.com#mirrors.tuna.tsinghua.edu.cn/docker-ce#g' \
        /etc/yum.repos.d/docker-ce.repo

  - 安装 Docker-CE
   
        yum install -y docker-ce


  - ##### 修改 Docker 目录(/var/lib/docker)
    - **可选步骤**
    - 如默认 /var/lib 目录容量较小时，需要进行修改
    - 本例中将 docker 目录由 **/var/lib/docker** 改为 **/data/docker**
    - 操作如下
      - 创建目录

            mkdir /data/docker

      - 修改 docker 配置 (vim /usr/lib/systemd/system/docker.service)

            [Unit]
            Description=Docker Application Container Engine
            Documentation=https://docs.docker.com
            After=network.target firewalld.service
            [Service]
            Type=notify
            EnvironmentFile=-/run/flannel/docker
            EnvironmentFile=-/run/docker_opts.env
            EnvironmentFile=-/run/flannel/subnet.env
            EnvironmentFile=-/etc/sysconfig/docker
            EnvironmentFile=-/etc/sysconfig/docker-storage
            EnvironmentFile=-/etc/sysconfig/docker-network
            EnvironmentFile=-/run/docker_opts.env
            ExecStart=/usr/bin/dockerd \
                  --data-root /data/docker \
                  $DOCKER_OPT_BIP \
                  $DOCKER_OPT_IPMASQ \
                  $DOCKER_OPT_MTU
            ExecReload=/bin/kill -s HUP $MAINPID
            LimitNOFILE=infinity
            LimitNPROC=infinity
            LimitCORE=infinity
            TimeoutStartSec=0
            Delegate=yes
            [Install]
            WantedBy=multi-user.target

        - 添加 **--data-root /data/docker**

      - Reload 配置

            systemctl daemon-reload

      - 启动 Docker，生成目录
        
            systemctl start docker

      - 修改 SELinux 权限

            chcon -R -u system_u /data/docker
            chcon -R -t container_var_lib_t /data/docker
            chcon -R -t container_share_t /data/docker/overlay2 


### 8. kubernetes
#### 以下版本均已经过测试
- 1.8.1
- 1.8.7
- 1.10.5

## Certificate
### 1. 签发CA，在 50-55 上进行(可以是任一安装 openssl 的主机)
- #### 创建 /etc/ssl/gen 目录并进入(也可以是其它目录)

      mkdir /etc/ssl/gen
      cd /etc/ssl/gen

- #### 准备额外的选项, 配置文件 ca.cnf
  - ##### File: ca.cnf

        [ req ]
        req_extensions = v3_req
        distinguished_name = req_distinguished_name
        [req_distinguished_name]

        [ v3_req ]
        keyUsage = critical, cRLSign, keyCertSign, digitalSignature, keyEncipherment
        extendedKeyUsage = serverAuth, clientAuth
        subjectKeyIdentifier = hash
        authorityKeyIdentifier = keyid:always,issuer
        basicConstraints = critical, CA:true, pathlen:2

- #### 创建 CA Key

      openssl genrsa -out ca.key 3072

- #### 签发CA

      openssl req -x509 -new -nodes -key ca.key -days 1095 -out ca.pem -subj \
              "/CN=kubernetes/OU=System/C=CN/ST=Shanghai/L=Shanghai/O=k8s" \
              -config ca.cnf -extensions v3_req

    - 有效期 **1095** (d) = 3 years
    - 注意 -subj 参数中仅 'C=CN' 与 'Shanghai' 可以修改，**其它保持原样**，否则集群会遇到权限异常问题

### 2. 签发客户端证书

- #### 为 API Server 签发证书
  - ##### apiserver.cnf

        [ req ]
        req_extensions = v3_req
        distinguished_name = req_distinguished_name
        [req_distinguished_name]
        [ v3_req ]
        basicConstraints = critical, CA:FALSE
        keyUsage = critical, digitalSignature, keyEncipherment
        extendedKeyUsage = serverAuth, clientAuth
        #subjectKeyIdentifier = hash
        #authorityKeyIdentifier = keyid:always,issuer
        subjectAltName = @alt_names
        [alt_names]
        IP.1 = 10.0.0.1
        IP.2 = 192.168.50.51
        IP.3 = 192.168.50.55
        IP.4 = 192.168.50.56
        DNS.1 = kubernetes
        DNS.2 = kubernetes.default
        DNS.3 = kubernetes.default.svc
        DNS.4 = kubernetes.default.svc.cluster
        DNS.5 = kubernetes.default.svc.cluster.local

    - IP.2、IP.3 与 IP.4 加入到证书中，方便 API Server 后期的高可用
      - IP.2 为 HA VIP
      - IP.3 为 API Server 1
      - IP.4 为 API Server 2
    - 如果需要, 可以加上其它IP, 如额外的API Server

  - ##### 生成 key

        openssl genrsa -out apiserver.key 3072

  - ##### 生成证书请求

        openssl req -new -key apiserver.key -out apiserver.csr -subj \
                "/CN=kubernetes/OU=System/C=CN/ST=Shanghai/L=Shanghai/O=k8s" \
                -config apiserver.cnf

      - CN、OU、O 字段为认证时使用, 请勿修改
      - 注意 -subj 参数中仅 'C'、'ST' 与 'L' 可以修改，**其它保持原样**，否则集群会遇到权限异常问题

  - ##### 签发证书

        openssl x509 -req -in apiserver.csr 
                -CA ca.pem -CAkey ca.key -CAcreateserial \
                -out apiserver.pem -days 1095 \
                -extfile apiserver.cnf -extensions v3_req

    - 注意: 需要先去掉 apiserver.cnf 注释掉的两行

- #### 为 Kubelet 签发证书
  - ##### kubelet.cnf

        [ req ]
        req_extensions = v3_req
        distinguished_name = req_distinguished_name
        [req_distinguished_name]
        [ v3_req ]
        basicConstraints = critical, CA:FALSE
        keyUsage = critical, digitalSignature, keyEncipherment

  - ##### 设置名称变量

        name=kubelet
        conf=kubelet.cnf

  - ##### 生成 key

        openssl genrsa -out $name.key 3072

  - ##### 生成证书请求

        openssl req -new -key $name.key -out $name.csr -subj \
                "/CN=admin/OU=System/C=CN/ST=Shanghai/L=Shanghai/O=system:masters" \
                -config $conf

  - ##### 签发证书

        openssl x509 -req -in $name.csr -CA ca.pem \
                -CAkey ca.key -CAcreateserial -out $name.pem \
                -days 1095 -extfile $conf -extensions v3_req

- #### 为 kube-proxy 签发证书
  - ##### 复制 kubelet.cnf 文件

        cp kubelet.cnf kube-proxy.cnf

  - ##### 设置名称变量

        name=kube-proxy
        conf=kube-proxy.cnf

  - ##### 生成 key

        openssl genrsa -out $name.key 3072

  - ##### 生成证书请求

        openssl req -new -key $name.key -out $name.csr -subj \
                "/CN=system:kube-proxy/OU=System/C=CN/ST=Shanghai/L=Shanghai/O=k8s" \
                -config $conf

  - ##### 签发证书

        openssl x509 -req -in $name.csr \
                -CA ca.pem -CAkey ca.key -CAcreateserial \
                -out $name.pem -days 1095 \
                -extfile $conf -extensions v3_req

## Install
### 1. Etcd
- #### [>> Etcd with SSL](https://github.com/Statemood/documents/blob/master/kubernetes-1.8/etcd_cluster_with_ssl.md)

### 2. Flannel
- #### [>> Flannel with SSL](https://github.com/Statemood/documents/blob/master/kubernetes-1.8/flanneld_with_ssl.md)

### 3. Docker
- #### 使用 yum 安装 Docker 1.12, 依次在各节点执行安装

      yum install -y docker

- #### 或通过以下方法安装 Docker-CE

### 4. 使用 yum 安装 libnetfilter_conntrack conntrack-tools

      yum install -y libnetfilter_conntrack-devel libnetfilter_conntrack conntrack-tools

- #### 针对 k8s 1.8.1 以上版本
- #### 在所有节点执行

### 5. Kubernetes
- #### 使用 curl 命令下载二进制安装包

      curl -O https://dl.k8s.io/v1.8.1/kubernetes-server-linux-amd64.tar.gz

  - ##### 更多下载信息 >> [CHANGELOG-1.8.md#downloads-for-v181](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG-1.8.md#downloads-for-v181)
- #### 解压

      tar zxf kubernetes.tar.gz

- #### 安装

      cd kubernetes/server/bin
      [root@50-55 bin]# ll
      total 1853348
      -rwxr-x---. 1 root root  54989694 Oct 12 07:38 apiextensions-apiserver
      -rwxr-x---. 1 root root 109034012 Oct 12 07:38 cloud-controller-manager
      -rw-r-----. 1 root root         7 Oct 12 07:38 cloud-controller-manager.docker_tag
      -rw-r-----. 1 root root 110388224 Oct 12 07:38 cloud-controller-manager.tar
      -rwxr-x---. 1 root root 235977824 Oct 12 07:38 hyperkube
      -rwxr-x---. 1 root root 136284941 Oct 12 07:38 kubeadm
      -rwxr-x---. 1 root root  53836759 Oct 12 07:38 kube-aggregator
      -rw-r-----. 1 root root         7 Oct 12 07:38 kube-aggregator.docker_tag
      -rw-r-----. 1 root root  55190528 Oct 12 07:38 kube-aggregator.tar
      -rwxr-x---. 1 root root 192911402 Oct 12 07:38 kube-apiserver
      -rw-r-----. 1 root root         7 Oct 12 07:38 kube-apiserver.docker_tag
      -rw-r-----. 1 root root 194265600 Oct 12 07:38 kube-apiserver.tar
      -rwxr-x---. 1 root root 128087389 Oct 12 07:38 kube-controller-manager
      -rw-r-----. 1 root root         7 Oct 12 07:38 kube-controller-manager.docker_tag
      -rw-r-----. 1 root root 129441280 Oct 12 07:38 kube-controller-manager.tar
      -rwxr-x---. 1 root root  52274414 Oct 12 07:38 kubectl
      -rwxr-x---. 1 root root  55869752 Oct 12 07:38 kubefed
      -rwxr-x---. 1 root root 137505064 Oct 12 07:38 kubelet
      -rwxr-x---. 1 root root  47866320 Oct 12 07:38 kube-proxy
      -rw-r-----. 1 root root         7 Oct 12 07:38 kube-proxy.docker_tag
      -rw-r-----. 1 root root  94978048 Oct 12 07:38 kube-proxy.tar
      -rwxr-x---. 1 root root  53754721 Oct 12 07:38 kube-scheduler
      -rw-r-----. 1 root root         7 Oct 12 07:38 kube-scheduler.docker_tag
      -rw-r-----. 1 root root  55108608 Oct 12 07:38 kube-scheduler.tar
      [root@50-55 bin]#
      [root@50-55 bin]# rm -rfv *.*
      removed ‘cloud-controller-manager.docker_tag’
      removed ‘cloud-controller-manager.tar’
      removed ‘kube-aggregator.docker_tag’
      removed ‘kube-aggregator.tar’
      removed ‘kube-apiserver.docker_tag’
      removed ‘kube-apiserver.tar’
      removed ‘kube-controller-manager.docker_tag’
      removed ‘kube-controller-manager.tar’
      removed ‘kube-proxy.docker_tag’
      removed ‘kube-proxy.tar’
      removed ‘kube-scheduler.docker_tag’
      removed ‘kube-scheduler.tar’
      [root@50-56 bin]# ll
      total 1228924
      -rwxr-x---. 1 root root  54989694 Oct 12 07:38 apiextensions-apiserver
      -rwxr-x---. 1 root root 109034012 Oct 12 07:38 cloud-controller-manager
      -rwxr-x---. 1 root root 235977824 Oct 12 07:38 hyperkube
      -rwxr-x---. 1 root root 136284941 Oct 12 07:38 kubeadm
      -rwxr-x---. 1 root root  53836759 Oct 12 07:38 kube-aggregator
      -rwxr-x---. 1 root root 192911402 Oct 12 07:38 kube-apiserver
      -rwxr-x---. 1 root root 128087389 Oct 12 07:38 kube-controller-manager
      -rwxr-x---. 1 root root  52274414 Oct 12 07:38 kubectl
      -rwxr-x---. 1 root root  55869752 Oct 12 07:38 kubefed
      -rwxr-x---. 1 root root 137505064 Oct 12 07:38 kubelet
      -rwxr-x---. 1 root root  47866320 Oct 12 07:38 kube-proxy
      -rwxr-x---. 1 root root  53754721 Oct 12 07:38 kube-scheduler
      [root@50-55 bin]# chmod 755 *
      [root@50-55 bin]# cp -rf * /usr/bin

  - **删除无用文件**
  - **更改文件权限为 755**
  - 复制到 /usr/bin 目录下
  - **在普通节点上，仅需安装 kubelet 和 kube-proxy 两个服务**

- #### SELinux

      [root@50-55 bin]# for i in * \
                        do chcon -u system_u -t bin_t /usr/bin/$i; done

- #### 复制配置文件
  - ##### 将 etc-kubernetes 目录复制保存为 /etc/kubernetes


- #### 复制 systemctl 配置文件
  - ##### 将 systemctl 目录内文件复制到 /usr/lib/systemd/system/
  - ##### 在普通节点上，仅需安装 kubelet 和 kube-proxy 两个服务

- #### 执行 systemctl daemon-reload

      systemctl daemon-reload


## Configurations
### 1. 生成Token文件
- kubelet在首次启动时，会向kube-apiserver发送TLS Bootstrapping请求。如果kube-apiserver验证其与自己的token.csv一致，则为kubelet生成CA与key

        cd /etc/kubernetes
        [root@50-55 kubernetes]# echo "`head -c 16 /dev/urandom | od -An -t x | tr -d ' '`,kubelet-bootstrap,10001,\"system:kubelet-bootstrap\"" > token.csv

  - 在使用Dashboard时，可以使用 token 进行认证

### 2. 生成kubectl的kubeconfig文件
- #### 设置集群参数

      [root@50-55 kubernetes]# \
                  kubectl config set-cluster kubernetes \
                  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
                  --server=https://192.168.50.55:6443

- #### 设置客户端认证参数

      [root@50-55 kubernetes]# \
                  kubectl config set-credentials admin \
                  --client-certificate=/etc/kubernetes/ssl/kubelet.pem \
                  --client-key=/etc/kubernetes/ssl/kubelet.key

- #### 设置上下文参数

      [root@50-55 kubernetes]# \
                  kubectl config set-context kubernetes \
                  --cluster=kubernetes \
                  --user=admin

- #### 设置默认上下文

      [root@50-55 kubernetes]# kubectl config use-context kubernetes

    - kubelet.pem 证书的OU字段值为system:masters，kube-apiserver预定义的RoleBinding cluster-admin 将 Group system:masters 与 Role cluster-admin 绑定，该Role授予了调用kube-apiserver相关API的权限

    - 生成的kubeconfig被保存到~/.kube/config文件

### 3. 生成kubelet的bootstrapping kubeconfig文件
- #### 生成kubelet的bootstrapping kubeconfig文件

      [root@50-55 kubernetes]# \
                  kubectl config set-cluster kubernetes \
                  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
                  --server=https://192.168.50.55:6443 \
                  --kubeconfig=bootstrap.kubeconfig

- #### 设置客户端认证参数

      [root@50-55 kubernetes]# \
                  kubectl config set-credentials kubelet-bootstrap \
                  --token=`awk -F ',' '{print $1}' token.csv` \
                  --kubeconfig=bootstrap.kubeconfig

- #### 生成默认上下文参数

      [root@50-55 kubernetes]# \
                  kubectl config set-context default \
                  --cluster=kubernetes \
                  --user=kubelet-bootstrap \
                  --kubeconfig=bootstrap.kubeconfig

- #### 切换默认上下文

      [root@50-55 kubernetes]# \
                  kubectl config use-context default \
                  --kubeconfig=bootstrap.kubeconfig

    - --embed-certs为true时表示将certificate-authority证书写入到生成的bootstrap.kubeconfig文件中
    - 设置kubelet客户端认证参数时没有指定秘钥和证书，后续由kube-apiserver自动生成
    - 生成的bootstrap.kubeconfig文件会在当前文件路径下

### 5. 生成kubelet的 kubeconfig 文件
- #### 生成kubelet的 kubeconfig 文件

- #### 设置集群参数

      [root@50-55 kubernetes]# \
                  kubectl config set-cluster kubernetes \
                  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
                  --server=https://192.168.50.55:6443 \
                  --kubeconfig=kubelet.kubeconfig    

- #### 设置客户端认证参数

      [root@50-55 kubernetes]#
                  kubectl config set-credentials kubelet \
                  --client-certificate=/etc/kubernetes/ssl/kubelet.pem \
                  --client-key=/etc/kubernetes/ssl/kubelet.key \
                  --kubeconfig=kubelet.kubeconfig

- #### 生成上下文参数

      [root@50-55 kubernetes]# \
                  kubectl config set-context default \
                  --cluster=kubernetes \
                  --user=kubelet \
                  --kubeconfig=kubelet.kubeconfig

- #### 切换默认上下文

      [root@50-55 kubernetes]# \
                  kubectl config use-context default \
                  --kubeconfig=kubelet.kubeconfig


### 6. 生成kube-proxy的kubeconfig文件
- #### 设置集群参数

      [root@50-55 kubernetes]# \
                  kubectl config set-cluster kubernetes \
                  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
                  --server=https://192.168.50.55:6443 \
                  --kubeconfig=kube-proxy.kubeconfig    

- #### 设置客户端认证参数

      [root@50-55 kubernetes]#
                  kubectl config set-credentials kube-proxy \
                  --client-certificate=/etc/kubernetes/ssl/kube-proxy.pem \
                  --client-key=/etc/kubernetes/ssl/kube-proxy.key \
                  --kubeconfig=kube-proxy.kubeconfig

- #### 生成上下文参数

      [root@50-55 kubernetes]# \
                  kubectl config set-context default \
                  --cluster=kubernetes \
                  --user=kube-proxy \
                  --kubeconfig=kube-proxy.kubeconfig

- #### 切换默认上下文

      [root@50-55 kubernetes]# \
                  kubectl config use-context default \
                  --kubeconfig=kube-proxy.kubeconfig

    - --embed-cert 都为 true，这会将certificate-authority、client-certificate和client-key指向的证书文件内容写入到生成的kube-proxy.kubeconfig文件中
    - kube-proxy.pem证书中CN为system:kube-proxy，kube-apiserver预定义的 RoleBinding cluster-admin将User system:kube-proxy与Role system:node-proxier绑定，该Role授予了调用kube-apiserver Proxy相关API的权限

### 7. 将kubeconfig文件复制至所有节点上
- #### 将生成的两个 kubeconfig 文件复制到所有节点的 /etc/kubernetes 目录内

      [root@50-55 kubernetes]# cp bootstrap.kubeconfig kube-proxy.kubeconfig /etc/kubernetes/


### 8. 修改文件 /etc/kubernetes/apiserver
- #### File: /etc/kubernetes/apiserver

      ###
      # kubernetes system config
      #
      # The following values are used to configure the kube-apiserver
      #

      # The address on the local server to listen to.
      KUBE_API_ADDRESS="--bind-address=192.168.50.55 --insecure-bind-address=192.168.50.55"

      # The port on the local server to listen on.
      KUBE_API_PORT="--secure-port=6443 --insecure-port=8080"

      # Port minions listen on
      # KUBELET_PORT="--kubelet-port=10250"

      # Comma separated list of nodes in the etcd cluster
      KUBE_ETCD_SERVERS="--etcd-servers=https://192.168.50.55:2379,https://192.168.50.56:2379,https://192.168.50.57:2379"

      # Address range to use for services
      KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.0.0.0/12"

      # default admission control policies
      KUBE_ADMISSION_CONTROL="--admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota"

      # Add your own!
      KUBE_API_ARGS="--allow-privileged=true \
                     --service_account_key_file=/etc/kubernetes/ssl/apiserver.key \
                     --tls-cert-file=/etc/kubernetes/ssl/apiserver.pem \
                     --tls-private-key-file=/etc/kubernetes/ssl/apiserver.key \
                     --client-ca-file=/etc/kubernetes/ssl/ca.pem \
                     --etcd-cafile=/etc/kubernetes/ssl/ca.pem \
                     --etcd-certfile=/etc/etcd/ssl/etcd.pem \
                     --etcd-keyfile=/etc/etcd/ssl/etcd.key \
                     --token-auth-file=/etc/kubernetes/token.csv \
                     --runtime-config=rbac.authorization.k8s.io/v1alpha1 \
                     --authorization-mode=RBAC \
                     --kubelet-https=true \
                     --enable-bootstrap-token-auth"

    -   kube-apiserver 1.6版本开始使用etcd v3 API和存储格式
    -   --authorization-mode=RBAC指定在安全端口使用RBAC授权模式，拒绝未通过授权的请求
    -   kube-scheduler、kube-controller-manager和kube-apiserver部署在同一台机器上，它们使用非安全端口和kube-apiserver通信
    -   kubelet、kube-proxy、kubectl部署在其它Node节点上，如果通过安全端口访问 kube-apiserver，则必须先通过TLS证书认证，再通过RBAC授权
    -   kube-proxy、kubectl通过在使用的证书里指定相关的User、Group来达到通过RBAC授权的目的
    -   如果使用了kubelet TLS Boostrap机制，则不能再指定--kubelet-certificate-authority、--kubelet-client-certificate和--kubelet-client-key选项，否则后续kube-apiserver校验kubelet证书时出现x509: certificate signed by unknown authority错误
    -   --admission-control值必须包含ServiceAccount
    -   --bind-address不能为127.0.0.1
    -   --service-cluster-ip-range指定Service Cluster IP地址段，该地址段不能路由可达
    -   --service-node-port-range=${NODE_PORT_RANGE}指定 NodePort 的端口范围
    -   缺省情况下kubernetes对象保存在etcd /registry路径下，可以通过--etcd-prefix参数进行调整

    - #### For 1.10+, 参数 --service_account_key_file 变为 --service-account-key-file

### 9. 修改文件 /etc/kubernetes/controller-manager
- #### File: /etc/kubernetes/controller-manager

      ###
      # The following values are used to configure the kubernetes controller-manager

      # defaults from config and apiserver should be adequate

      # Add your own!
      KUBE_CONTROLLER_MANAGER_ARGS="\
          --master=https://192.168.50.55:6443 \
          --service_account_private_key_file=/etc/kubernetes/ssl/ca.key \
          --root-ca-file=/etc/kubernetes/ssl/ca.pem \
          --allocate-node-cidrs=true \
          --cluster-name=kubernetes \
          --cluster-signing-cert-file=/etc/kubernetes/ssl/ca.pem \
          --cluster-signing-key-file=/etc/kubernetes/ssl/ca.key \
          --leader-elect=true \
          --service-cluster-ip-range=10.0.0.0/12 \
          --cluster-cidr=10.64.0.0/10 \
          --kubeconfig=/etc/kubernetes/kubelet.kubeconfig"

    - --master=http://{MASTER_IP}:8080：使用非安全8080端口与kube-apiserver 通信
    - --cluster-cidr指定Cluster中Pod的CIDR范围，该网段在各Node间必须路由可达(flanneld保证)
    - --service-cluster-ip-range参数指定Cluster中Service的CIDR范围，该网络在各 Node间必须路由不可达，必须和kube-apiserver中的参数一致
    - --cluster-signing-* 指定的证书和私钥文件用来签名为TLS BootStrap创建的证书和私钥
    - --root-ca-file用来对kube-apiserver证书进行校验，指定该参数后，才会在Pod容器的ServiceAccount中放置该CA证书文件
    - --leader-elect=true部署多台机器组成的master集群时选举产生一处于工作状态的 kube-controller-manager进程

    - #### For 1.10+:
      - 参数 --service_account_key_file 变为 --service-account-key-file

### 10. 修改文件 /etc/kubernetes/scheduler
- #### File: /etc/kubernetes/scheduler

      ###
      # kubernetes scheduler config

      # default config should be adequate

      # Add your own!
      KUBE_SCHEDULER_ARGS="\
          --address=127.0.0.1 \
          --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \
          --leader-elect=true"

### 11. 修改文件 /etc/kubernetes/config
- #### File: /etc/kubernetes/config

      ###
      # kubernetes system config
      #
      # The following values are used to configure various aspects of all
      # kubernetes services, including
      #
      #   kube-apiserver.service
      #   kube-controller-manager.service
      #   kube-scheduler.service
      #   kubelet.service
      #   kube-proxy.service
      # logging to stderr means we get it in the systemd journal
      KUBE_LOGTOSTDERR="--logtostderr=true"

      # journal message level, 0 is debug
      KUBE_LOG_LEVEL="--v=2"

      # Should this cluster be allowed to run privileged docker containers
      KUBE_ALLOW_PRIV="--allow-privileged=true"

      # How the controller-manager, scheduler, and proxy find the apiserver
      KUBE_MASTER="--master=https://192.168.50.55:6443"

### 12. 修改文件 /etc/kubernetes/kubelet
- #### File: /etc/kubernetes/kubelet

      ###
      # kubernetes kubelet (minion) config

      # The address for the info server to serve on (set to 0.0.0.0 or "" for all interfaces)
      KUBELET_ADDRESS="--address=192.168.50.55"

      # The port for the info server to serve on
      # KUBELET_PORT="--port=10250"

      # You may leave this blank to use the actual hostname
      KUBELET_HOSTNAME=""

      # location of the api-server
      #KUBELET_API_SERVER="--api-server=https://192.168.50.55:6443"

      # pod infrastructure container
      KUBELET_POD_INFRA_CONTAINER="--pod-infra-container-image=img.rulin.me/library/pod-infrastructure:latest"

      KUBELET_ARGS="--cgroup-driver=systemd \
                    --tls-cert-file=/etc/kubernetes/ssl/kubelet-50-55.pem \
                    --tls-private-key-file=/etc/kubernetes/ssl/kubelet-50-55.key \
                    --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \
                    --bootstrap-kubeconfig=/etc/kubernetes/bootstrap.kubeconfig \
                    --cert-dir=/etc/kubernetes/ssl"

  - #### --api-server 参数 kubelet 已不再使用
  - #### --experimental-bootstrap-kubeconfig 已弃用，使用新参数 --bootstrap-kubeconfig
  - #### 在其它节点上，注意修改为正确的证书文件名

  - #### For 1.10+:
    - 弃用参数:
      - --address
      - --allow-privileged
      - --cgroup-driver
      - --tls-cert-file
      - --tls-private-key-file


### 13. 修改文件 /etc/kubernetes/proxy
- #### File: /etc/kubernetes/proxy

      ###
      # kubernetes proxy config

      # default config should be adequate

      # Add your own!
      KUBE_PROXY_ARGS="--bind-address=192.168.50.55 \
                       --cluster-cidr=10.0.0.0/12 \
                       --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig"


### 14. Group & User
- #### Add Group & User

      [root@50-55 kubernetes]# groupadd -g 200 kube
      [root@50-55 kubernetes]# useradd -g kube kube -u 200 -d / -s /sbin/nologin -M

### 15. Work directory: /var/lib/kubelet
- #### Create directory

      [root@50-55 kubernetes]# mkdir /var/lib/kubelet
      [root@50-55 kubernetes]# chown kube 

- #### Set SELinux rules

      [root@50-55 kubernetes]# \
                  chcon -u system_u -t svirt_sandbox_file_t /var/lib/kubelet

### 16. Permission
- #### Files Permission, ensure that kubeconfig files we created are readable for user kube

      [root@50-55 kubernetes]# setfacl -m u:kube:r /etc/kubernetes/*.kubeconfig

    *-*

### 17. For 1.8.7
- #### 在 1.8.7 上，需要在每个节点安装以下包

      [root@50-55 kubernetes]# yum install -y  conntrack-tools libnetfilter_conntrack libnetfilter_conntrack-devel

  - **此操作为解决问题**: Jan 31 14:16:43 localhost kube-proxy: E0131 14:16:43.924024   30629 proxier.go:1716] Failed to delete stale service IP 10.0.0.10 connections, error: error deleting connection tracking state for UDP service IP: 10.0.0.10, error: error looking for path of conntrack: exec: "**conntrack**": executable file not found in $PATH            

## **Startup**
### 1. 在 API Server 节点

- #### Start & Enable kube-apiserver

      [root@50-55 kubernetes]# systemctl start  kube-apiserver
      [root@50-55 kubernetes]# systemctl enable kube-apiserver

- #### Start & Enable controller-manager

      [root@50-55 kubernetes]# systemctl start  kube-controller-manager
      [root@50-55 kubernetes]# systemctl enable kube-controller-manager

- #### Start & Enable scheduler

      [root@50-55 kubernetes]# systemctl start  kube-scheduler
      [root@50-55 kubernetes]# systemctl enable kube-scheduler

- #### Start & Enable kubelet

      [root@50-55 kubernetes]# systemctl start  kubelet
      [root@50-55 kubernetes]# systemctl enable kubelet

- #### Start & Enable kube-proxy

      [root@50-55 kubernetes]# systemctl start  kube-proxy
      [root@50-55 kubernetes]# systemctl enable kube-proxy

- #### Quick commands

      [root@50-55 kubernetes]# \
                  for k in kube-apiserver \
                           kube-controller-manager \
                           kube-scheduler \
                           kubelet \
                           kube-proxy \
                  do  systemctl start  $k; \
                      systemctl enable $k; \
                      systemctl status $k; \
                  done

### 2. On Kubelet Nodes
  - #### Start & Enable kubelet

        [root@50-56 ~]# systemctl start  kubelet
        [root@50-56 ~]# systemctl enable kubelet

  - #### Start & Enable kube-proxy

        [root@50-56 ~]# systemctl start  kube-proxy
        [root@50-56 ~]# systemctl enable kube-proxy


### 4. 检查集群状态

    [root@50-55 kubernetes]# kubectl get cs
    NAME                 STATUS    MESSAGE              ERROR
    scheduler            Healthy   ok                   
    controller-manager   Healthy   ok                   
    etcd-2               Healthy   {"health": "true"}   
    etcd-1               Healthy   {"health": "true"}   
    etcd-0               Healthy   {"health": "true"}  

## References
1. [Create-The-File-Of-Kubeconfig-For-K8s](https://o-my-chenjian.com/2017/04/26/Create-The-File-Of-Kubeconfig-For-K8s/)
