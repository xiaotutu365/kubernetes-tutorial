## Kubernetes的安装

配置集群时注意：时间同步

### 环境准备
#### 三台机器
kube1-master:192.168.3.166
kube2-node1:192.168.3.242
kube3-node2:192.168.3.190

#### 配置主机名(/etc/hostname)
* 192.168.3.166(作为master)
```shell
> vim /etc/hostname
kube1-master
```
* 192.168.3.242(作为node1)
```shell
> vim /etc/hostname
kube2-node1
```
* 192.168.3.190(作为node2)
```shell
> vim /etc/hostname
kube3-node2
```

#### 配置hosts(/etc/hosts)
* 192.168.3.166(作为master)
```shell
> vim /etc/hosts
192.168.3.166 kube1-master
192.168.3.242 kube2-node1
192.168.3.190 kube3-node2
```
* 192.168.3.242(作为node1)
```shell
> vim /etc/hosts
192.168.3.166 kube1-master
192.168.3.242 kube2-node1
192.168.3.190 kube3-node2
```
* 192.168.3.190(作为node2)
```shell
> vim /etc/hosts
192.168.3.166 kube1-master
192.168.3.242 kube2-node1
192.168.3.190 kube3-node2
```

配置完成后，重启计算机，可以相互ping，如果可以ping通，则表示配置成功。

### 配置master
#### 配置仓库
* 获取docker repo
```shell
> cd /etc/yum.repos.d
> wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
> # 查询docker-ce可用版本
> yum list docker-ce --showduplicates|sort -r
```
* 获取kubernetes
``` shell
> vim kubernetes.repo
[kubernetes]
name=Kubernetes Repo
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
enabled=1
> :wq（保存退出）
> yum repolist
```

#### 开始安装
```shell
> yum install docker-ce kubelet kubeadm kubectl
```
这样虽然可以安装，但是后续会出现docker和kubernetes版本不兼容的问题，kubernetes的当前stable版本是V1.12.2，docker的当前stable版本是V18.09，但是kubernetes V1.12.2并不兼容docker V18.09，而只支持docker V18.06，所以需要指定docker的版本，如下：
```shell
> yum install docker-ce-18.06.1.ce
```
安装完成后需要配置国内镜像，否则会出现下载速度很慢或者有的镜像无法获取的现象，配置镜像的方法如下：
```shell
> cd /etc/docker
{
  "registry-mirrors": ["https://krg1ud1i.mirror.aliyuncs.com"]
}
> systemctl daemon-reload
> systemctl restart docker
```

安装过程中会出现gpgcheck错误的问题，可以用如下方法解决：
```shell
> wget https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
> rpm --import yum-key.gpg
> wget https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
> rpm --import rpm-package-key-gpg
```
继续安装，则安装成功！

#### 修改相关配置文件
##### 修改docker配置文件
```shell
> vim /usr/lib/systemd/system/docker.service
# 在ExecStart上面增加如下语句
Environment="HTTPS_PROXY=http://www.ik8s.io:10080" 
Environment="NO_PROXY=127.0.0.0/8,172.20.0.0/16"
```
修改完成后，启动docker
```shell
> systemctl daemon-reload
> systemctl start docker
```
```shell
> cat /proc/sys/net/bridge/bridge-nf-call-ip6tables
> 1
> cat /proc/sys/net/bridge/bridge-nf-call-iptables
> 1
```
确保这两个文件存储的都是1，就不用改了。
到这里，docker的配置修改完成。

##### 修改kubernetes配置文件
首先使用如下命令查看：
```shell
> rpm -ql kubelet
> cat /etc/sysconfig/kubelet
```
启动kubelet
```shell
# 启动kubelet
> systemctl start kubelet
# 查看kubelet状态
> systemctl status kubelet
# 为失败：code=exited,status=255
```
使用如下命令查看：
```shell
> tail /var/log/messages
```
打印如下错误信息：
```text
Nov 10 23:31:53 bigdata kubelet: F1110 23:31:53.115865    4150 server.go:190] failed to load Kubelet config file /var/lib/kubelet/config.yaml, error failed to read kubelet config file "/var/lib/kubelet/config.yaml", error: open /var/lib/kubelet/config.yaml: no such file or directory
```
这个时候，先停止kubelet：
```shell
> systemctl stop kubelet
```
设置开机自启动：
```shell
> systemctl enable kubelet
> systemctl enable docker
```
##### kubeadm 初始化
配置忽略kubelet：
```shell
> vim /etc/sysconfig/kubelet
KUBELET_EXTRA_ARGS="--fail-swap-on=false"
```
**执行初始化命令前，需要先拉取镜像，因为这些镜像无法从国内docker镜像加速器获取**
```shell
> kubeadm config images list
```
显示需要拉取的镜像：
```text
k8s.gcr.io/kube-apiserver:v1.12.2
k8s.gcr.io/kube-controller-manager:v1.12.2
k8s.gcr.io/kube-scheduler:v1.12.2
k8s.gcr.io/kube-proxy:v1.12.2
k8s.gcr.io/pause:3.1
k8s.gcr.io/etcd:3.2.24
k8s.gcr.io/coredns:1.2.2
```
这里采用先从国内镜像加速器获取这些镜像，然后使用tag标记为kubernetes的镜像tag：
* 先从国内镜像加速器获取镜像：
```shell
docker pull mirrorgooglecontainers/kube-apiserver:v1.12.2
docker pull mirrorgooglecontainers/kube-controller-manager:v1.12.2
docker pull mirrorgooglecontainers/kube-scheduler:v1.12.2
docker pull mirrorgooglecontainers/kube-proxy:v1.12.2
docker pull mirrorgooglecontainers/pause:3.1
docker pull mirrorgooglecontainers/etcd:3.2.24
docker pull coredns/coredns:1.2.2
```
* 再标记为kubernetes的镜像tag
```shell
docker tag mirrorgooglecontainers/kube-apiserver:v1.12.2 k8s.gcr.io/kube-apiserver:v1.12.2
docker tag mirrorgooglecontainers/kube-controller-manager:v1.12.2 k8s.gcr.io/kube-controller-manager:v1.12.2
docker tag mirrorgooglecontainers/kube-scheduler:v1.12.2 k8s.gcr.io/kube-scheduler:v1.12.2
docker tag mirrorgooglecontainers/kube-proxy:v1.12.2 k8s.gcr.io/kube-proxy:v1.12.2
docker tag mirrorgooglecontainers/pause:3.1 k8s.gcr.io/pause:3.1
docker tag mirrorgooglecontainers/etcd:3.2.24 k8s.gcr.io/etcd:3.2.24
docker tag coredns/coredns:1.2.2 k8s.gcr.io/coredns:1.2.2
```

执行初始化：
```shell
> kubeadm init --kubernetes-version=v1.12.2 --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12 --ignore-preflight-errors=Swap
```
执行初始化命令前，需要启动`kubelet`：
```shell
> systemctl start kubelet
```
初始化完成后，会输出一些信息，这些信息在后面配置node集群中是必须的，所以需要保存，如下：
```text
To start using your cluster, you need to run the following as a regular user:
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You can now join any number of machines by running the following on each node
as root:
  kubeadm join 192.168.3.166:6443 --token sjh8aj.5nashad8b75bwkh8 --discovery-token-ca-cert-hash sha256:e31b4a3086ad933e7ca25bbf8ca711c10257c2a3b022649c8747ff182e305a95
```
初始化完成后，需要使用flannel配置网络：
```shell
> kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
接着执行如下命令：
```shell
> mkdir -p $HOME/.kube
> sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
```
使用下列命令查看节点信息：
```shell
> kubectl get nodes
> kubectl get pods -n kube-system -o wide
```

### 配置nodes
在node1和node2上同样需要配置`docker`,`kubeadm`,`kubelet`,`kubectl`，首先将master上的一些资源文件复制到node1和node2上：
```shell
> scp docker-ce.repo kubernetes.repo rpm-package-key.gpg root@kube2-node1:/etc/yum.repos.d
> scp docker-ce.repo kubernetes.repo rpm-package-key.gpg root@kube3-node2:/etc/yum.repos.d
```
然后按照在master上安装的方式，分别安装`docker`,`kubeadm`,`kubelet`,`kubectl`。
