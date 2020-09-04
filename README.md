#Installing K8s in Fedora

This is a step by step guide for a local (lab) vanilla K8s on Fedora Server.

Tested with a fresh install of Fedora 32 Server.
In my case, it's just Fedora 32 Server in an old laptop. 

Based on work by Robert Calva.


### Resolving our FQDN Locally

```
[root@fedora ~]# echo "192.168.0.142 `hostname`" >> /etc/hosts 
[root@fedora ~]# cat /etc/hosts 
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.0.142 fedora-laptop
[root@fedora ~]# 

```


### SELinux as permissive

```
[root@fedora-laptop ~]# setenforce Permissive
[root@fedora-laptop ~]# getenforce
Permissive
```


### Disabling swap

```
[root@fedora-laptop ~]# sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
[root@fedora-laptop ~]# swapoff -a
```


### Configure Firewall

```
[root@fedora-laptop ~]# firewall-cmd --zone=public --add-port=10250/tcp
success
[root@fedora-laptop ~]# firewall-cmd --zone=public --add-port=6443/tcp
success
```


### Docker & Fedora 32 has an issue with cgroups v2

```
[root@fedora ~]# dnf -y install grubby
[root@fedora ~]# grubby --update-kernel=ALL --args="systemd.unified_cgroup_hierarchy=0"
[root@fedora ~]# dnf -y install moby-engine

```

### Install Traffic Control (iproute-tc)

```
[root@fedora-laptop ~]# dnf install -y tc
```

### Install utilities
```
[root@fedora-laptop ~]# sudo dnf install -y jq
```


### Setup Docker Service

```
[root@fedora-laptop ~]# systemctl enable --now docker
[root@fedora-laptop ~]# sudo usermod -aG docker $(whoami)
[root@fedora-laptop ~]# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
[root@fedora-laptop ~]#
```


### Install Packages
- kubernetes
- kubernetes-client
- kubernetes-node 
- kubernetes-master 
- kubernetes-kubeadm
- openvswitch-ovn-kubernetes.x86_64 
- flannel
- podman.x86_64

```
[root@fedora-laptop ~]# dnf install -y kubernetes kubernetes-kubeadm openvswitch-ovn-kubernetes.x86_64 flannel  podman.x86_64
```


### Modify Kubelet Service (Fedora 31 & Kubernetes 1.15 issue)

Edit  `/etc/systemd/system/kubelet.service.d/kubeadm.conf` and delete `--allow-privileged=true` and `--fail-swap-on=true` flags.

```
[root@fedora-laptop ~]# cat /etc/systemd/system/kubelet.service.d/kubeadm.conf
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_SYSTEM_PODS_ARGS=--pod-manifest-path=/etc/kubernetes/manifests"
Environment="KUBELET_NETWORK_ARGS=--network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/usr/libexec/cni"
Environment="KUBELET_DNS_ARGS=--cluster-dns=10.96.0.10 --cluster-domain=cluster.local"
Environment="KUBELET_AUTHZ_ARGS=--authorization-mode=Webhook --client-ca-file=/etc/kubernetes/pki/ca.crt"
Environment="KUBELET_EXTRA_ARGS=--cgroup-driver=systemd"
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_SYSTEM_PODS_ARGS $KUBELET_NETWORK_ARGS $KUBELET_DNS_ARGS $KUBELET_AUTHZ_ARGS $KUBELET_EXTRA_ARGS
Restart=always
StartLimitInterval=0
RestartSec=10
```


### Start Kubelet Service

```
[root@fedora-laptop ~]# systemctl enable --now kubelet
```


### Install!

If steps fail from here, run `kubeadm reset` and start over from this step.

```
[root@fedora-laptop ~]# kubeadm init --kubernetes-version=stable 
I0903 22:25:00.565644  128039 version.go:248] remote version is much newer: v1.19.0; falling back to: stable-1.15
[init] Using Kubernetes version: v1.15.12
[preflight] Running pre-flight checks
	[WARNING Firewalld]: firewalld is active, please ensure ports [6443 10250] are open or your cluster may not function correctly
	[WARNING SystemVerification]: this Docker version is not on the list of validated versions: 19.03.11. Latest validated version: 18.09
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Activating the kubelet service
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [fedora-laptop localhost] and IPs [192.168.0.142 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [fedora-laptop localhost] and IPs [192.168.0.142 127.0.0.1 ::1]
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [fedora-laptop kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.0.142]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[kubelet-check] Initial timeout of 40s passed.
[apiclient] All control plane components are healthy after 41.512418 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.15" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node fedora-laptop as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node fedora-laptop as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: l5b5wn.1d8mpsa38loe5945
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.0.142:6443 --token l5b5wn.1d8mpsa38loe5945 \
    --discovery-token-ca-cert-hash sha256:1c1cc19683a19b06a25cee1a1aebffa10fd95fba6a8e60bb07e848cc1aff36be 
[root@fedora-laptop ~]# 
```


### Configuring our Kubernetes environment

```
[pseguel@fedora-laptop ~]$ mkdir -p $HOME/.kube
[pseguel@fedora-laptop ~]$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
[pseguel@fedora-laptop ~]$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```


### Check kube-system status

```
[pseguel@fedora-laptop ~]$  kubectl get all --all-namespaces
NAMESPACE     NAME                                        READY   STATUS    RESTARTS   AGE
kube-system   pod/coredns-5d4dd4b4db-8tbrn                0/1     Pending   0          3m39s
kube-system   pod/coredns-5d4dd4b4db-pn4b7                0/1     Pending   0          3m40s
kube-system   pod/etcd-fedora-laptop                      1/1     Running   0          3m9s
kube-system   pod/kube-apiserver-fedora-laptop            1/1     Running   0          2m54s
kube-system   pod/kube-controller-manager-fedora-laptop   1/1     Running   0          2m50s
kube-system   pod/kube-proxy-fttqx                        1/1     Running   0          3m40s
kube-system   pod/kube-scheduler-fedora-laptop            1/1     Running   0          3m8s


NAMESPACE     NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
default       service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP                  3m49s
kube-system   service/kube-dns     ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   3m45s

NAMESPACE     NAME                        DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                 AGE
kube-system   daemonset.apps/kube-proxy   1         1         1       1            1           beta.kubernetes.io/os=linux   3m45s

NAMESPACE     NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
kube-system   deployment.apps/coredns   0/2     2            0           3m45s

NAMESPACE     NAME                                 DESIRED   CURRENT   READY   AGE
kube-system   replicaset.apps/coredns-5d4dd4b4db   2         2         0       3m40s





```


### Install Flannel (Config L3 network fabric)

```
[pseguel@fedora-laptop ~]$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
podsecuritypolicy.policy/psp.flannel.unprivileged created
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.apps/kube-flannel-ds-amd64 created
daemonset.apps/kube-flannel-ds-arm64 created
daemonset.apps/kube-flannel-ds-arm created
daemonset.apps/kube-flannel-ds-ppc64le created
daemonset.apps/kube-flannel-ds-s390x created
[pseguel@fedora-laptop ~]$ 
``` 


### Check Flannel status

Repeat until everything is running (pod/coredns-* are ContainerCreating).

```
[pseguel@fedora-laptop ~]$ kubectl get all --all-namespaces
NAMESPACE     NAME                                        READY   STATUS              RESTARTS   AGE
kube-system   pod/coredns-5d4dd4b4db-8tbrn                0/1     ContainerCreating   0          6m32s
kube-system   pod/coredns-5d4dd4b4db-pn4b7                0/1     ContainerCreating   0          6m33s
kube-system   pod/etcd-fedora-laptop                      1/1     Running             0          6m2s
kube-system   pod/kube-apiserver-fedora-laptop            1/1     Running             0          5m47s
kube-system   pod/kube-controller-manager-fedora-laptop   1/1     Running             0          5m43s
kube-system   pod/kube-flannel-ds-amd64-n5x6h             0/1     Error               4          2m25s
kube-system   pod/kube-proxy-fttqx                        1/1     Running             0          6m33s
kube-system   pod/kube-scheduler-fedora-laptop            1/1     Running             0          6m1s


NAMESPACE     NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
default       service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP                  6m42s
kube-system   service/kube-dns     ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   6m38s

NAMESPACE     NAME                                     DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                 AGE
kube-system   daemonset.apps/kube-flannel-ds-amd64     1         1         0       1            0           <none>                        2m25s
kube-system   daemonset.apps/kube-flannel-ds-arm       0         0         0       0            0           <none>                        2m25s
kube-system   daemonset.apps/kube-flannel-ds-arm64     0         0         0       0            0           <none>                        2m25s
kube-system   daemonset.apps/kube-flannel-ds-ppc64le   0         0         0       0            0           <none>                        2m25s
kube-system   daemonset.apps/kube-flannel-ds-s390x     0         0         0       0            0           <none>                        2m25s
kube-system   daemonset.apps/kube-proxy                1         1         1       1            1           beta.kubernetes.io/os=linux   6m38s

NAMESPACE     NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
kube-system   deployment.apps/coredns   0/2     2            0           6m38s

NAMESPACE     NAME                                 DESIRED   CURRENT   READY   AGE
kube-system   replicaset.apps/coredns-5d4dd4b4db   2         2         0       6m33s




```


### Check nodes status

Node should be `Ready`.

```
[pseguel@fedora-laptop ~]$ kubectl get nodes
NAME            STATUS   ROLES    AGE    VERSION
fedora-laptop   Ready    master   8m1s   v1.15.8-beta.0
```


### Enable our Master node as Worker node

```
[pseguel@fedora-laptop ~]$ kubectl get nodes -o json | jq .items[].spec.taints
[
  {
    "effect": "NoSchedule",
    "key": "node-role.kubernetes.io/master"
  }
]
[pseguel@fedora-laptop ~]$ kubectl taint nodes --all node-role.kubernetes.io/master-
node/fedora-laptop untainted
[pseguel@fedora-laptop ~]$ kubectl get nodes -o json | jq .items[].spec.taints
null
```

We expect a null after last command when Master is enabled as Worker.



### Firewall Rules on Master

```
[pseguel@fedora-laptop ~]$ sudo -i 
[root@fedora-laptop ~]# firewall-cmd --permanent --add-port=6443/tcp
success
[root@fedora-laptop ~]# firewall-cmd --permanent --add-port=2379-2380/tcp
success
[root@fedora-laptop ~]# firewall-cmd --permanent --add-port=10250/tcp
success
[root@fedora-laptop ~]# firewall-cmd --permanent --add-port=10251/tcp
success
[root@fedora-laptop ~]# firewall-cmd --permanent --add-port=10252/tcp
success
[root@fedora-laptop ~]# firewall-cmd --permanent --add-port=10255/tcp
success
[root@fedora-laptop ~]# firewall-cmd --permanent --add-port=8472/udp
success
[root@fedora-laptop ~]# firewall-cmd --add-masquerade --permanen
success
[root@fedora-laptop ~]# 
```

Apply this rule if you want NodePorts exposed on control plane IP as well

```
[root@fedora-laptop ~]# firewall-cmd --permanent --add-port=30000-32767/tcp
success
```


###  Firewall Rules on Nodes

```
firewall-cmd --permanent --add-port=10250/tcp
firewall-cmd --permanent --add-port=10255/tcp
firewall-cmd --permanent --add-port=8472/udp
firewall-cmd --permanent --add-port=30000-32767/tcp
firewall-cmd --add-masquerade --permanent
```

## Creating our first pod

```
[pseguel@fedora-laptop ~]$ cat pod.json 
{
    "apiVersion": "v1",
    "kind": "Pod",
    "metadata": {
        "labels": {
            "app": "apache-centos7"
        },
        "name": "apache-centos7"
    },
    "spec": {
        "containers": [
            {
                "image": "centos/httpd",
                "name": "apache-centos7",
                "ports": [
                    {
                        "containerPort": 80,
                        "hostPort": 80,
                        "protocol": "TCP"
                    }
                ]
            }
        ]
    }
}

[pseguel@fedora-laptop ~]$ kubectl create -f ./pod.json 
pod/apache-centos7 created

```

### Set SELinux contexts

#### for kubernetes files
chcon -R -t svirt_sandbox_file_t /etc/kubernetes/

#### for etcd files
chcon -R -t svirt_sandbox_file_t /var/lib/etcd

