# Install Kubernetes by kubeadm on Loongarch64

## Letting iptables see bridged traffic
Make sure that the br_netfilter module is loaded. This can be done by running lsmod | grep br_netfilter. To load it explicitly call sudo modprobe br_netfilter.

As a requirement for your Linux Node's iptables to correctly see bridged traffic, you should ensure net.bridge.bridge-nf-call-iptables is set to 1 in your sysctl config, e.g.

```shell
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

## Turn off swap
Turn off swap by:
```shell
sudo swapoff -a 
```

## Install Container runtimes(Docker)
1. On each of your nodes, install the Docker for your Linux distribution as per Install Docker Engine.

2. Configure the Docker daemon, in particular to use systemd for the management of the container’s cgroups.
```shell
sudo mkdir -p /etc/docker
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "insecure-registries" : [ "0.0.0.0/0" ],
  "registry-mirrors": ["https://cr.loongnix.cn"],
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
```

3. Restart Docker and enable on boot:
```shell
sudo systemctl enable docker
sudo systemctl daemon-reload
sudo systemctl restart docker
```

## Installing kubeadm, kubelet and kubectl
```shell
cat <<EOF | sudo tee /etc/yum.repos.d/Loongnix-kubernetes.repo
#
# Loongnix-kubernetes.repo
#

[loongnix-kubernetes]
name=Loongnix server $releasever - kubernetes
baseurl=http://pkg.loongnix.cn:8080/loongnix-server/$releasever/cloud/$basearch/release/kubernetes/
gpgcheck=0
enabled=8
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-LOONGNIX
module_hotfixes=1
EOF

# Set SELinux in permissive mode (effectively disabling it)
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

sudo systemctl enable --now kubelet
```

## Install Kubernetes
To initialize the control-plane node run:
```shell
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --kubernetes-version=v1.20.0 --image-repository=cr.loongnix.cn/kubernetes
```

`kubeadm init` first runs a series of prechecks to ensure that the machine is ready to run Kubernetes. These prechecks expose warnings and exit on errors. `kubeadm init` then downloads and installs the cluster control plane components. This may take several minutes. After it finishes you should see:
```shell
[kubelet] Creating a ConfigMap "kubelet-config-1.20" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node loongson1 as control-plane by adding the labels "node-role.kubernetes.io/master=''" and "node-role.kubernetes.io/control-plane='' (deprecated)"
[mark-control-plane] Marking the node loongson1 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: tr1qqs.ymw7bc1pjumug0is
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

kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>

```


To make kubectl work for your non-root user, run these commands, which are also part of the kubeadm init output:
```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```


## Installing a Pod network add-on(Flannel)
You can install a Pod network add-on, for instance Flannel, with the following command on the control-plane node or a node that has the kubeconfig credentials:
```shell
kubectl apply -f https://github.com/flomesh-io/loong64/blob/main/kubernetes/flannel.yaml
```


After all of these steps are finished successfully, you should see all Kubernetes components are UP and RUNNING:
```shell
[test@loongson1 kubeadm]$ kubectl get po -n kube-system
NAMESPACE     NAME                                READY   STATUS    RESTARTS   AGE
kube-system   coredns-7cb7cc6b47-fqksw            1/1     Running   0          2m30s
kube-system   coredns-7cb7cc6b47-kmrmp            1/1     Running   0          2m30s
kube-system   etcd-loongson1                      1/1     Running   0          2m36s
kube-system   kube-apiserver-loongson1            1/1     Running   0          2m36s
kube-system   kube-controller-manager-loongson1   1/1     Running   0          2m36s
kube-system   kube-flannel-ds-bc9zg               1/1     Running   0          29s
kube-system   kube-proxy-npvx8                    1/1     Running   0          2m30s
kube-system   kube-scheduler-loongson1            1/1     Running   0          2m36s
```

## Control plane node isolation
By default, your cluster will not schedule Pods on the control-plane node for security reasons. If you want to be able to schedule Pods on the control-plane node, for example for a single-machine Kubernetes cluster for development, run:
```shell
kubectl taint nodes --all node-role.kubernetes.io/master-
```


With output looking something like:
```shell
node "loongson1" untainted
taint "node-role.kubernetes.io/master:" not found
taint "node-role.kubernetes.io/master:" not found
```

This will remove the node-role.kubernetes.io/master taint from any nodes that have it, including the control-plane node, meaning that the scheduler will then be able to schedule Pods everywhere.
