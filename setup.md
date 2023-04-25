# k8s+docker配置部署方式
## 1.前期环境需求
>CentOS7虚拟机三台，镜像文件linux内核版本3.10，CPU>=2，内存>=2G，硬盘空间>=20GB  
**CentOS7镜像源url**= http://mirrors.aliyun.com/centos/7/isos/x86_64/  
建议使用CentOS7-DVD2009.iso   
我使用的虚拟机软件为*VirtualBox*   
>安装过程中，建议为虚拟机添加NAT网卡和HOST-ONLY网卡  

**如下图所示**  
![](/src/img/network%20adaptor.jpg "虚拟机网卡组成")

## 2.网络配置
>安装好虚拟机之后，会发现CentOS连不上网，无法使用`yum -y install`来下载东西，这是因为网卡都没有配置ip地址，可以使用`ip addr`命令来查看

![](/src/img/netcfg.jpg "cfg")
>其中，在我的环境中enp0s3是NAT的网卡，enp0s8是HOST-ONLY网卡。没有ip地址可以使用`dhclient`先启动DHCP获取IP地址，你可以再输入一次`ip addr`进行验证。这里需要修改一下网卡的cfg文件，否则每次重启都需要输入一次`dhclient`命令，修改的点主要在`ONBOOT=YES`，建议使用`cat xxxx |grep ONBOOT -n`定位行数，再在vi编辑器中`:行数`跳转定位。HOST-ONLY网卡我配置了**static**模式，固定了ip，这是为了后续使用此ip部署k8s。  

![](/src/img/ipcfg.jpg)

## 3.配置操作
>关闭SElinux  
``` bash
setenforce 0  
sed -i  '/^SELINUX/ s/enforcing/disabled/' /etc/selinux/config
```

>禁用且关闭SWAP
```bash
swapoff -a
sed -i 's/.\*swap.\*/#&/' /etc/fstab
```

>关闭防火墙
```bash
systemctl stop firewalld
systemctl disable firewalld
```
>修改网卡配置
```bash
cat >> /etc/sysctl.conf << eof
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
eof
modprobe  br_netfilter 
sysctl -p
```
>启动内核模块
```bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
cut -f1 -d " "  /proc/modules | grep -e ip_vs -e nf_conntrack_ipv4
cat >>/etc/sysconfig/modules/ipvs.modules <<eof
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
eof
```
>配置hosts文件
```bash
cat >> /etc/hosts <<eof
xxx.xxx.xxx.xxx master
xxx.xxx.xxx.xxx node1
xxx.xxx.xxx.xxx node2
eof
```
>设置主机名称
```bash
hostnamectl set-hostname xxxx
#hostnamectl set-hostname master
#hostnamectl set-hostname node1
#hostnamectl set-hostname node2
```
>重启查看更改
```bash
reboot
```
## 4.安装 kubelet && kubeadm && kubectl
### **添加Kubernetes  `yum` 源**
>这里使用aliyun的 `yum` 源
```bash
cat >>/etc/yum.repos.d/kubernetes.repo <<eof
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
	http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
eof
```
### **安装kubelet && kubeadm && kubectl**
>建议使用1.16.3版本，高版本可能会导致一些错误
```bash
yum install kubelet-1.16.3 kubeadm-1.16.3 kubectl 1.16.3 -y
```
>启动kubelet服务
```bash
systemctl enable kubelet
systemctl start kubelet
```
>这时候使用`systemctl status kubelet`应当报错`Error(255)`

## 5.安装及配置Docker

### **安装Docker**
```bash
yum install -y yum-utils
yum-config-manager \
        --add-repo \
        http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum install -y docker-ce docker-ce-cli containerd.io 
```
# **注意**
## 如果报错`Error downloading packages: 3:docker-ce-23.0.4-1.el7.x86_64: [Errno 256]`  
## 建议使用`vi\vim`编辑`etc/yum.repos.d/docker-ce.repo`文件  
## 修改docker-ce-test下的`enabled=0`为`enabled=1`
### **配置docker**
### **1.配置cgroup-driver为systemd**
```bash
cat <<EOF > /etc/docker/daemon.json
{
   "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF
systemctl daemon-reload 
systemctl enable docker 
systemctl restart docker
```
### **2.拉取所需镜像**
>验证所需镜像版本
```bash
kubeadm config images list
I0425 15:06:49.371066   26647 version.go:251] remote version is much newer: v1.27.1; falling back to: stable-1.16
k8s.gcr.io/kube-apiserver:v1.16.15
k8s.gcr.io/kube-controller-manager:v1.16.15
k8s.gcr.io/kube-scheduler:v1.16.15
k8s.gcr.io/kube-proxy:v1.16.15
k8s.gcr.io/pause:3.1
k8s.gcr.io/etcd:3.3.15-0
k8s.gcr.io/coredns:1.6.2
```
>拉取镜像
```bash
docker pull kubeimage/kube-apiserver-amd64:v1.16.3
docker pull kubeimage/kube-controller-manager-amd64:v1.16.3
docker pull kubeimage/kube-scheduler-amd64:v1.16.3
docker pull kubeimage/kube-proxy-amd64:v1.16.3
docker pull kubeimage/pause-amd64:3.1
docker pull kubeimage/etcd-amd64:3.3.15-0
docker pull coredns/coredns:1.6.2
```
>对拉取到的镜像打tag
```
docker tag kubeimage/kube-apiserver-amd64:v1.16.3  k8s.gcr.io/kube-apiserver:v1.16.3
docker tag kubeimage/kube-controller-manager-amd64:v1.16.3  k8s.gcr.io/kube-controller-manager:v1.16.3
docker tag kubeimage/kube-scheduler-amd64:v1.16.3  k8s.gcr.io/kube-scheduler:v1.16.3
docker tag kubeimage/kube-proxy-amd64:v1.16.3 k8s.gcr.io/kube-proxy:v1.16.3
docker tag kubeimage/pause-amd64:3.1 k8s.gcr.io/pause:3.1
docker tag kubeimage/etcd-amd64:3.3.15-0 k8s.gcr.io/etcd:3.3.15-0
docker tag coredns/coredns:1.6.2 k8s.gcr.io/coredns:1.6.2
```
## 6.配置kubernetes及安装网络插件Calico
# **master节点操作**
>kubeadm init
```bash
kubeadm init --pod-network-cidr=10.244.0.0/16 \
  --kubernetes-version=v1.16.3 \
  --apiserver-advertise-address=192.168.118.7 #替换成master节点IP
```
>输出结果  

![](/src/img/kubeadminit.jpg "输出结果")
>然后根据提示完成下述步骤
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
>安装网络插件Calico
```bash
kubectl apply -f https://docs.projectcalico.org/v3.10/manifests/calico.yaml
```
# **node节点操作**
>加入集群
```bash
kubeadm join 192.168.118.7:6443 --token s4yj6d.c3pgcngafcrtmf38 \
    --discovery-token-ca-cert-hash sha256:17cf843df163918e964cb2396362389c9be073a9e7445d3bc5c92bd855710251 
```
>结果

![](/src/img/node.jpg)

## master节点进行查看  
>使用`kubectl get nodes -o wide`  

![](/src/img/mastercheck1.jpg)
>使用`kubectl describe node node1`

![](/src/img/masterdescribe.jpg)

# 后续可完善部分
- 尝试安装flannel网络插件的配置方式  
- 尝试安装1.22版本以上的kubenetes（可能会影响docker对其的支持)  

# 参考网址
## 1.https://blog.hungtcs.top/2019/11/27/23-K8S%E5%AE%89%E8%A3%85%E8%BF%87%E7%A8%8B%E7%AC%94%E8%AE%B0/  
## 2.https://blog.csdn.net/weibo1230123/article/details/121680887
## 3.https://www.cnblogs.com/Sunzz/p/15184167.html

