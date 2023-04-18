---
title: "kubeadm部署高可用kubernetes集群"
date: 2021-08-15T17:22:42+08:00
lastmod: 2021-08-15T17:22:42+08:00
draft: false
description: "kubeadm部署高可用kubernetes集群"
tags: ["kubeadm", "k8s", "haproxy", "keepalived"]
categories: ["kubernetes"]
author: "wanzi"

comment: true   # 关闭评论
toc: true       # 关闭文章目录
autoCollapseToc: true
contentCopyright: '<a rel="license noopener" href="https://creativecommons.org/licenses/by-nc-nd/4.0/" target="_blank">CC BY-NC-ND 4.0</a>'
reward: true     # 关闭打赏
mathjax: true    # 打开 mathjax
---
为了后便后期验证私有化部署
，最近内网环境需要快速搭建一套k8s集群，由于之前对于规模比较大的集群，我一般采用kubeasz和kubespray来搞定，这次对于小环境集群，直接用kubeadm部署会更加高效。

下面记录kubeadm部署过程：

集群节点：
```yaml
192.168.1.206 sd-cluster-206 node
192.168.1.207 sd-cluster-207 master,etcd
192.168.1.208 sd-cluster-208 master,etcd,haproxy,keepalived
192.168.1.209 sd-cluster-209 master,etcd,haproxy,keepalived
```

镜像版本：
```shell
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.18.3
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.18.3
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.18.3
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.18.3
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.6.5
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.3-0
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.2
docker pull registry.cn-shanghai.aliyuncs.com/gcr-k8s/flannel:v0.14.0
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/nginx-ingress-controller:v0.48.1
```

## 一、基本环境设置

### 1、安装docker并增加hosts
```shell
yum install -y yum-utils device-mapper-persistent-data lvm2 git
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum install docker-ce -y
systemctl  start docker
systemctl  enable docker
systemctl  status docker
```

### 2、配置hosts
```shell
cat  >>  /etc/hosts << hhhh
192.168.1.207 sd-cluster-207
192.168.1.208 sd-cluster-208
192.168.1.209 sd-cluster-209
hhhh
```

### 3、禁用防火墙并设置selinux
```shell
systemctl stop firewalld
systemctl disable firewalld
setenforce 0
sed -i 's/SELINUX=permissive/SELINUX=disabled/' /etc/sysconfig/selinux
```

### 4、关闭swap
Kubernetes 1.8开始要求关闭系统的Swap，如果不关闭，默认配置下kubelet将无法启动。方法一,通过kubelet的启动参数–fail-swap-on=false更改这个限制。方法二,关闭系统的Swap。

```shell
# swapoff -a
```
修改/etc/fstab文件，注释掉SWAP的自动挂载，使用free -m确认swap已经关闭。

### 5、安装所有节点的kubeadm和kubelet、ipvsadm
参考：https://developer.aliyun.com/mirror/kubernetes
```shell
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
yum install -y --nogpgcheck  kubelet-1.18.3 kubeadm-1.18.3 ipvsadm 
systemctl enable kubelet && systemctl start kubelet
```
ps: 由于官网未开放同步方式, 可能会有索引gpg检查失败的情况, 这时请用 yum install -y --nogpgcheck kubelet kubeadm kubectl 安装


增加ipvsadm，如果重新开机，需要重新加载（可以写在 /etc/rc.local 中开机自动加载）

```shell
cat <<EOF >> /etc/rc.local
modprobe ip_vs
modprobe ip_vs_rr
modprobe ip_vs_wrr
modprobe ip_vs_sh
modprobe nf_conntrack_ipv4
EOF
chmod +x /etc/rc.local
```

## 二、keepalived配合haproxy实现负载均衡器高可用
```shell
yum -y install keepalived haproxy
```
### 1、keepalived配置
192.168.1.208配置
```yaml
vrrp_script chk_ha {
	script "killall -0 haproxy"  
	interval 2 		
	weight -20 					
}

vrrp_instance VI_1 {
	state MASTER
    interface bond0
	virtual_router_id 33
	mcast_src_ip 192.168.1.208
	priority 100 		
	nopreempt 			
	advert_int 1 				
	authentication {
		auth_type PASS
		auth_pass hak8s
	}

	track_script {
		chk_ha			## 执行haproxy监测
	}

	virtual_ipaddress {
		192.168.1.205/24		##VIP 
	}
}
```
192.168.1.209配置
```yaml
vrrp_script chk_ha {
	script "killall -0 haproxy"
	interval 2
	weight -20
}
vrrp_instance VI_1 {
	state BACKUP
	interface bond0
	virtual_router_id 33
	mcast_src_ip 192.168.1.209
	priority 90
	advert_int 1
	authentication {
		auth_type PASS
		auth_pass hak8s
	}
	track_script {
		chk_ha
	}
	virtual_ipaddress {
		192.168.1.205/24
	}
}
```
### 2、haproxy配置
208和209两个master的haproxy配置一样：
```yaml
global
    log /dev/log local0
    maxconn 65535
    chroot /var/lib/haproxy
    pidfile /var/run/haproxy.pid
    stats socket /var/lib/haproxy/stats
    stats timeout 30s
    user haproxy
    group haproxy
    daemon

defaults
    log global
    retries 3
    option redispatch
    option dontlognull
    timeout connect 5000
    timeout client  50000
    timeout server  50000

frontend k8s-apiserver
    bind *:8443
    mode tcp
    balance roundrobin
    server sd-cluster-04 192.168.1.207:6443 weight 5 check inter 2000 rise 2 fall 3
    server sd-cluster-05 192.168.1.208:6443 weight 3 check inter 2000 rise 2 fall 3
    server sd-cluster-06 192.168.1.209:6443 weight 3 check inter 2000 rise 2 fall 3
```
至此，haproxy高可用已经搭建完成，验证方法，直接关闭208的keepalived服务，看vip是否飘逸到209机器上。


## 三、kubeadm初始化k8s集群

### 1、初始化第一台master

kubeadm-config.yaml配置
```yaml
#kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta2
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 192.168.1.208
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: sd-cluster-208
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controlPlaneEndpoint: "192.168.1.208:8443"
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: v1.18.3
networking:
  dnsDomain: cluster.local
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.96.0.0/16
scheduler: {}
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs
```

```shell
# kubeadm init --config kubeadm-config.yaml --upload-certs
W0828 03:02:37.249435   17451 configset.go:202] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
[init] Using Kubernetes version: v1.18.3
[preflight] Running pre-flight checks
	[WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [sd-cluster-208 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.1.208 192.168.1.205]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [sd-cluster-208 localhost] and IPs [192.168.1.208 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [sd-cluster-208 localhost] and IPs [192.168.1.208 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[endpoint] WARNING: port specified in controlPlaneEndpoint overrides bindPort in the controlplane address
[kubeconfig] Writing "admin.conf" kubeconfig file
[endpoint] WARNING: port specified in controlPlaneEndpoint overrides bindPort in the controlplane address
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[endpoint] WARNING: port specified in controlPlaneEndpoint overrides bindPort in the controlplane address
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[endpoint] WARNING: port specified in controlPlaneEndpoint overrides bindPort in the controlplane address
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
W0828 03:02:43.787797   17451 manifests.go:225] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
[control-plane] Creating static Pod manifest for "kube-scheduler"
W0828 03:02:43.789895   17451 manifests.go:225] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 23.015352 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.18" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Storing the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
[upload-certs] Using certificate key:
0adac55426d376e72f21ec3aee2465f754e78810700843cb75c270be26eaeaf1
[mark-control-plane] Marking the node sd-cluster-208 as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node sd-cluster-208 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: abcdef.0123456789abcdef
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[endpoint] WARNING: port specified in controlPlaneEndpoint overrides bindPort in the controlplane address
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join 192.168.1.205:8443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:7400f57a033f7527c80a4b015b54b4b6a88ccb2184ab9a6e39b709fb56e10486 \
    --control-plane --certificate-key 0adac55426d376e72f21ec3aee2465f754e78810700843cb75c270be26eaeaf1

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.1.205:8443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:7400f57a033f7527c80a4b015b54b4b6a88ccb2184ab9a6e39b709fb56e10486
```

配置master节点kubeconfig

```shell
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

### 2、新master节点加入集群

```shell
# kubeadm join 192.168.1.205:8443 --token abcdef.0123456789abcdef \
>     --discovery-token-ca-cert-hash sha256:7400f57a033f7527c80a4b015b54b4b6a88ccb2184ab9a6e39b709fb56e10486 \
>     --control-plane --certificate-key 0adac55426d376e72f21ec3aee2465f754e78810700843cb75c270be26eaeaf1
[preflight] Running pre-flight checks
	[WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[preflight] Running pre-flight checks before initializing the new control plane instance
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[download-certs] Downloading the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [sd-cluster-209 localhost] and IPs [192.168.1.209 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [sd-cluster-209 localhost] and IPs [192.168.1.209 127.0.0.1 ::1]
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [sd-cluster-209 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.1.209 192.168.1.205]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Valid certificates and keys now exist in "/etc/kubernetes/pki"
[certs] Using the existing "sa" key
[kubeconfig] Generating kubeconfig files
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[endpoint] WARNING: port specified in controlPlaneEndpoint overrides bindPort in the controlplane address
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
W0828 03:12:57.996811   48328 manifests.go:225] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
W0828 03:12:58.009390   48328 manifests.go:225] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
[control-plane] Creating static Pod manifest for "kube-scheduler"
W0828 03:12:58.011318   48328 manifests.go:225] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
[check-etcd] Checking that the etcd cluster is healthy
[kubelet-start] Downloading configuration for the kubelet from the "kubelet-config-1.18" ConfigMap in the kube-system namespace
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...
[etcd] Announced new etcd member joining to the existing etcd cluster
[etcd] Creating static Pod manifest for "etcd"
[etcd] Waiting for the new etcd member to join the cluster. This can take up to 40s
{"level":"warn","ts":"2021-08-28T03:13:15.215-0400","caller":"clientv3/retry_interceptor.go:61","msg":"retrying of unary invoker failed","target":"passthrough:///https://192.168.1.209:2379","attempt":0,"error":"rpc error: code = DeadlineExceeded desc = context deadline exceeded"}
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[mark-control-plane] Marking the node sd-cluster-209 as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node sd-cluster-209 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]

This node has joined the cluster and a new control plane instance was created:

* Certificate signing request was sent to apiserver and approval was received.
* The Kubelet was informed of the new secure connection details.
* Control plane (master) label and taint were applied to the new node.
* The Kubernetes control plane instances scaled up.
* A new etcd member was added to the local/stacked etcd cluster.

To start administering your cluster from this node, you need to run the following as a regular user:

	mkdir -p $HOME/.kube
	sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
	sudo chown $(id -u):$(id -g) $HOME/.kube/config

Run 'kubectl get nodes' to see this node join the cluster.
```

### 3、新node节点加入集群
```shell
kubeadm join 192.168.1.205:8443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:7400f57a033f7527c80a4b015b54b4b6a88ccb2184ab9a6e39b709fb56e10486
```

### 4、安装flannel网络组件
```shell
# wget http://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
# sed -i 's#quay.io/coreos/flannel:v0.14.0#registry.cn-shanghai.aliyuncs.com/gcr-k8s/flannel:v0.14.0#g' kube-flannel.yml
# kubectl apply -f kube-flannel.yml -n kube-system
podsecuritypolicy.policy/psp.flannel.unprivileged created
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.apps/kube-flannel-ds created
# kubectl get pod -n kube-system |grep flannel
kube-flannel-ds-gtdl8                    1/1     Running   0          24s
kube-flannel-ds-jtndz                    1/1     Running   0          24s
kube-flannel-ds-nd79b                    1/1     Running   0          24s
```
### 5、安装ingress
```shell
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/nginx-ingress-controller:v0.48.1

wget https://github.com/kubernetes/ingress-nginx/releases/download/helm-chart-3.35.0/ingress-nginx-3.35.0.tgz
tar zxvf helm-chart-3.35.0.tar.gz
cd ingress-nginx
vim values.yaml  #更新镜像为上面的国内阿里镜像版本

#helm install ingress-nginx --namespace ingress-nginx  . -n ingress-nginx
NAME: ingress-nginx
LAST DEPLOYED: Tue Aug 31 17:02:28 2021
NAMESPACE: ingress-nginx
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The ingress-nginx controller has been installed.
Get the application URL by running these commands:
  export HTTP_NODE_PORT=$(kubectl --namespace ingress-nginx get services -o jsonpath="{.spec.ports[0].nodePort}" ingress-nginx-controller)
  export HTTPS_NODE_PORT=$(kubectl --namespace ingress-nginx get services -o jsonpath="{.spec.ports[1].nodePort}" ingress-nginx-controller)
  export NODE_IP=$(kubectl --namespace ingress-nginx get nodes -o jsonpath="{.items[0].status.addresses[1].address}")

  echo "Visit http://$NODE_IP:$HTTP_NODE_PORT to access your application via HTTP."
  echo "Visit https://$NODE_IP:$HTTPS_NODE_PORT to access your application via HTTPS."

An example Ingress that makes use of the controller:

  apiVersion: networking.k8s.io/v1beta1
  kind: Ingress
  metadata:
    annotations:
      kubernetes.io/ingress.class: nginx
    name: example
    namespace: foo
  spec:
    rules:
      - host: www.example.com
        http:
          paths:
            - backend:
                serviceName: exampleService
                servicePort: 80
              path: /
    # This section is only required if TLS is to be enabled for the Ingress
    tls:
        - hosts:
            - www.example.com
          secretName: example-tls

If TLS is enabled for the Ingress, a Secret containing the certificate and key must also be provided:

  apiVersion: v1
  kind: Secret
  metadata:
    name: example-tls
    namespace: foo
  data:
    tls.crt: <base64 encoded cert>
    tls.key: <base64 encoded key>
  type: kubernetes.io/tls
```
对于新建Ingress参考如下设置：
```shell
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-nginx
spec:
  rules:
    - host: "my-nginx.test.com"
      http:
        paths:
          - path: /
            backend:
              serviceName: my-nginx
              servicePort: 8080
```

至此，kubeadm部署kubernetes高可用集群完成

