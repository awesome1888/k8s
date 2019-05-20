# Kubernetes with Vagrant

## Provisioning draft

### common
~~~
printf "\n10.0.15.10 k8s-master\n10.0.15.10 master\n10.0.15.21 node01\n10.0.15.22 node02\n" | sudo tee -a /etc/hosts
sudo setenforce 0
sudo sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
sudo modprobe br_netfilter
echo "br_netfilter" | sudo tee -a /etc/modules-load.d/br_netfilter.conf
echo '1' | sudo tee -a /proc/sys/net/bridge/bridge-nf-call-iptables
printf "\nnet.bridge.bridge-nf-call-iptables = 1\n" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

sudo swapoff -a

# todo:
# comment out swap in /etc/fstab

# install docker
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install -y docker-ce wget

sudo mkdir /etc/docker
sudo tee /etc/docker/daemon.json > /dev/null <<'EOF'
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
sudo systemctl start docker
sudo systemctl enable docker
sudo docker info | grep -i cgroup

# install kubernetes
sudo tee /etc/yum.repos.d/kubernetes.repo > /dev/null <<'EOF'
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
        https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
sudo yum install -y kubelet kubeadm kubectl
sudo systemctl start kubelet
sudo systemctl enable kubelet
~~~

### master
~~~
sudo kubeadm init --apiserver-advertise-address=10.0.15.10 --pod-network-cidr=10.244.0.0/16
# copy the "kubeadm join" command
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
~~~

### for each node(x)
~~~
sudo kubeadm join 10.0.15.10:6443 --token 82gs8m.vakwqrnev5r4z5qi --discovery-token-ca-cert-hash sha256:d72fc8e7cfc7339d7f52939ab653c7d9b4037ff5c66dc0c8d39c5fc29a2dda52
~~~

### master again
~~~
kubectl get nodes
~~~

## Useful commands

~~~
systemctl daemon-reload; systemctl restart kubelet;
kubectl get nodes
kubectl get pods --all-namespaces
sudo systemctl list-unit-files | grep enabled
sudo systemctl | grep kubelet
sudo journalctl -u kubelet.service -n 100 --no-pager
ip a
kubectl get pod -o=custom-columns=NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName --all-namespaces
kubeadm token create
~~~

## Useful links

* https://www.howtoforge.com/tutorial/centos-kubernetes-docker-cluster/
* https://medium.com/@wso2tech/multi-node-kubernetes-cluster-with-vagrant-virtualbox-and-kubeadm-9d3eaac28b98
* https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/
* https://kubernetes.io/docs/setup/cri/#docker
* https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/#instructions
* https://rancher.com/blog/2019/2019-03-21-comparing-kubernetes-cni-providers-flannel-calico-canal-and-weave/
* https://elastisys.com/wp-content/uploads/2018/01/kubernetes-ha-setup.pdf
* https://kubernetes.io/docs/tasks/debug-application-cluster/monitor-node-health/
* https://www.katacoda.com/courses/kubernetes/playground
* https://labs.play-with-k8s.com/
* https://stackoverflow.com/questions/48983354/kubernetes-list-all-pods-and-its-nodes?rq=1
* https://kubernetes.io/docs/concepts/
* https://github.com/kubernetes/kubeadm/issues/659
