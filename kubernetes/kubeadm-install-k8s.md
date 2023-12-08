# kubeadm install k8s

## kubeadm install k8s

[kubeadm官方文档](https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

#### 配置主机名

```shell
hostnamectl set-hostname k8s-master
```

#### 关闭防火墙

```shell
systemctl stop firewalld
```

#### 关闭selinux

```shell
sed -i 's/enforcing/disabled/' /etc/selinux/config

setenforce 0
```

#### 互作本地解析

```shell
vim /etc/hosts
:1     localhost       localhost.localdomain   localhost6      localhost6.localdomain6
127.0.0.1       localhost       localhost.localdomain   localhost4      localhost4.localdomain4

172.26.95.187   iZ0jl26jymoe0uwwdjv6w0Z iZ0jl26jymoe0uwwdjv6w0Z
172.26.95.187  k8s-master
```

#### ssh免密

```shell
ssh-keygen
....

ssh-copy-id root@172.26.95.187

```

#### 加载br\_netfilter模块

```shell
# 加载模块
[root@k8s-master ~]# modprobe br_netfilter

# 查看加载请看
[root@k8s-master ~]# lsmod | grep br_netfilter
br_netfilter           22256  0 
bridge                151336  1 br_netfilter

# 永久生效
cat <<EOF | tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

```

#### 允许iptables检查桥接流量

```shell
[root@k8s-master ~]# cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

[root@k8s-master ~]# cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

[root@k8s-master ~]# sudo sysctl --system

```

#### 关闭swp

```shell
# 临时关闭
[root@k8s-master ~]# swapoff -a

# 永久关闭
[root@k8s-master ~]# sed -ri 's/.*swap.*/#&/' /etc/fstab

```

#### 时间同步

```shell
yum install ntpdate -y
# 同步网络时间
[root@k8s-master ~]# ntpdate time.nist.gov
26 Apr 19:58:05 ntpdate[13947]: the NTP socket is in use, exiting

# 将网络时间写入硬件时间
[root@k8s-master ~]# hwclock --systohc

```

#### 安装docker

```shell
yum install -y yum-utils            device-mapper-persistent-data            lvm2 --skip-broken
yum-config-manager     --add-repo     https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
sed -i 's/download.docker.com/mirrors.aliyun.com\/docker-ce/g' /etc/yum.repos.d/docker-ce.repo
yum makecache fast
yum install -y docker-ce
```

#### 安装kubeadm、kubelet、kubectl

```shell
添加镜像源
[root@k8s-master ~]# cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

#建立 k8s YUM 缓存
[root@k8s-master ~]# yum makecache

# 查看可安装版本
[root@k8s-master ~]# yum list kubelet --showduplicates

...
...
kubelet.x86_64                                           1.23.0-0                                             kubernetes
kubelet.x86_64                                           1.23.1-0                                             kubernetes
kubelet.x86_64                                           1.23.2-0                                             kubernetes
kubelet.x86_64                                           1.23.3-0                                             kubernetes
kubelet.x86_64                                           1.23.4-0                                             kubernetes
kubelet.x86_64                                           1.23.5-0                                             kubernetes
kubelet.x86_64                                           1.23.6-0                                             kubernetes

# 开始安装（指定你要安装的版本）
[root@k8s-master ~]# yum install -y kubelet-1.23.6 kubeadm-1.23.6 kubectl-1.23.6

# 设置开机自启动并启动kubelet（kubelet由systemd管理）
[root@k8s-master ~]# systemctl enable kubelet && systemctl start kubelet

```

#### k8s初始化

**master节点执行**

```shell
[root@k8s-master ~]# kubeadm init \
>   --apiserver-advertise-address=172.26.95.187 \
>   --image-repository registry.aliyuncs.com/google_containers \
>   --kubernetes-version v1.23.6 \
>   --service-cidr=10.96.0.0/12 \
>   --pod-network-cidr=10.244.0.0/16 \
>   --ignore-preflight-errors=all
```

**节点初始化遇到的问题及解决办法**

```shell
执行kubeadm init后报错如下
...
[kubelet-check] It seems like the kubelet isn't running or healthy.
[kubelet-check] The HTTP call equal to 'curl -sSL http://localhost:10248/healthz' failed with error: Get "http://localhost:10248/healthz": dial tcp [::1]:10248: connect: connection refused.
[kubelet-check] It seems like the kubelet isn't running or healthy.
[kubelet-check] The HTTP call equal to 'curl -sSL http://localhost:10248/healthz' failed with error: Get "http://localhost:10248/healthz": dial tcp [::1]:10248: connect: connection refused.
[kubelet-check] It seems like the kubelet isn't running or healthy.
[kubelet-check] The HTTP call equal to 'curl -sSL http://localhost:10248/healthz' failed with error: Get "http://localhost:10248/healthz": dial tcp [::1]:10248: connect: connection refused.
[kubelet-check] It seems like the kubelet isn't running or healthy.
[kubelet-check] The HTTP call equal to 'curl -sSL http://localhost:10248/healthz' failed with error: Get "http://localhost:10248/healthz": dial tcp [::1]:10248: connect: connection refused.
[kubelet-check] It seems like the kubelet isn't running or healthy.
[kubelet-check] The HTTP call equal to 'curl -sSL http://localhost:10248/healthz' failed with error: Get "http://localhost:10248/healthz": dial tcp [::1]:10248: connect: connection refused.

        Unfortunately, an error has occurred:
                timed out waiting for the condition

        This error is likely caused by:
                - The kubelet is not running
                - The kubelet is unhealthy due to a misconfiguration of the node in some way (required cgroups disabled)

        If you are on a systemd-powered system, you can try to troubleshoot the error with the following commands:
                - 'systemctl status kubelet'
                - 'journalctl -xeu kubelet'

        Additionally, a control plane component may have crashed or exited when started by the container runtime.
        To troubleshoot, list all containers using your preferred container runtimes CLI.

        Here is one example how you may list all Kubernetes containers running in docker:
                - 'docker ps -a | grep kube | grep -v pause'
                Once you have found the failing container, you can inspect its logs with:
                - 'docker logs CONTAINERID'

error execution phase wait-control-plane: couldn't initialize a Kubernetes cluster
To see the stack trace of this error execute with --v=5 or higher

解决办法
在/etc/docker/daemon.json文件中添加以下内容
vim /etc/docker/daemon.json

{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
相关链接
问题分析：
之前我的Docker是用yum安装的，docker的cgroup驱动程序默认设置为system。默认情况下Kubernetes cgroup为systemd，我们需要更改Docker cgroup驱动，
https://blog.csdn.net/qq_43762191/article/details/125567365?ops_request_misc=&request_id=&biz_id=102&utm_term=%5Bkubelet-check%5D%20The%20HTTP%20call%20&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduweb~default-0-125567365.nonecase&spm=1018.2226.3001.4187
```

**参数说明**

```shell
--apiserver-advertise-address  # 集群master地址
--image-repository             # 指定k8s镜像仓库地址
--kubernetes-version           # 指定K8s版本（与kubeadm、kubelet版本保持一致）
--service-cidr                 # Pod统一访问入口
--pod-network-cidr             # Pod网络（与CNI网络保持一致）

```

**初始化内容**

```shell
[root@k8s-master ~]# kubeadm init \
>   --apiserver-advertise-address=172.26.95.187 \
>   --image-repository registry.aliyuncs.com/google_containers \
>   --kubernetes-version v1.23.6 \
>   --service-cidr=10.96.0.0/12 \
>   --pod-network-cidr=10.244.0.0/16 \
>   --ignore-preflight-errors=all
[init] Using Kubernetes version: v1.23.6
[preflight] Running pre-flight checks
        [WARNING Service-Docker]: docker service is not enabled, please run 'systemctl enable docker.service'
        [WARNING SystemVerification]: this Docker version is not on the list of validated versions: 24.0.2. Latest validated version: 20.10
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [k8s-master kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 172.26.95.187]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [k8s-master localhost] and IPs [172.26.95.187 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [k8s-master localhost] and IPs [172.26.95.187 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 5.003064 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.23" in namespace kube-system with the configuration for the kubelets in the cluster
NOTE: The "kubelet-config-1.23" naming of the kubelet ConfigMap is deprecated. Once the UnversionedKubeletConfigMap feature gate graduates to Beta the default name will become just "kubelet-config". Kubeadm upgrade will handle this transition transparently.
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node k8s-master as control-plane by adding the labels: [node-role.kubernetes.io/master(deprecated) node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node k8s-master as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: d81z17.1cgfgwaee1l858ni
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
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

kubeadm join 172.26.95.187:6443 --token d81z17.1cgfgwaee1l858ni \
        --discovery-token-ca-cert-hash sha256:c4cfbe4dd5ac4e92b89a7d544a0a3f18d94a5382d947ba6e7f97f613d79a5027
```

**根据输出提示创建相关文件**

```shell
[root@k8s-master ~]# mkdir -p $HOME/.kube
[root@k8s-master ~]# cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
[root@k8s-master ~]# chown $(id -u):$(id -g) $HOME/.kube/config

```

**查看节点**

```shell
#查看节点，节点状态NotReady，是因为还没安装calcio网络插件
[root@k8s-master ~]# kubectl get nodes
NAME         STATUS     ROLES                  AGE   VERSION
k8s-master   NotReady   control-plane,master   23m   v1.23.6
[root@k8s-master ~]#
```

**查看pod**

```shell
#查看pod,coredns状态pending，是因为还没安装calcio网络插件
[root@k8s-master ~]# kubectl get pods -A
NAMESPACE     NAME                                 READY   STATUS    RESTARTS   AGE
kube-system   coredns-6d8c4cb4d-lrgkd              0/1     Pending   0          22m
kube-system   coredns-6d8c4cb4d-vc226              0/1     Pending   0          22m
kube-system   etcd-k8s-master                      1/1     Running   1          23m
kube-system   kube-apiserver-k8s-master            1/1     Running   1          23m
kube-system   kube-controller-manager-k8s-master   1/1     Running   1          23m
kube-system   kube-proxy-hdzdp                     1/1     Running   0          22m
kube-system   kube-scheduler-k8s-master            1/1     Running   1          23m
[root@k8s-master ~]#
```

**容器网络（CNI）部署**

```shell
#安装calcio，事先准备好calcio yaml文件
[root@k8s-master ~]# kubectl create -f calcio.yaml 
poddisruptionbudget.policy/calico-kube-controllers created
serviceaccount/calico-kube-controllers created
serviceaccount/calico-node created
configmap/calico-config created
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
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
clusterrole.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrole.rbac.authorization.k8s.io/calico-node created
clusterrolebinding.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrolebinding.rbac.authorization.k8s.io/calico-node created
daemonset.apps/calico-node created
deployment.apps/calico-kube-controllers created

#查看pod状态
[root@k8s-master ~]# kubectl get pods -A
NAMESPACE     NAME                                       READY   STATUS     RESTARTS   AGE
kube-system   calico-kube-controllers-7b8458594b-lv5zw   0/1     Pending    0          35s
kube-system   calico-node-qfx7s                          0/1     Init:1/3   0          35s
kube-system   coredns-6d8c4cb4d-bb2m7                    0/1     Pending    0          11m
kube-system   coredns-6d8c4cb4d-lfv66                    0/1     Pending    0          10m
kube-system   etcd-k8s-master                            1/1     Running    1          45m
kube-system   kube-apiserver-k8s-master                  1/1     Running    1          45m
kube-system   kube-controller-manager-k8s-master         1/1     Running    1          45m
kube-system   kube-proxy-hdzdp                           1/1     Running    0          44m
kube-system   kube-scheduler-k8s-master                  1/1     Running    1          45m

#再次查看发现pod已经running
[root@k8s-master ~]# kubectl get pods -A
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-7b8458594b-lv5zw   1/1     Running   0          112s
kube-system   calico-node-qfx7s                          1/1     Running   0          112s
kube-system   coredns-6d8c4cb4d-bb2m7                    1/1     Running   0          12m
kube-system   coredns-6d8c4cb4d-lfv66                    1/1     Running   0          12m
kube-system   etcd-k8s-master                            1/1     Running   1          46m
kube-system   kube-apiserver-k8s-master                  1/1     Running   1          46m
kube-system   kube-controller-manager-k8s-master         1/1     Running   1          46m
kube-system   kube-proxy-hdzdp                           1/1     Running   0          46m
kube-system   kube-scheduler-k8s-master                  1/1     Running   1          46m

#查看节点状态发现已经Ready
[root@k8s-master ~]# kubectl get nodes
NAME         STATUS   ROLES                  AGE   VERSION
k8s-master   Ready    control-plane,master   46m   v1.23.6
```

[calico.yaml](https://www.yuque.com/wojiaomht/hunxat/krhr3ya9ldbsi6xy?view=doc\_embed)

#### work节点加入集群

略。。。
