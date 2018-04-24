## 三节点k8s高可用集群部署
### 实验环境：
1. 3台centos 1611版本虚拟机，mini安装。Linux localhost 3.10.0-514.el7.x86_64 #1 SMP Tue Nov 22 16:42:41 UTC 2016 x86_64 x86_64 x86_64 GNU/Linux
2. > docker version
>
>>Client:
>>
>> Version:         1.13.1
>>
>> API version:     1.26
>>
>> Package version: <unknown>
>>
>> Go version:      go1.8.3
>>
>> Git commit:      774336d/1.13.1
>>
>> Built:           Wed Mar  7 17:06:16 2018
>>
>> OS/Arch:         linux/amd64
>> Server:
>>
>> Version:         1.13.1
>>
>> API version:     1.26 (minimum version 1.12)
>>
>> Package version: <unknown>
>>
>> Go version:      go1.8.3
>>
>> Git commit:      774336d/1.13.1
>>
>> Built:           Wed Mar  7 17:06:16 2018
>>
>> OS/Arch:         linux/amd64
>>
>> Experimental:    false
>>
3. etcd Version: 3.1.13
4. kubeadm，kubelet，kubectl，kube-cni版本如下：
>> -rw-r--r--  1 root    root    18176678 Mar 30 00:08 5844c6be68e95a741f1ad403e5d4f6962044be060bc6df9614c2547fdbf91ae5-kubelet-1.10.0-> 0.x86_64.rpm
>>
>> -rw-r--r--  1 root    root    17767206 Mar 30 00:07 8b0cb6654a2f6d014555c6c85148a5adc5918de937f608a30b0c0ae955d8abce-kubeadm-1.10.0-0.x86_64.rpm
>>
>> -rw-r--r--  1 root    root     7945458 Mar 30 00:07 9ff2e28300e012e2b692e1d4445786f0bed0fd5c13ef650d61369097bfdd0519-kubectl-1.10.0-> 0.x86_64.rpm
>>
>> -rw-r--r--  1 root    root     9008838 Mar  5 21:56 fe33057ffe95bfae65e2f269e1b05e99308853176e24a4d027bc082b471a07c0-kubernetes-cni-0.6.0-0.x86_64.rpm
>>
5. k8s网络组件：flannel:v0.10.0-amd64
6. 实验网络规划如下：
> host1 172.18.0.154/22 
>
> host2 172.18.0.155/22
>
> host3 172.18.0.156/22
>
> VIP 172.18.0.192/22
>
7. [视频教程](https://pan.baidu.com/s/1XVagd765eGacuoR_cgesiQ)
### 安装步骤：
0. [下载脚本](https://pan.baidu.com/s/1oK7PRLeeYHrouNCRgIQlcQ)
1. 在3台主机中执行基础环境配置脚本 base-env-config.sh
2. 在主机1执行脚本 host1-base-env.sh
3. 在主机2执行脚本 host2-base-env.sh
4. 在主机3执行脚本 host3-base-env.sh
5. 在host1主机执行如下命令
> [root@host1~]# ll /etc/etcd/ssl/
>
> total 12
>
> -rw-r--r--. 1 root root 1387 Mar 30 16:04 ca.pem
>
> -rw-------. 1 root root 1675 Mar 30 16:04 etcd-key.pem
>
> -rw-r--r--. 1 root root 1452 Mar 30 16:04 etcd.pem
>
> [root@host1 ~]# scp -r /etc/etcd/ssl root@172.18.0.155:/etc/etcd/
>
> [root@host1 ~]# scp -r /etc/etcd/ssl root@172.18.0.156:/etc/etcd/
>
6. 在3台主机中分别执行脚本 etcd.sh
7. 查看keepalived状态。如下图所示，则三个节点机之间心跳正常。
>systemctl status keepalived
8. 查看etcd运行状态。
在host1,host2,host3分别执行如下命令：
> etcdctl  --endpoints=https://${NODE_IP}:2379  --ca-file=/etc/etcd/ssl/ca.pem  --cert-file=/etc/etcd/ssl/etcd.pem  --key-file=/etc/etcd/ssl/etcd-key.pem cluster-health
>
9. 在3台主机上安装kubeadm,kubelet,kubctl,docker
> yum install kubelet kubeadm kubectl kubernetes-cni docker -y
>
10. 在3台主机禁用docker启动项SELinux
> sed -i 's/--selinux-enabled/--selinux-enabled=false/g' /etc/sysconfig/docke
>
11. 在3台主机的kubelet配置文件中添加如下参数
> sed -i '9a\Environment="KUBELET_EXTRA_ARGS=--pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/osoulmate/pause-amd64:3.0"' /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
>
12. 在3台主机添加docker加速器配置（可选）
> cat <<EOF > /etc/docker/daemon.json
>> {
>>> "registry-mirrors": ["https://wcmntott.mirror.aliyuncs.com"] 
>> }
> EOF
>
13. 在3台主机分别执行以下命令
> systemctl daemon-reload
>
> systemctl enable docker && systemctl restart docker
>
> systemctl enable kubelet && systemctl restart kubelet
>
14. 在3台主机中分别执行kubeadmconfig.sh生成配置文件config.yaml
15. 在host1主机中首先执行kubeadm初始化操作
> 命令如下：
> kubeadm init --config config.yaml
>
16. 执行初始化后操作
> mkdir -p $HOME/.kube
>
> sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
>
> sudo chown $(id -u):$(id -g) $HOME/.kube/config
>
17. 将主机host1中kubeadm初始化后生成的证书拷贝至host2,host3相应目录下
> scp -r /etc/kubernetes/pki root@172.18.0.155:/etc/kubernetes/
>
> scp -r /etc/kubernetes/pki root@172.18.0.156:/etc/kubernetes/
> 
18. 为主机host1安装网络组件 podnetwork【这里选用flannel】
> kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
> 
> systemctl stop kubelet
> 
> systemctl restart docker
> 
> docker pull registry.cn-hangzhou.aliyuncs.com/osoulmate/flannel:v0.10.0-amd64
>
> systemctl start kubelet
> 
19. 在host2,host3上执行kubeadm初始化操作
> kubeadm init --config config.yaml
> 
> kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
> 
> systemctl stop kubelet
>
> systemctl restart docker
>
> docker pull registry.cn-hangzhou.aliyuncs.com/osoulmate/flannel:v0.10.0-amd64
>
> systemctl start kubelet
>
20. 查看集群各节点状态
> [root@localhost ~]# kubectl get nodes
>
> NAME      STATUS    ROLES     AGE       VERSION
>
> host1     Ready     master    5m        v1.10.0
>
> host2     Ready     master    1m        v1.10.0
>
> host3     Ready     master    1m        v1.10.0
>
> [root@localhost ~]# kubectl get po --all-namespaces
>
> NAMESPACE     NAME                            READY     STATUS    RESTARTS   AGE
>
> kube-system   coredns-7997f8864c-k9dcx        1/1       Running   0          5m
>
> kube-system   coredns-7997f8864c-sv9rv        1/1       Running   0          5m
>
> kube-system   kube-apiserver-host1            1/1       Running   1          4m
>
> kube-system   kube-apiserver-host2            1/1       Running   0          1m
>
> kube-system   kube-apiserver-host3            1/1       Running   0          1m
>
> kube-system   kube-controller-manager-host1   1/1       Running   1          4m
>
> kube-system   kube-controller-manager-host2   1/1       Running   0          1m
>
> kube-system   kube-controller-manager-host3   1/1       Running   0          1m
>
> kube-system   kube-flannel-ds-88tz5           1/1       Running   0          1m
>
> kube-system   kube-flannel-ds-g9dpj           1/1       Running   0          2m
>
> kube-system   kube-flannel-ds-h58tp           1/1       Running   0          1m
>
> kube-system   kube-proxy-6fsvq                1/1       Running   1          5m
>
> kube-system   kube-proxy-g8xnb                1/1       Running   1          1m
>
> kube-system   kube-proxy-gmqv9                1/1       Running   1          1m
>
> kube-system   kube-scheduler-host1            1/1       Running   1          5m
> 
> kube-system   kube-scheduler-host2            1/1       Running   1          1m
> 
> kube-system   kube-scheduler-host3            1/1       Running   0          1m
21. 高可用验证
> 将host1关机，在host3上执行
>
> while true; do  sleep 1; kubectl get node;date; done
>
> 在host2上观察keepalived是否已切换为主状态。


