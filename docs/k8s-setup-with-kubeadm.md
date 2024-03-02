# Setup kubernetes cluster from scratch using kubeadm

## Install container runtime (containerd) => all nodes
* Download containerd
```shell
wget https://github.com/containerd/containerd/releases/download/v1.7.13/containerd-1.7.13-linux-amd64.tar.gz

tar Cxzvf /usr/local containerd-1.7.13-linux-amd64.tar.gz 
```

* Create systemd service `/usr/local/lib/systemd/system/containerd.service`
```shell
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target local-fs.target

[Service]
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd

Type=notify
Delegate=yes
KillMode=process
Restart=always
RestartSec=5

# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity

# Comment TasksMax if your systemd version does not supports it.
# Only systemd 226 and above support this version.
TasksMax=infinity
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target
```
* Reload systemd
```shell
systemctl daemon-reload
systemctl enable --now containerd
```
* Install runc
```shell
wget https://github.com/opencontainers/runc/releases/download/v1.1.12/runc.amd64

install -m 755 runc.amd64 /usr/local/sbin/runc
```
* Install CNI plugin
```shell
wget https://github.com/containernetworking/plugins/releases/download/v1.4.0/cni-plugins-linux-amd64-v1.4.0.tgz

mkdir -p /opt/cni/bin
tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.4.0.tgz 
```

* Containerd configuration

Generate default configuration :
```shell
mkdir -p /etc/containerd/
containerd config default > /etc/containerd/config.toml
```
change cgroup driver to systemd :

```
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
```


## Preparing the hosts => all nodes

1. Disable swap on the nodes

Disable swap with the command bellow and permanently in `/etc/fstab`
```shell
swapoff -a 
```

2. Enable IPv4 forwarding using iptables

```shell
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```

3. Enable overlay network and bridge netfilter 

```shell
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
sysctl --system
```

4. Disable SELinux 
```shell
# Set SELinux in permissive mode (effectively disabling it)
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```
5. Configure k8s artifacts repo

```shell
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.28/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.28/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
```
6. Install kubectl kubelet and kubeadm

```shell
sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet
```

## Cluster initialization => control plan node

* Configure kubelet with systemd

```shell
# config.yaml
controlPlaneEndpoint: 192.168.0.10
apiServer:
  extraArgs:
    authorization-mode: Node,RBAC
    advertise-address: 192.168.0.10
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.k8s.io
kind: ClusterConfiguration
kubernetesVersion: v1.28.7
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
  podSubnet: 10.244.0.0/16
scheduler: {}
---
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: systemd
```

* Run kubeadm init

```shell
kubeadm init --config config.yaml
```
* Configure kubectl

```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
* Joinning nodes

```shell
#You can now join any number of control-plane nodes by copying certificate authorities
#and service account keys on each node and then running the following as root:

kubeadm join 10.0.0.10:6443 --token j3wd1v.a77wbxn7mbz80xyc \
--discovery-token-ca-cert-hash sha256:15d679f9e290b7e353a4c6b70bc97414807e7758268f03dbf87e9e0efd50f8c5 \
--control-plane 

#Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.0.0.10:6443 --token j3wd1v.a77wbxn7mbz80xyc \
	--discovery-token-ca-cert-hash sha256:15d679f9e290b7e353a4c6b70bc97414807e7758268f03dbf87e9e0efd50f8c5 

```

## Install CNI : cilium => control plan node

* Install cilium cli

```shell
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```

* Install cilium in the cluster

```shell
cilium install --version 1.15.1
```

* Check connectivity test
```shell
cilium connectivity test
```
