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
sudo kubeadm init --apiserver-advertise-address=10.0.15.10 --pod-network-cidr=10.244.0.0/16 --apiserver-cert-extra-sans=localhost
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

## Course notes

`kubectl` - a CLI tool for managing kubernetes clusters.
In order to use `kubectl` locally, we need first to start the cluster, by typing:

~~~
minikube start
~~~

Then

~~~
kubectl apply -f ./deployment.yaml
kubectl expose deployment tomcat-deployment --type=NodePort
minikube service tomcat-deployment --url
curl <URL>
~~~

`kubectl get pods` - get all info about the containers running
`kubectl describe pod <pod_name>` - get detail info about the pod
`kubectl port-forward <pod_name> [local_port]:[remote_port]` - map a remote port of the remote pod onto the local port of the local machine, useful
`kubectl attach <pod_name> -c <container>`
`kubectl exec [-it] <pod_name> [-c CONTAINER] -- COMMAND [args...]` - exec a command inside the container, same as `docker exec`
    example:
    `kubectl exec -it tomcat-deployment-56ff5c79c5-z9nxw bash`
`kubectl run hazelcast --image=hazelcast --port=5701` - run an image on the cluster without writing any `deployment.yrml`
`kubectl scale --replicas=4 deployment/tomcat-deployment` - scale the deployment without stopping it
`kubectl get deployments` - list deployments
`kubectl expose deployment tomcat-deployment --type=NodePort` - exposes the deployment by exposing one or several node ports
`kubectl describe services` - get info about the services
`kubectl expose deployment tomcat-deployment --type=LoadBalancer --port=8080 --target-port=8080 --name tomcat-load-balancer` - exposes the deployment through the load balancer

`Kubernetes node` is a VM or a physical machine which runs pods.
`kubelet` manages pods within a node.
`kube-proxy` makes sure that the networking defined in the deployment works as required.
`kube-dns` manages DNS system internally and automatically

Kubernetes supports node replication and scaling on the same or multiple nodes.
`Stateful` application keep session, `stateless` - don't.

Pod can run one or many containers.
A service is something that exposes a container to the outer world. A service can be created as load balancer.

We can use labels to mark nodes, like this:
`kubectl label node minikube storageType=ssd`
And then we can use `nodeSelector` to deploy pods only on the matched nodes.

Healthcheck probes can be either "readiness" or "liveness".

`Namespaces` allow to separate the cluster onto smaller logical clusters.

`Volumes`

`kubectl get persistentvolume` - list volumes on a cluster

`kubectl delete deployment <deployment_name>` - to stop all pods in the deployment, but other things like `persistentvolumes` will go on.
In order to completely remove stuff we can run
`kubectl delete -f ./wordpress-deployment.yaml` - `kubectl` will look into the file and remove all the listed resources

`Secrets` can be either files in a `volume` (i.e. private keys) or environment variables.
Secrets provide separation of the data, but does not provide encryption (they are base64-encoded)

`kubectl create secret generic db-user-pass --from-file=./username.txt --from-file=./password.txt`
`kubectl create secret mysql-pass --from-literal=password=YOUR_PASSWORD`
`kubectl get secret`

Monitoring

Heapster - the container monitor / InfluxDB / Grafana

Namespaces - allow to create virtual clusters on the same physical cluster, they provide separation. By default, the 'default' namespace is used.
Probably it makes sense to use namespaces when you need to run several projects on one cluster. Different namespaces can have different resource restrictions.

`kubectl create namespace <namespace-name>`
`kubectl get namespace`

Example:
~~~
kubectl create namespace cpu-limited-tomcat; # create a namespace
kubectl create -f ./cpu-limits.yaml --namespace=cpu-limited-tomcat; # provide the quota for the namespace
kubectl create -f ./tomcat-deployment.yaml --namespace=cpu-limited-tomcat;
kubectl describe deployment tomcat-deployment —namespace=cpu-limited-tomcat;
~~~

Auditing - some kind of logs

Helm vs Terraform
Helm trunes yaml files into templates.
When use service providers: Terraform or Helm
When using baremetal: Helm

~~~
helm create; # build and name a new chart
helm fetch; # download and unpack a chart into a folder
helm rollback; # rollback a release to a previous version
helm status; # show the status of a release
helm upgrade; # upgrade a release
helm history;
~~~

Minikube notes:

`minikube service wordpress --url`
`minikube start`
`minikube stop`

Kubernetes can do horizontal autoscaling out of the box with the Horizontal Pod Autoscaler.

`kubectl autoscale deployment wordpress --cpu-percent=50 --min=1 --max=5`

## kubectl to remote cluster

Check if we can ping all nodes!

How to access the remote cluster locally:

~~~~
scp -r vagrant@10.0.15.10:/home/vagrant/.kube .; # this works because the network is not internal and we enabled password authentication
cp -r .kube $HOME/;
# scp -i /Users/sergei/proj/k8s/master/.vagrant/machines/default/virtualbox/private_key -r vagrant@10.0.15.10:/home/vagrant/.kube .
~~~~

https://stackoverflow.com/questions/46360361/invalid-x509-certificate-for-kubernetes-master

`sudo kubeadm reset;`
`vagrant ssh-config;`

virtualbox__intnet: true


* 10.0.15.10 - master
* 10.0.15.21 - node01
* 10.0.15.22 - node02


## More of notes

<div>Icons made by <a href="https://www.flaticon.com/authors/freepik" title="Freepik">Freepik</a> from <a href="https://www.flaticon.com/" title="Flaticon">www.flaticon.com</a></div>


kubectl expose deployment/kubernetes-bootcamp --type="NodePort" --port 8080



195.140.146.219  master
195.140.147.21  node01

kubeadm init --apiserver-advertise-address=195.140.146.219 --pod-network-cidr=10.244.0.0/16 --apiserver-cert-extra-sans=localhost,127.0.0.1;

kubeadm join 195.140.146.219:6443 --token llaobt.qj0y2bowml35h1hn     --discovery-token-ca-cert-hash sha256:149355d2c48ab58caa9e8dfcd9dfb34ad681fc5d91525acc04f4c77d3c7f7014
kubeadm join 192.168.0.3:6443 --token llaobt.qj0y2bowml35h1hn     --discovery-token-ca-cert-hash sha256:149355d2c48ab58caa9e8dfcd9dfb34ad681fc5d91525acc04f4c77d3c7f7014

  [WARNING Firewalld]: firewalld is active, please ensure ports [6443 10250] are open or your cluster may not function correctly
  [WARNING SystemVerification]: this Docker version is not on the list of validated versions: 19.03.5. Latest validated version: 18.09


      tee /tmp/metallb-conf.yml > /dev/null <<'EOF'
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
          - 195.140.147.21
EOF



adduser bot;
gpasswd -a bot wheel;
gpasswd -a bot docker;


ssh-keygen -t rsa -b 4096;


sudo iptables -A INPUT -p tcp --dport 6443 -s 195.140.147.21 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 10250 -s 195.140.147.21 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT

kubectl logs -n ingress-nginx --tail=5 nginx-ingress-controller-568867bf56-fkg54
kubectl logs -n metallb-system --tail=5 controller-65895b47d4-qlhd7
kubectl -n metallb-system describe pod controller-65895b47d4-qlhd7

kubectl -n k8s-proj-prod get ingress

wget https://195.140.147.21:10250/containerLogs/ingress-nginx/nginx-ingress-controller-568867bf56-fkg54/nginx-ingress-controller --no-check-certificate


docker exec -u root feef175d2a3895d6097cc2ee455355826c8f19a1471a7ef13ccf1452311b588c -- /bin/bash


https://github.com/danderson/metallb/issues/327

