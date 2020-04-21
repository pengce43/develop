======================================== 系统环境配置 ========================================

1、关闭防火墙:
ansible master -m shell -a 'systemctl stop firewalld && systemctl disable firewalld'
systemctl disable --now firewalld
systemctl disable --now dnsmasq
systemctl disable --now NetworkManager

2、关闭selinux:
ansible master -m shell -a 'sed -i 's/^SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux'
ansible master -m shell -a 'setenforce 0'
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
sed -i 's/^SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux

3、关闭swap分区:
 swapoff -a && sysctl -w vm.swappiness=0
  ansible master -m shell -a 'swapoff -a && sysctl -w vm.swappiness=0'
 注释swap挂载
  sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
 ansible master -m shell -a "sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab"
 查看：
 ansible master -m shell -a "sed -n '/swap/p' /etc/fstab"

4、设置主机名:
192.168.31.10
hostnamectl set-hostname k8s-master01
192.168.31.11
hostnamectl set-hostname k8s-master02
192.168.31.12
hostnamectl set-hostname k8s-master03
配置/etc/hosts:
192.168.31.10 k8s-master01
192.168.31.11 k8s-master02
192.168.31.12 k8s-master03

5、设置内核优化:
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
vm.swappiness=0
vm.overcommit_memory=1
vm.panic_on_oom=0
fs.may_detach_mounts = 1
fs.inotify.max_user_watches=89100
fs.file-max=52706963
fs.nr_open=52706963
net.netfilter.nf_conntrack_max=2310720
EOF

modprobe br_netfilter
sysctl --system

命令
ansible master -m copy -a 'src=/etc/sysctl.d/k8s.conf dest=/etc/sysctl.d/k8s.conf'
ansible master -m shell -a 'modprobe br_netfilter'
ansible master -m shell -a 'sysctl -p /etc/sysctl.d/k8s.conf'

6、时间同步:
ansible master -m yum -a "name=ntpdate state=latest" 
ansible master -m cron -a "name='k8s cluster crontab' minute=*/30 hour=* day=* month=* weekday=* job='ntpdate time7.aliyun.com >/dev/null 2>&1'"
ansible master -m shell -a "ntpdate time7.aliyun.com"

7、安装docker-ce:
ansible master -m shell -a 'yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo'
ansible master -m shell -a 'yum install docker-ce-18.09.5 -y'
ansible master -m shell -a 'systemctl start docker && systemctl enable docker'
mkdir /etc/docker
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

8、加载ipvs模块
# 加载 ipvs 相关内核模块
modprobe ip_vs
modprobe ip_vs_rr
modprobe ip_vs_wrr
modprobe ip_vs_sh
modprobe nf_conntrack_ipv4
modprobe br_netfilter

# 加入开机启动中
cat <<EOF >>/etc/rc.d/rc.local
modprobe ip_vs
modprobe ip_vs_rr
modprobe ip_vs_wrr
modprobe ip_vs_sh
modprobe nf_conntrack_ipv4
modprobe br_netfilter
EOF

chmod +x /etc/rc.d/rc.local


9、设置内核参数
vi /etc/sysctl.d/k8s.conf
添加：
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1

10、安装ipvs相关管理软件
yum install ipset ipvsadm -y
reboot

安装docker
cat > /etc/docker/daemon.json <<EOF
{
 "exec-opts": ["native.cgroupdriver=systemd"],
 "log-driver": "json-file",
 "log-opts": {
  "max-size": "100m"
 }
}
EOF
======================================== 基本组件安装 ========================================
1、配置kubeadm的阿里yum源
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

2、安装kubeadm、kubelet、kubectl,注意这里安装版本v1.14.3:
yum install -y kubelet-1.14.3 kubeadm-1.14.3 kubectl-1.14.3
yum install -y kubelet-1.14.1 kubeadm-1.14.1 kubectl-1.14.1 cri-tools-1.12.0 kubernetes-cni-0.7.5 ipvsadm --disableexcludes=kubernetes
DOCKER_CGROUPS=$(docker info | grep 'Cgroup' | cut -d' ' -f4)
echo $DOCKER_CGROUPS
cat > /etc/sysconfig/kubelet <<EOF
KUBELET_EXTRA_ARGS="--cgroup-driver=$DOCKER_CGROUPS --pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google_containers/pause-amd64:3.1"
EOF

# 开机启动并 start now，之后 kubelet 启动是失败的，每隔几秒钟会自动重启，这是在等待 kubeadm 告诉它要做什么。
systemctl enable --now kubelet

======================================== 集群初始换 ========================================
1、安装keepalived和haproxy
yum install haproxy keepalived -y
mkdir /etc/haproxy
systemctl enable --now haproxy
systemctl enable --now keepalived

2、生成初始化配置文件
cd /root/K8S
kubeadm config print init-defaults > kubeadm-config.yaml

配置文件解释：
controlPlaneEndpoint：为vip地址和haproxy监听端口6443
imageRepository:由于国内无法访问google镜像仓库k8s.gcr.io，这里指定为阿里云镜像仓库registry.aliyuncs.com/google_containers
podSubnet:指定的IP地址段与后续部署的网络插件相匹配，这里需要部署flannel插件，所以配置为10.244.0.0/16
mode: ipvs:最后追加的配置为开启ipvs模式。

3、拉取镜像
kubeadm  config images pull  --config kubeadm-config.yaml  # 通过阿里源预先拉镜像

4、节点 初始化
kubeadm init --config=kubeadm-config.yaml --experimental-upload-certs | tee kubeadm-init.log
启动报错解决：https://blog.csdn.net/boling_cavalry/article/details/91306095

node节点加入集群
如果不幸，你忘记了上面的命令并且找不到了，可以通过 kubeadm token create --print-join-command 来获取

查看kubelet的日志：
journalctl -f -u kubelet
