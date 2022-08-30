---
title: Kubernetes installation Howto
weight: -20
---

Install Kubernetes Cluster on Ubuntu 22.04 with kubeadm.

<!--more-->

{{< toc >}}

## Upgrade your Ubuntu servers

```Shell
sudo apt update
sudo apt -y full-upgrade
[ -f /var/run/reboot-required ] && sudo reboot -f
```

## Install kubelet, kubeadm and kubectl

1. Add kubernetes repository to the server

   ```Shell
   sudo apt install curl apt-transport-https -y
   curl -fsSL  https://packages.cloud.google.com/apt/doc/apt-key.gpg|sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/k8s.gpg
   curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
   echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
   ```

2. Install required package

   ```Shell
   sudo apt update
   sudo apt install wget curl vim git kubelet kubeadm kubectl -y
   sudo apt-mark hold kubelet kubeadm kubectl
   ```

3. Installation confirmation.

   ```Shell
   $ kubectl version --client && kubeadm version
   WARNING: This version information is deprecated and will be replaced with the output from kubectl version --short.  Use --output=yaml|json to get the full version.
   Client Version: version.Info{Major:"1", Minor:"25", GitVersion:"v1.25.0", GitCommit:"a866cbe2e5bbaa01cfd5e969aa3e033f3282a8a2", GitTreeState:"clean", BuildDate:"2022-08-23T17:44:59Z", GoVersion:"go1.19", Compiler:"gc", Platform:"linux/amd64"}
   Kustomize Version: v4.5.7
   kubeadm version: &version.Info{Major:"1", Minor:"25", GitVersion:"v1.25.0", GitCommit:"a866cbe2e5bbaa01cfd5e969aa3e033f3282a8a2", GitTreeState:"clean", BuildDate:"2022-08-23T17:43:25Z", GoVersion:"go1.19", Compiler:"gc", Platform:"linux/amd64"}
   ```
4. Disable Swap Space
   ```Shell
   sudo swapoff -a 
   ```
5. Check if swap has been disabled by running the free command.
   ```Shell
   $ free -h
                  total        used        free      shared  buff/cache   available
   Mem:           966Mi       189Mi       127Mi       0.0Ki       649Mi       615Mi
   Swap:             0B          0B          0B
   ```
6. Enable kernel modules 
   ```Shell
   sudo modprobe overlay
   sudo modprobe br_netfilter
   ```
7. Add settings to sysctl
   ```Shell
   sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
   net.bridge.bridge-nf-call-ip6tables = 1
   net.bridge.bridge-nf-call-iptables = 1
   net.ipv4.ip_forward = 1
   EOF
   ```
8. Reload sysctl
   ```Shell
   sudo sysctl --system
   ```

## Install Container runtime (Master and Worker nodes) ##

To run containers in Pods, Kubernetes uses a **container runtime**. Supported container runtimes are:

- Docker
- CRI-O
- Containerd

{{< hint type=note >}}
**Info**\
**Dockershim** has been removed from the Kubernetes project as of release 1.24.
You need to install a container runtime into each node in the cluster so that Pods can run there. 
Kubernetes 1.25 requires that you use a runtime that conforms with the Container Runtime Interface (CRI).
{{< /hint >}}

### Install Docker runtime ###

   ```Shell
   # Add repo and Install packages
   sudo apt update
   sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
   curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
   sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
   sudo apt update
   sudo apt install -y containerd.io docker-ce docker-ce-cli

   # Create required directories
   sudo mkdir -p /etc/systemd/system/docker.service.d
   # Create daemon json config file
   sudo tee /etc/docker/daemon.json <<EOF
   {
      "exec-opts": ["native.cgroupdriver=systemd"],
      "log-driver": "json-file",
      "log-opts": {
      "max-size": "100m"
   },
      "storage-driver": "overlay2"
   }
   EOF

   # Start and enable Services
   sudo systemctl daemon-reload 
   sudo systemctl restart docker
   sudo systemctl enable docker

   # Configure persistent loading of modules
   sudo tee /etc/modules-load.d/k8s.conf <<EOF
   overlay
   br_netfilter
   EOF

   # Ensure you load modules
   sudo modprobe overlay
   sudo modprobe br_netfilter

   # Set up required sysctl params
   sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
   net.bridge.bridge-nf-call-ip6tables = 1
   net.bridge.bridge-nf-call-iptables = 1
   net.ipv4.ip_forward = 1
   EOF
   ```

{{< hint type=note >}}
We proceed with the container runtime as **Docker CE**.
{{< /hint >}}

### Install Mirantis cri-dockerd as Docker Engine shim for Kubernetes ###

1. Prepare to update
   ```Shell
   sudo apt update
   sudo apt install git wget curl
   # get the latest release version
   VER=$(curl -s https://api.github.com/repos/Mirantis/cri-dockerd/releases/latest|grep tag_name | cut -d '"' -f 4|sed 's/v//g')
   echo $VER
   ```
2. Download the archive file from Github cri-dockerd releases page
   ```Shell
   wget https://github.com/Mirantis/cri-dockerd/releases/download/v${VER}/cri-dockerd-${VER}.amd64.tgz
   tar xvf cri-dockerd-${VER}.amd64.tgz
   sudo mv cri-dockerd/cri-dockerd /usr/local/bin/
   # Validate 
   cri-dockerd --version
   ```
3. Configure systemd units for cri-dockerd:
   ```Shell
   wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.service
   wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.socket
   sudo mv cri-docker.socket cri-docker.service /etc/systemd/system/
   sudo sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service
   
4. Start and enable the services
   ```Shell
   sudo systemctl daemon-reload
   sudo systemctl enable cri-docker.service
   sudo systemctl enable --now cri-docker.socket
   ```

5. Confirm the service is now running:
   
   ```Shell
   $ systemctl status cri-docker.socket
   ```
6. Configure the kubelet to use cri-dockerd
   ```Shell
   sudo kubeadm config images pull --cri-socket /run/cri-dockerd.sock 
   ```
## Bootstrap Control plane (This is for single node controller) ##
1. Initialise kubernetes cluster.
   ```Shell
      sudo kubeadm init \
            --pod-network-cidr=10.244.0.0/16 \
            --cri-socket /run/cri-dockerd.sock \
            --ignore-preflight-errors=NumCPU,Mem
   ```
2. To start using your cluster, you need to run the following as a regular user:
   ```Shell
   mkdir -p $HOME/.kube
   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config
   ```
3. Check Kubernetes cluster status.
   ```Shell
   $ kubectl get nodes
   NAME              STATUS     ROLES           AGE    VERSION
   ip-172-31-30-23   NotReady   control-plane   2m1s   v1.25.0
   ```
4. Initialize control plane (run on first master node)
   ```Shell
   lsmod | grep br_netfilter
   sudo systemctl enable kubelet
   ```
5. Check cluster info.
   ```Shell
   $ kubectl cluster-info
   Kubernetes control plane is running at https://172.31.30.23:6443
   CoreDNS is running at https://172.31.30.23:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

   To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
   ```

## Install Kubernetes network plugin ##
1. Download flannel config file.

   ```Shell
   wget https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
   ```
2. Verify below section in the file and ensure it matches ip range mentioned above.

   ```Shell
   .......
   .......
   net-conf.json: |
      {
         "Network": "10.244.0.0/16",
         "Backend": {
         "Type": "vxlan"
      }
   .......
   .......
   ```
3. Check kube flannel status.
   ```Shell
   $ kubectl get pods -n kube-flannel
   NAME                    READY   STATUS    RESTARTS   AGE
   kube-flannel-ds-zqqqx   1/1     Running   0          92s
   ```
4. Verify ip range.
   ```Shell
   $ ip r
   default via 172.31.16.1 dev eth0 proto dhcp src 172.31.30.23 metric 100 
   10.244.0.0/24 dev cni0 proto kernel scope link src 10.244.0.1 
   172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown 
   172.31.0.2 via 172.31.16.1 dev eth0 proto dhcp src 172.31.30.23 metric 100 
   172.31.16.0/20 dev eth0 proto kernel scope link src 172.31.30.23 metric 100 
   172.31.16.1 dev eth0 proto dhcp scope link src 172.31.30.23 metric 100 
   ```
5. Now the cluster will be ready to serve.
   ```Shell
   $ kubectl get nodes
   NAME              STATUS   ROLES           AGE   VERSION
   ip-172-31-30-23   Ready    control-plane   14m   v1.25.0
   ```
6. Get the cluster details to check.
   ```Shell
   $ kubectl get nodes -o wide
   NAME              STATUS   ROLES           AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION    CONTAINER-RUNTIME
   ip-172-31-30-23   Ready    control-plane   13m   v1.25.0   172.31.30.23   <none>        Ubuntu 22.04.1 LTS   5.15.0-1017-aws   docker://20.10.17
   ```

## Add worker nodes to the k8 cluster ##
1. Run below command on the master/control plane node.
   ```Shell
   kubeadm token create --print-join-command
   ```
   Output
   ```Shell
   kubeadm join 172.31.30.23:6443 --token 4lw1ho.uycbcyuw08rma2kb --discovery-token-ca-cert-hash sha256:3952b82c082f32b38a487cf26a99698223540fad81e8fc99e57e6fb28d35d3c9 
   ```
2. Run the above command in output within worker node. Now the cluster will show as below.
   ```Shell
   $ kubectl get nodes
   NAME              STATUS   ROLES           AGE     VERSION
   ip-172-31-28-98   Ready    <none>          22s     v1.25.0
   ip-172-31-30-23   Ready    control-plane   4h26m   v1.25.0
   ```    

<!--
{{< hint type=note >}}
**Info**\
Keep in mind this method is not recommended and needs some extra steps to get it working.
If you want to use the Theme as submodule keep in mind that your build process need to
run the described steps as well.
{{< /hint >}}
-->


