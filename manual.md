## Step การติดตั้ง Kubernetes cluster
* ติดตั้ง Docker และ Kubernetes ในทุก ๆ  node ทั้ง Master และ Worker node
* สร้าง Master node และ Cluster
* สร้าง Worker node และทำการ join เข้า Cluster

## 1. ติดตั้ง Docker บน Ubuntu
* [Reference](https://docs.docker.com/engine/install/ubuntu/)

Check docker wis working !!
```
$docker version

Client: Docker Engine - Community
 Version:           20.10.14
 API version:       1.41
 Go version:        go1.16.15
 Git commit:        a224086
 Built:             Thu Mar 24 01:48:02 2022
 OS/Arch:           linux/amd64
 Context:           default
 Experimental:      true

Server: Docker Engine - Community
 Engine:
  Version:          20.10.14
  API version:      1.41 (minimum version 1.12)
  Go version:       go1.16.15
  Git commit:       87a90dc
  Built:            Thu Mar 24 01:45:53 2022
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.5.11
  GitCommit:        3df54a852345ae127d1fa3092b95168e4a88e2f8
 runc:
  Version:          1.0.3
  GitCommit:        v1.0.3-0-gf46b6ba
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0
```

## 2. ติดตั้ง Kubernetes บน Ubuntu

Pre-installation
```
$sudo swapoff -a
$sudo vi /etc/fstab  

#/swap.img  none    swap    sw  0    0 

$sudo apt update
$sudo apt install -y apt-transport-https

$curl -s \
    https://packages.cloud.google.com/apt/doc/apt-key.gpg |\
    sudo apt-key add -

$sudo touch /etc/apt/sources.list.d/kubernetes.list
$echo \
    "deb http://apt.kubernetes.io/ kubernetes-xenial main" |\
    sudo tee -a /etc/apt/sources.list.d/kubernetes.list

```

Installation
```
$sudo apt update   
$sudo apt install -y kubeadm  
```

Check
```
$kubeadm --help
$kubectl --help 
$kubelet --help

```

## 3. สร้าง Cluster และ Master node

Initial Master node
* [Solved problen when can't not initial](https://stackoverflow.com/questions/52119985/kubeadm-init-shows-kubelet-isnt-running-or-healthy)

### More error
```
$sudo kubeadm init --pod-network-cidr=10.244.0.0/16

[init] Using Kubernetes version: v1.24.0
[preflight] Running pre-flight checks
error execution phase preflight: [preflight] Some fatal errors occurred:
	[ERROR CRI]: container runtime is not running: output: time="2022-05-12T17:40:15Z" level=fatal msg="getting status of runtime: rpc error: code = Unimplemented desc = unknown service runtime.v1alpha2.RuntimeService"
, error: exit status 1
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
To see the stack trace of this error execute with --v=5 or higher
```
Solution
```
$sudo rm /etc/containerd/config.toml
$sudo systemctl restart containerd
```


Initial cluster
```
$sudo kubeadm init --pod-network-cidr=10.244.0.0/16

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

kubeadm join 159.223.87.252:6443 --token x5mnpi.zwuvvgi3uhheyhbi \
	--discovery-token-ca-cert-hash sha256:c00b886e00f16eb7f0e947c1cf7ad4f6c7f4002cecf5f3b5a097c5295ba19b3e
```

Start cluster
```
$mkdir -p $HOME/.kube
$sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Install Pods network for cluster
```
$kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml  

$kubectl get pods --all-namespaces

NAMESPACE     NAME                             READY   STATUS    RESTARTS   AGE
kube-system   coredns-64897985d-jkhn7          1/1     Running   0          5m46s
kube-system   coredns-64897985d-kj4vz          1/1     Running   0          5m46s
kube-system   etcd-master                      1/1     Running   0          6m1s
kube-system   kube-apiserver-master            1/1     Running   0          6m1s
kube-system   kube-controller-manager-master   1/1     Running   0          6m3s
kube-system   kube-flannel-ds-klcqs            1/1     Running   0          3m41s
kube-system   kube-flannel-ds-r8mlm            1/1     Running   0          93s
kube-system   kube-proxy-c88rk                 1/1     Running   0          5m47s
kube-system   kube-proxy-jgblt                 1/1     Running   0          93s
kube-system   kube-scheduler-master            1/1     Running   0          6m1s
```

Check node in cluster
```
$kubectl get nodes -w

master   NotReady   control-plane,master   2m40s   v1.23.5
...
master   Ready      control-plane,master   2m46s   v1.23.5
```

## 4. ทำการ join worker node เข้า cluster

Copy from step 3
```
$sudo kubeadm join 159.223.87.252:6443 --token x5mnpi.zwuvvgi3uhheyhbi \
	--discovery-token-ca-cert-hash sha256:c00b886e00f16eb7f0e947c1cf7ad4f6c7f4002cecf5f3b5a097c5295ba19b3e


This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

Check nodes in Master node
```
$kubectl get nodes

NAME     STATUS   ROLES                  AGE     VERSION
master   Ready    control-plane,master   5m22s   v1.23.5
worker   Ready    <none>                 51s     v1.23.5
```

## Step 5 :: Deploy ผ่าน Master node
```
$kubectl apply -f https://k8s.io/examples/pods/simple-pod.yaml
pod/nginx created

$kubectl get pods -o wide
NAME    READY   STATUS    RESTARTS   AGE   IP           NODE     NOMINATED NODE   READINESS GATES
nginx   1/1     Running   0          33s   10.244.1.2   worker   <none>           <none>
```

Ready to go next ...
