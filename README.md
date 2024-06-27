# Cluster Information

The network space used for the node cluster is `10.10.10.0/24`.

| Node | VM Name | hostname | IP |
| ---- | ---- | --- | --- |
| master | `kubemaster01` | `kubemaster01` | `192.168.56.21` |
| node 1 | `kubenode01` | `kubenode01` | `192.168.56.31` |
| node 2 | `kubenode02` | `kubenode01` | `192.168.56.32` |
| node 3 | `kubenode03` | `kubenode01` | `192.168.56.33` |


# Clone this repo

Clone this repo and open in a shell/gitbash.

# Software Installation

Install **Vagrant** and **VirtualBox**:

Dependencies (Linux):

```
sudo apt install mkisofs
```

* Install [Vagrant](https://developer.hashicorp.com/vagrant/docs/installation)
* Install [VirtualBox](https://www.virtualbox.org/) (For Linux and MacOS)

Verify Vagrant is installed:
```
$ vagrant version
Installed Version: 2.4.1
Latest Version: 2.4.1
```

If you receive the following error, you'll need to install `fuse2`.  For modern distros, `fuse3` will be installed.  Please read the [AppImage Fuse docs](https://docs.appimage.org/user-guide/troubleshooting/fuse.html#the-appimage-tells-me-it-needs-fuse-to-run) on how to properly install `fuse2` alongside `fuse3`.  

WARNING: Failing to install `fuse2` correctly (ie, installing `fuse`) could result in [this critical bug](https://bugs.launchpad.net/ubuntu/+source/gdm3/+bug/1717878) that disables your system!

```
$ vagrant version
dlopen(): error loading libfuse.so.2

AppImages require FUSE to run. 
You might still be able to extract the contents of this AppImage 
if you run it with the --appimage-extract option. 
See https://github.com/AppImage/AppImageKit/wiki/FUSE 
for more information
```

For [Ubuntu (>=22.04)](https://docs.appimage.org/user-guide/troubleshooting/fuse.html#setting-up-fuse-2-x-alongside-of-fuse-3-x-on-recent-ubuntu-22-04-debian-and-their-derivatives):

```
sudo apt install libfuse2
```


Test from this project's directory:
```
vagrant status
Current machine states:

kubemaster01              not created (virtualbox)
kubenode01                not created (virtualbox)
kubenode02                not created (virtualbox)
kubenode03                not created (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
```

# Create and Configure VMs

This cluster is configured to run with one master node and three workers.  If you'd like to change that, update the Vagrantfile and modify `NUM_MASTER_NODE` and/or `NUM_WORKER_NODE`.

## Create and start the VMs using vagrant
```
vagrant up
```

The output for should start with:

```
Bringing machine 'kubemaster01' up with 'virtualbox' provider...
Bringing machine 'kubenode01' up with 'virtualbox' provider...
Bringing machine 'kubenode02' up with 'virtualbox' provider...
Bringing machine 'kubenode03' up with 'virtualbox' provider...
```

Verify the VMs are running:
```
vagrant@kubemaster01:~$ vagrant status
Current machine states:

kubemaster01              running (virtualbox)
kubenode01                running (virtualbox)
kubenode02                running (virtualbox)
kubenode03                running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.

```

## Configurations for all VMs (worker and master nodes)

Additional setup is needed.  Open a shell/gitbash for each vm using vagrant.  

```
vagrant ssh <vm-name>
```

For eaxmple: to enter the `kubemaster01` VM:
```
vagrant ssh kubemaster01
```

Other VMs would be:
```
vagrant ssh kubenode01
```

Optionally set default editor to `nano` (recommended for non-vi users):

```
export EDITOR=nano
echo 'EDITOR=nano' >> ~/.bashrc
```

Optionally (recommended), update all packages on all VMs (just hit `<enter>` on any popups dialogs that appear).  Don't forget to reboot!:

```
sudo apt --option=Dpkg::Options::=--force-confold --option=Dpkg::options::=--force-unsafe-io --assume-yes --quiet upgrade
sudo reboot
```


## Configure master node(s) only (`kubemaster01` vm):

First, we need to initialize the master VM as the Kubernetes master node.

Run `kubeadm init`:

```
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.56.21
```

Output should look like:

```
vagrant@kubemaster01:~$ sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.56.21

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

kubeadm join 192.168.56.21:6443 --token t1wb6k.x8pdo6qxlibz3r2i \
	--discovery-token-ca-cert-hash sha256:14db5e4b61e127a5a6afb72a056bb351d2b5a27984aed1070ce78d207265a78f 
```

Notice the above output lists commands that you need to run.  Save the last command (`kubeadm join ...`) to run on worker nodes.

First, let's work on the `kubemaster01` vm only.

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Next, install Weave Net:

```
vagrant@kubemaster01:~$ kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
serviceaccount/weave-net created
clusterrole.rbac.authorization.k8s.io/weave-net created
clusterrolebinding.rbac.authorization.k8s.io/weave-net created
role.rbac.authorization.k8s.io/weave-net created
rolebinding.rbac.authorization.k8s.io/weave-net created
daemonset.apps/weave-net created
```

Verify Weave Net is installed:
```
vagrant@kubemaster01:~$ kubectl get pods -A
NAMESPACE     NAME                                 READY   STATUS              RESTARTS      AGE
kube-system   coredns-5dd5756b68-lfktv             0/1     ContainerCreating   0             13s
kube-system   coredns-5dd5756b68-vxzcn             0/1     ContainerCreating   0             13s
kube-system   etcd-kubemaster01                    1/1     Running             1 (76s ago)   43s
kube-system   kube-apiserver-kubemaster01          1/1     Running             1 (46s ago)   78s
kube-system   kube-controller-manager-kubemaster01 1/1     Running             1 (76s ago)   78s
kube-system   kube-proxy-s2vkv                     1/1     Running             1 (12s ago)   13s
kube-system   kube-scheduler-kubemaster01          1/1     Running             1 (76s ago)   43s
kube-system   weave-net-w622v                      2/2     Running             1 (3s ago)    13s
```

Modify Weave Net's configuration.  Run the following command, which will open a configuration in your preferred editor:

```
kubectl edit ds weave-net -n kube-system
```

Find the Weave Net container (`name: weave`) and add a new enviroment variable (just above, under the `env` section.  Use the correct spacing/alignment!).  Save and exit.

* `name`: `IPALLOC_RANGE`
* `value`: `10.244.0.0/16`

It should look like:
```
spec:
  minReadySeconds: 5
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      name: weave-net
  template:
    metadata:
      creationTimestamp: null
      labels:
        name: weave-net
    spec:
      containers:
      - command:
        - /home/weave/launch.sh
        env:
        - name: IPALLOC_RANGE
          value: 10.244.0.0/16
        - name: INIT_CONTAINER
          value: "true"
        - name: HOSTNAME
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: spec.nodeName
        image: weaveworks/weave-kube:latest
        imagePullPolicy: Always
        name: weave

```

When everything is good, you should see:

```
vagrant@kubemaster01:~$ kubectl edit ds weave-net -n kube-system
daemonset.apps/weave-net edited
```

Verify Weave Net is running:
```
kubectl get pods -A
```

## Configure worker nodes (`kubenode*`)

Lastly, join the worker nodes to the master.  Run the following on each `kubenode*` instance (note, this command comes from the output of the command we ran earlier, which we took note of):

```
kubeadm join 192.168.56.21:6443 --token t1wb6k.x8pdo6qxlibz3r2i \
	--discovery-token-ca-cert-hash sha256:14db5e4b61e127a5a6afb72a056bb351d2b5a27984aed1070ce78d207265a78f 
```

# Suspend, Resume and Destroy VMs

When not using the VMs for your cluster, you can suspend and then resume them:

```
vagrant suspend
vagrant resume
```

You can completely remove the created VMs:

```
vagrant destroy
    kubenode03: Are you sure you want to destroy the 'kubenode03' VM? [y/N] y
==> kubenode03: Forcing shutdown of VM...
==> kubenode03: Destroying VM and associated drives...
    kubenode02: Are you sure you want to destroy the 'kubenode02' VM? [y/N] y
==> kubenode02: Forcing shutdown of VM...
==> kubenode02: Destroying VM and associated drives...
    kubenode01: Are you sure you want to destroy the 'kubenode01' VM? [y/N] y
==> kubenode01: Forcing shutdown of VM...
==> kubenode01: Destroying VM and associated drives...
    kubemaster01: Are you sure you want to destroy the 'kubemaster01' VM? [y/N] y
==> kubemaster01: Forcing shutdown of VM...
==> kubemaster01: Destroying VM and associated drives...

```

# Troubleshooting

See [Troubleshooting kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/troubleshooting-kubeadm/) for additional information.

Verify the following are configured:

```
vagrant@kubemaster01:~$ lsmod | grep br_netfilter
br_netfilter           32768  0
bridge                409600  1 br_netfilter
vagrant@kubemaster01:~$ lsmod | grep overlay
overlay               196608  10
```

```
vagrant@kubemaster01:~$ sudo sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
```

```
vagrant@kubemaster01:~$ sudo more /etc/modules-load.d/k8s.conf
overlay
br_netfilter
```

```
vagrant@kubemaster01:~$ sudo more /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
```

```
vagrant@kubemaster01:~$ sudo more /etc/containerd/config.toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
```


```
vagrant@kubemaster01:~$ sudo more /etc/hosts
127.0.0.1 localhost

# The following lines are desirable for IPv6 capable hosts
::1 ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts
192.168.56.21  kubemaster01
192.168.56.31  kubenode01
192.168.56.32  kubenode02
192.168.56.33  kubenode03
127.0.1.1 kubemaster01 kubemaster01
```
