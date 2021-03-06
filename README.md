# Build Bare Metal Kubernetes Dualstack Calico Metallb RaspberryPI 4 Ubuntu Server 64bit. (64bit required!!!)

[![GitHub issues](https://img.shields.io/github/issues/trackhe/Raspberry64bitKubernetesServerDualstack.svg?style=flat-square)](https://GitHub.com/trackhe/Raspberry64bitKubernetesServerDualstack/issues/)
[![GitHub pull-requests](https://img.shields.io/github/issues-pr/trackhe/Raspberry64bitKubernetesServerDualstack?style=flat-square)](https://GitHub.com/trackhe/Raspberry64bitKubernetesServerDualstack/pull/)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg?style=flat-square)](https://github.com/Trackhe/Raspberry64bitKubernetesServerDualstack/pulls)
[![Maintenance](https://img.shields.io/badge/Maintained%3F-yes-brightgreen.svg?style=flat-square)](https://GitHub.com/trackhe/Raspberry64bitKubernetesServerDualstack/graphs/commit-activity)

**Required:**
Running RaspberryPI 4 with the Ubuntu Server 64bit from here.

Image: [ubuntu/server64](https://ubuntu.com/download/raspberry-pi/thank-you?version=20.04.1&architecture=arm64+raspi)
new versions can be found here: [ubuntu/server/raspberrypi](https://ubuntu.com/download/raspberry-pi) download the 64bit version. (64bit required!!!)

**copy the image on your SD card with:**
Raspberry Pi Imager [Imager](https://www.raspberrypi.org/software/)

Before you start your pi copy the "ssh" file on the SDCard.
download: [ssh/file](https://raw.githubusercontent.com/Trackhe/Raspberry64bitKubernetesServerDualstack/master/ssh)

[Another Guide for installing Ubuntu Server arm64 on Pi (and connect with W-Lan)](https://linuxize.com/post/how-to-install-ubuntu-on-raspberry-pi/)

--- First Steps after boot Pi. ---
Connect with an SSH client to your Pi/'s

login with username `ubuntu` and password `ubuntu`.
```
sudo apt update && sudo apt -y dist-upgrade
```

install your dependencies and tools.
```
sudo apt -y install net-tools dphys-swapfile git
```

Downgrade Fireall to legacy.
```
wget https://raw.githubusercontent.com/theAkito/rancher-helpers/master/scripts/debian-buster_fix.sh && \
chmod +x debian-buster_fix.sh && \
sudo ./debian-buster_fix.sh
```

change your timezone. you can see your actually time on the rpi with `timedatectl` and set with
```
sudo dpkg-reconfigure tzdata
```
if you want to use it in your script automaticly use   `timedatectl set-timezone 'Europe/Berlin'` here is a list [timezone/list](https://gist.github.com/adamgen/3f2c30361296bbb45ada43d83c1ac4e5)


configure `gpu_mem`.
```
cat <<EOF | sudo tee -a /boot/firmware/usercfg.txt
gpu_mem=16
EOF
```

enable `cgroup_memory` for kubernetes.
```
sudo sh -c 'echo $(cat /boot/firmware/cmdline.txt) "cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1" > /boot/firmware/cmdline.txt'
```

Prepare OS.
```
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
```

```
sudo modprobe overlay && \
sudo modprobe br_netfilter
```

```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1
EOF
```
and apply it.
```
sudo sysctl --system
```

change the hostname before we begin with kubernetes.
you need to edit the hosts file with
```
sudo nano /etc/hosts
```
and in the hostname file.
```
sudo nano /etc/hostname
```

so you finished now then reboot.
```
sudo reboot
```

--- Install Kubernetes Dualstack ---

Install containerd:

```
sudo apt-get update && sudo apt-get install -y containerd
```

Configure containerd
```
sudo mkdir -p /etc/containerd && \
sudo containerd config default | sudo tee /etc/containerd/config.toml
```

Restart containerd:
```
sudo systemctl enable containerd && \
sudo systemctl restart containerd
```

Disable SWAP:
```
sudo swapoff -a
sudo dphys-swapfile swapoff && \
sudo dphys-swapfile uninstall && \
sudo systemctl disable dphys-swapfile
```

First Step Install packages.cloud.google kubernetes.list:
```
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add - && \
sudo sh -c 'echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list'
```

You will install these packages on all of your machines:
* kubeadm: the command to bootstrap the cluster.
* kubelet: the component that runs on all of the machines in your cluster and does things like starting pods and containers.
* kubectl: the command line util to talk to your cluster.

```
sudo apt-get update && sudo apt-get install -y kubelet kubeadm kubectl
```

serviceSubnet can be only /108 or greater becuase https://github.com/kubernetes/kubernetes/pull/90115
Soo you can use:
```
cat >> kubeadm.yaml << EOF
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
---                                                          
apiVersion: kubeadm.k8s.io/v1beta2
kind: InitConfiguration
nodeRegistration:
  kubeletExtraArgs:
    node-ip: "192.168.178.103,fd11:1111:1111:1111:dea6:32ff:fecc:d391"
---
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
featureGates:
  IPv6DualStack: true
networking:
  podSubnet: 200.200.0.0/16,fd7a:cccc:dddd::/48
  serviceSubnet: 10.200.0.0/16,fd00:0002:0:0:1::/108
EOF
```
init master. up to this point everything is the same for master and cluster. The cluster is ready to join master at this point.
```
sudo kubeadm init --config kubeadm.yaml
```

Save you join command from output!!!

Configure Kubectl:
```
mkdir -p $HOME/.kube && \
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config && \
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

So you need to deploy a Network Pod.
```
wget https://docs.projectcalico.org/v3.17/manifests/calico.yaml && \
kubectl apply -f calico.yaml
```

Be sure `kubectl get pods --all-namespaces` result:

```
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-number             1/1     Running   0          100s
kube-system   calico-node-number                         1/1     Running   0          95s
kube-system   coredns-number                             1/1     Running   0          100s
kube-system   coredns-number                             1/1     Running   0          100s
kube-system   etcd-hostname                              1/1     Running   0          120s
kube-system   kube-apiserver-hostname                    1/1     Running   0          120s
kube-system   kube-controller-manager-hostname           1/1     Running   1          120s
kube-system   kube-proxy-number                          1/1     Running   0          100s
kube-system   kube-scheduler-hostname                    1/1     Running   1          120s
```

Modifying Calico to support IPv6
```
wget https://raw.githubusercontent.com/Trackhe/Raspberry64bitKubernetesServerDualstack/master/deployment/calicov6.yaml && \
kubectl apply -f calicov6.yaml
```

Again, wait until all the Calico pods are running and have passed the readiness probes.

You need to untain your node to run pods on master Node:
```
kubectl taint nodes --all node-role.kubernetes.io/master-
```

After that be sure your Server reboot and start the Kubelet service. you can test it by using `sudo service kubelet status`.
if the service after the reboot not running or running into error 255 then use once `sudo kubeadm init --skip-phases=preflight --config kubeadm.yaml` that should be fix it.
you can boot your workers also if you head already join they. if thes dosn come back to ready use `sudo kubeadm join --skip-phases=preflight ip:port --token token.name --discovery-token-ca-cert-hash sha256:token`

Now its time to join your worker to the master with your saved command. use `sudo kubeadm join ip:port --token token.name --discovery-token-ca-cert-hash sha256:token` if you forget the command use `sudo kubeadm token create --print-join-command` on master to get a new.

Install Dashboard:

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.1.0/aio/deploy/recommended.yaml
```
and Metrics Server
```
kubectl apply -f https://raw.githubusercontent.com/Trackhe/Raspberry64bitKubernetesServerDualstack/master/deployment/components.yaml
```

Download dashboard user, ClusterRoleBinding, deploy and get the login token.


!!! Not working for 1.19
```
wget https://raw.githubusercontent.com/Trackhe/Raspberry64bitKubernetesServerDualstack/master/deployment/admin-user.yaml && \
kubectl apply -f admin-user.yaml && \
wget https://raw.githubusercontent.com/Trackhe/Raspberry64bitKubernetesServerDualstack/master/deployment/admin-cluster-role-binding.yaml && \
kubectl apply -f admin-cluster-role-binding.yaml && \
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep dashboard-admin-sa | awk '{print $1}')
```

!!! For > 1.20
```
kubectl create serviceaccount dashboard-admin-sa && \
kubectl create clusterrolebinding dashboard-admin-sa --clusterrole=cluster-admin --serviceaccount=default:dashboard-admin-sa && \
kubectl get secrets
```
Search ashboard-admin-sa-token-***** and get the token with:
```
kubectl describe secret dashboard-admin-sa-token-*****
```

Now you can connect to the server with a terminal via `ssh -L 8001:localhost:8001 pi@piipaddresse` -> `su` -> `raspberry` or the password -> run `kubectl proxy --address='0.0.0.0' --accept-hosts='^*$'`
and reach the Dashboard via `http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/overview?namespace=_all`

For Mac terminal: `ssh -L 8001:localhost:8001 pi@piipaddresse`
Linux Like Ubuntu : Unknown pls use google
Windows PShell : Unknown pls use google


Metallb install:
```
kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e "s/strictARP: false/strictARP: true/" | \
sed -e "s/mode: ""/mode: "ipvs"/" | \
kubectl apply -f - -n kube-system
```

```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.5/manifests/namespace.yaml && \
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.5/manifests/metallb.yaml && \
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```

create in kubernetes dashboard a config map !!edit the ip adresses. to your lan reachable adresses.
```
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.178.200-192.168.178.254
      - fd11:1111:1111:1111::/108
```
and change the configmap coredns.   If you dont see the coredns config map. Select all Namespaces and under Configuration and Storage you find Config maps
```
.:53 {
    errors
    health {
       lameduck 5s
    }
    ready
    template ANY ANY fritz.box {
      rcode NXDOMAIN
    }
    kubernetes cluster.local in-addr.arpa ip6.arpa {
       pods insecure
       fallthrough in-addr.arpa ip6.arpa
       ttl 30
    }
    prometheus :9153
    forward . /etc/resolv.conf {
       max_concurrent 1000
    }
    cache 30
    loop
    reload
    loadbalance
}
```

Install Network Node Manager to use
```
kubectl apply -f https://raw.githubusercontent.com/kakao/network-node-manager/master/deploy/network-node-manager_ipvs.yml
```

Install Ip Masq Agent
```
kubectl apply -f https://raw.githubusercontent.com/Trackhe/Raspberry64bitKubernetesServerDualstack/master/deployment/ip-masq-agent.yaml
```

//Optional | Next Stuff is Optional


Install calicoctl
```
curl -o calicoctl -L  https://github.com/projectcalico/calicoctl/releases/download/v3.16.6/calicoctl-linux-arm64 && \
chmod +x calicoctl && \
sudo mv calicoctl /usr/local/bin/
```


Install kubectl autocompletion
```
echo 'source <(kubectl completion bash)' >>~/.bashrc && \
bash --login
```


Ceph install
```

```

//Optional (Not Updated)
Longhorn install:
```
sudo snap install helm --classic && \
git clone https://github.com/longhorn/longhorn.git && \
cd longhorn/chart/ && \
rm values.yaml && \
wget https://raw.githubusercontent.com/Trackhe/Raspberry64bitKubernetesServerDualstack/master/deployment/values.yaml && \
kubectl create namespace longhorn-system && \
helm install longhorn . --namespace longhorn-system
```

You can also install weave-scope dashboard. This is not so Important.
```
wget https://raw.githubusercontent.com/Trackhe/Raspberry64bitKubernetesServerDualstack/master/deployment/scope.yaml && \
kubectl apply -f components.yaml
```

A Part to make the Dashboard on the LAN Reachable follows soon.

... Coming soon...

I hope you enjoy. Best Regards.

Feel free to make improvements. and share it with us.


kind: Service
apiVersion: v1
metadata:
  name: testd
  namespace: default
  labels:
    k8s-app: testnginx
spec:
  ipFamilyPolicy: SingleStack
  ports:
    - name: test1s
      protocol: TCP
      port: 80
      targetPort: 80
  selector:
    k8s-app: testnginx
  type: LoadBalancer
  loadBalancerIP: fd11:1111:1111:1111::2
  ipFamilies:
  - IPv6
