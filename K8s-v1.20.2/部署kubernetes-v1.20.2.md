# 01.部署k8s 1.20.2

## 组件版本

| 组件                    | 安装的版本 |
| ----------------------- | ---------- |
| kubeadm                 | 1.20.2     |
| kubectl                 | 1.20.2     |
| kubelet                 | 1.20.2     |
| kube-apiserver          | 1.20.2     |
| kube-controller-manager | 1.20.2     |
| kube-scheduler          | 1.20.2     |
| coredns                 | 1.7.0      |
| etcd                    | 3.4.13-0   |
| docker                  | 20.10.2    |
| calico                  | v3.17.1    |

## 环境准备

在所有节点都要操作

### 所有主机统一hosts

hosts统一如下

系统`CentOS 7.7+`以上

```
cat /etc/hosts
10.140.0.4 master
10.140.0.5 node01
10.140.0.6 node02
10.140.0.7 node03
```



### 所有机器升级内核（可选）

导入升级内核的yum源

```
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
```

查看可用版本 kernel-lt指长期稳定版 kernel-ml指最新版

```
 yum --disablerepo="*" --enablerepo="elrepo-kernel" list available
```

安装kernel-ml

```
yum --enablerepo=elrepo-kernel install kernel-ml kernel-ml-devel -y
```

### 设置启动项

查看系统上的所有可用内核

```
awk -F\' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg
```

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1594196209080-1a691d50-bd6d-4f77-a304-1d1b3155b109.png)

设置新的内核为grub2的默认版本

```
grub2-set-default 0
```



生成 grub 配置文件并重启

```
grub2-mkconfig -o /boot/grub2/grub.cfg
reboot
```



重启后

```
uname -r
```

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1594196209080-1a691d50-bd6d-4f77-a304-1d1b3155b109.png)

### 所有机器都关闭防火墙，swap，selinux



```
#关闭防火墙
systemctl disable --now firewalld

#关闭swap
swapoff -a
sed -ri '/^[^#]*swap/s@^@#@' /etc/fstab

#关闭selinux
setenforce 0
sed -ri '/^[^#]*SELINUX=/s#=.+$#=disabled#' /etc/selinux/config
```



### 所有机器yum更新



```
yum install epel-release -y

yum update -y
yum -y install  gcc bc gcc-c++ ncurses ncurses-devel cmake elfutils-libelf-devel openssl-devel flex* bison* autoconf automake zlib* fiex* libxml* ncurses-devel libmcrypt* libtool-ltdl-devel* make cmake  pcre pcre-devel openssl openssl-devel   jemalloc-devel tlc libtool vim unzip wget lrzsz bash-comp* ipvsadm ipset jq sysstat conntrack libseccomp conntrack-tools socat curl wget git conntrack-tools psmisc nfs-utils tree bash-completion conntrack libseccomp net-tools crontabs sysstat iftop nload strace bind-utils tcpdump htop telnet lsof 
```

### 所有机器都加载ipvs module

ipvs配置文件

```
cat > /etc/modules-load.d/ipvs.conf <<EOF
module=(
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
nf_conntrack_ipv4
br_netfilter
  )
for kernel_module in ${module[@]};do
    /sbin/modinfo -F filename $kernel_module |& grep -qv ERROR && echo $kernel_module >> /etc/modules-load.d/ipvs.conf || :
done
EOF
```



加载ipvs模块

```
systemctl daemon-reload
reboot
```



查询ipvs是否加载

```
$ lsmod | grep ip_vs
ip_vs_sh               12688  0 
ip_vs_wrr              12697  0 
ip_vs_rr               12600  11 
ip_vs                 145497  17 ip_vs_rr,ip_vs_sh,ip_vs_wrr
nf_conntrack          133095  7 ip_vs,nf_nat,nf_nat_ipv4,xt_conntrack,nf_nat_masquerade_ipv4,nf_conntrack_netlink,nf_conntrack_ipv4
libcrc32c              12644  3 ip_vs,nf_nat,nf_conntrack
```



### 所有机器都设置k8s系统参数



```
cat <<EOF > /etc/sysctl.d/k8s.conf
net.ipv6.conf.all.disable_ipv6 = 1           #禁用ipv6
net.ipv6.conf.default.disable_ipv6 = 1       #禁用ipv6
net.ipv6.conf.lo.disable_ipv6 = 1            #禁用ipv6
net.ipv4.neigh.default.gc_stale_time = 120   #决定检查过期多久邻居条目
net.ipv4.conf.all.rp_filter = 0              #关闭反向路由校验
net.ipv4.conf.default.rp_filter = 0          #关闭反向路由校验
net.ipv4.conf.default.arp_announce = 2       #始终使用与目标IP地址对应的最佳本地IP地址作为ARP请求的源IP地址
net.ipv4.conf.lo.arp_announce = 2            #始终使用与目标IP地址对应的最佳本地IP地址作为ARP请求的源IP地址
net.ipv4.conf.all.arp_announce = 2           #始终使用与目标IP地址对应的最佳本地IP地址作为ARP请求的源IP地址
net.ipv4.ip_forward = 1                      #启用ip转发功能
net.ipv4.tcp_max_tw_buckets = 5000           #表示系统同时保持TIME_WAIT套接字的最大数量
net.ipv4.tcp_syncookies = 1                  #表示开启SYN Cookies。当出现SYN等待队列溢出时，启用cookies来处理
net.ipv4.tcp_max_syn_backlog = 1024          #接受SYN同包的最大客户端数量
net.ipv4.tcp_synack_retries = 2              #活动TCP连接重传次数
net.bridge.bridge-nf-call-ip6tables = 1      #要求iptables对bridge的数据进行处理
net.bridge.bridge-nf-call-iptables = 1       #要求iptables对bridge的数据进行处理
net.bridge.bridge-nf-call-arptables = 1      #要求iptables对bridge的数据进行处理
net.netfilter.nf_conntrack_max = 2310720     #修改最大连接数
fs.inotify.max_user_watches=89100            #同一用户同时可以添加的watch数目
fs.may_detach_mounts = 1                     #允许文件卸载
fs.file-max = 52706963                       #系统级别的能够打开的文件句柄的数量
fs.nr_open = 52706963                        #单个进程可分配的最大文件数
vm.overcommit_memory=1                       #表示内核允许分配所有的物理内存，而不管当前的内存状态如何
vm.panic_on_oom=0                            #内核将检查是否有足够的可用内存供应用进程使用
vm.swappiness = 0                            #关闭swap
net.ipv4.tcp_keepalive_time = 600            #修复ipvs模式下长连接timeout问题,小于900即可
net.ipv4.tcp_keepalive_intvl = 30            #探测没有确认时，重新发送探测的频度
net.ipv4.tcp_keepalive_probes = 10           #在认定连接失效之前，发送多少个TCP的keepalive探测包
vm.max_map_count=524288                      #定义了一个进程能拥有的最多的内存区域
EOF
sysctl --system
```

### 所有机器都设置文件最大数



```
cat>/etc/security/limits.d/kubernetes.conf<<EOF
*       soft    nproc   131072
*       hard    nproc   131072
*       soft    nofile  131072
*       hard    nofile  131072
root    soft    nproc   131072
root    hard    nproc   131072
root    soft    nofile  131072
root    hard    nofile  131072
EOF
```



### 所有机器都设置docker 安装



#### docker yum



```
wget -P /etc/yum.repos.d/  https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```



#### 官方脚本检查



docker官方的内核检查脚本建议`(RHEL7/CentOS7: User namespaces disabled; add 'user_namespace.enable=1' to boot command line)`



```
grubby --args="user_namespace.enable=1" --update-kernel="$(grubby --default-kernel)"

#然后重启
reboot
```



#### docker安装



```
yum install docker-ce -y
```



#### 配置docker



```
cp /usr/share/bash-completion/completions/docker /etc/bash_completion.d/

mkdir -p /etc/docker/

cat > /etc/docker/daemon.json <<EOF
{
    "log-driver": "json-file",
    "exec-opts": ["native.cgroupdriver=systemd"],
    "log-opts": {
    "max-size": "100m",
    "max-file": "3"
    },
    "live-restore": true,
    "max-concurrent-downloads": 10,
    "max-concurrent-uploads": 10,
    "registry-mirrors": ["https://2lefsjdg.mirror.aliyuncs.com"],
    "storage-driver": "overlay2",
    "storage-opts": [
    "overlay2.override_kernel_check=true"
    ]
}
EOF
```



#### 启动docker



```
systemctl enable --now docker
```



## kubeadm部署



### 所有机器都设置kubeadm yum



在所有节点操作



```
cat <<EOF >/etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
EOF
```



### maser节点安装



```
yum install -y \
    kubeadm-1.20.2 \
    kubectl-1.20.2 \
    kubelet-1.20.2 \
    --disableexcludes=kubernetes && \
    systemctl enable kubelet
```



### node节点安装



```
yum install -y \
    kubeadm-1.20.2 \
    kubelet-1.20.2 \
    --disableexcludes=kubernetes && \
    systemctl enable kubelet
```



### kubeadm配置文件



在master节点操作



```
cat > /root/initconfig.yaml << EOF
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
imageRepository: registry.cn-hangzhou.aliyuncs.com/k8sxio  #镜像库
kubernetesVersion: v1.20.2  #安装的版本
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
networking: 
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12  #service ip段
  podSubnet: 10.244.0.0/16   #pod ip段
controlPlaneEndpoint: 10.140.0.4:6443  #apiserver ip
apiServer:
  timeoutForControlPlane: 4m0s
  extraArgs:
    authorization-mode: "Node,RBAC"
    runtime-config: api/all=true
    storage-backend: etcd3
    etcd-servers: https://10.140.0.4:2379  #etcd ip
  certSANs: #admin证书饱含的ip
  - 10.96.0.1
  - 127.0.0.1
  - localhost
  - 10.140.0.4
  - 10.140.0.5
  - 10.140.0.6
  - 10.140.0.7   
  - master
  - node01
  - node02
  - node03
  - kubernetes
  - kubernetes.default 
  - kubernetes.default.svc 
  - kubernetes.default.svc.cluster.local
  extraVolumes:
  - hostPath: /etc/localtime
    mountPath: /etc/localtime
    name: localtime
    readOnly: true
controllerManager:
  extraArgs:
    bind-address: "0.0.0.0"
    experimental-cluster-signing-duration: 867000h
  extraVolumes:
  - hostPath: /etc/localtime
    mountPath: /etc/localtime
    name: localtime
    readOnly: true
scheduler: 
  extraArgs:
    bind-address: "0.0.0.0"
  extraVolumes:
  - hostPath: /etc/localtime
    mountPath: /etc/localtime
    name: localtime
    readOnly: true
dns:
  type: CoreDNS
  imageRepository: registry.aliyuncs.com/k8sxio
  imageTag: 1.7.0
etcd:
  local:
    imageRepository: registry.aliyuncs.com/k8sxio
    imageTag: 3.4.13-0
    dataDir: /var/lib/etcd
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs
ipvs:
  excludeCIDRs: null
  minSyncPeriod: 0s
  scheduler: "rr"
  strictARP: false
  syncPeriod: 15s
iptables:
  masqueradeAll: true
  masqueradeBit: 14
  minSyncPeriod: 0s
  syncPeriod: 30s
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: "systemd"
failSwapOn: true
EOF
```



检查文件是否错误，忽略`warning`，错误的话会抛出error，没错则会输出到包含字符串`kubeadm join xxx`啥的



```
kubeadm init --config /root/initconfig.yaml --dry-run
```



预先拉取镜像

```
kubeadm config images pull --config /root/initconfig.yaml
```



### 部署master



在master节点操作



```
kubeadm init --config /root/initconfig.yaml --upload-certs
```

复制kubectl的kubeconfig，kubectl的kubeconfig路径默认是`~/.kube/config`



```
mkdir -p $HOME/.kube

sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config

sudo chown $(id -u):$(id -g) $HOME/.kube/config
```



init的yaml信息实际上会存在集群的configmap里，我们可以随时查看，该yaml在其他node和master join的时候会使用到



```
kubectl -n kube-system get cm kubeadm-config -o yaml
```



##### 设置kubectl的补全脚本



```
yum -y install bash-comp*

source <(kubectl completion bash)

echo 'source <(kubectl completion bash)' >> ~/.bashrc
```



### 部署node

在node节点执行`kubeadm join xxx`啥的

```
kubeadm join 10.140.0.4:6443 --token yp2f7o.488nmqxytjz2csl7 \
    --discovery-token-ca-cert-hash sha256:5724b5ac1636ff69a90cc25bf0776ee97012ce9354f526a36c9efe6289b29968
```



### 打标签



role只是一个label，可以打label，想显示啥就`node-role.kubernetes.io/xxxx`

```
kubectl get nodes
```

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1610697308182-a4b6ddae-d099-4d25-a2f1-d90e9ec025d4.png)

```
 kubectl label node node01 node-role.kubernetes.io/node=""
 kubectl label node node02 node-role.kubernetes.io/node=""
 kubectl label node node03 node-role.kubernetes.io/node=""
```

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1610697387264-6bd0c210-9fdd-4a2f-b8dd-92aa7fa20323.png)



## 部署网络插件Calico



没有网络插件，所有节点都是notready



在master上操作

```
 curl https://docs.projectcalico.org/manifests/calico.yaml -O
```

[📎calico.yaml](https://www.yuque.com/attachments/yuque/0/2021/yaml/1176682/1610699021667-15e9ce78-2ddc-4350-b994-acb728363515.yaml)

修改文件中的ip，默认是注释的，改成上面配置文件中的podSubnet地址

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1610698274212-42e306e8-2770-4b0d-be8c-7d3c610cfada.png)

```
kubectl apply -f calico.yaml
```

## 查看部署情况

```
kubectl get pods -A -o wide
```

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1610698467488-43dafade-d4ab-49bc-85ef-6837d2fe1ddd.png)