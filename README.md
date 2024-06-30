# Cluster Information

The network subnet used for this node cluster is `10.10.10.0/24`.

| Node | VM Name | hostname | IP |
| ---- | ---- | --- | --- |
| master | `kubemaster01` | `kubemaster01` | `192.168.56.21` |
| node 1 | `kubenode01` | `kubenode01` | `192.168.56.31` |
| node 2 | `kubenode02` | `kubenode01` | `192.168.56.32` |
| node 3 | `kubenode03` | `kubenode01` | `192.168.56.33` |

# Prerequisites

* Install [Vagrant](https://developer.hashicorp.com/vagrant/docs/installation)
* Install [VirtualBox](https://www.virtualbox.org/) (For Linux and MacOS)

It's higly recommended you install [brew.sh](https://brew.sh), which we'll be using to install other software.

Install VirtualBox via you're system package manager.

Vagrant can be install using `brew`:

```
brew install vagrant
```

## Software Installation

Dependencies (Linux):

```
sudo apt install mkisofs
```

Verify Vagrant is installed:
```
$ vagrant version
Installed Version: 2.4.1
Latest Version: 2.4.1
```

If you receive the following error, you'll need to install `fuse2`.  For modern distros, `fuse3` will be installed.  Please read the [AppImage Fuse docs](https://docs.appimage.org/user-guide/troubleshooting/fuse.html#the-appimage-tells-me-it-needs-fuse-to-run) on how to properly install `fuse2` alongside `fuse3`.  

**WARNING: Failing to install `fuse2` correctly (ie, installing `fuse`) could result in [this critical bug](https://bugs.launchpad.net/ubuntu/+source/gdm3/+bug/1717878) that disables your system!**

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

Install `scp` plugin:

```
vagrant plugin install vagrant-scp
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

## NAT vs BRIDGE

`Vagrantfile` and `cloud-init.yaml` are both configured to configure the VMs with NAT routing, which is perfectly fine if running your cluster from a laptop or workstation.  If you want your cluster network available, you'll should to use bridge networking, which is done in `Vagrantfile-bridge`.

The basic differences are as followed.

### NAT (local access)

In `Vagrantfile`, VM networking uses predefined IP addresses, with network subnet defined by `IP_NW`:

```
IP_NW = "192.168.56."
```

Master nodes are in the range of `192.168.56.[21-29]`:

```
MASTER_IP_START = 20
node.vm.network :private_network, ip: IP_NW + "#{MASTER_IP_START + i}"
```

And worker nodes are in the range of `192.168.56.[31-39]`:

```
NODE_IP_START = 30
node.vm.network :private_network, ip: IP_NW + "#{NODE_IP_START + i}"
```

The number of master and worker nodes are defined by the following:

```
NUM_MASTER_NODE = 1
NUM_WORKER_NODE = 3
```

In addition, `cloud-init.yaml` predefines the list of hostnames/IPs needed to add to each VM's `/etc/hosts` file.  So, if you change any of the configurations above, you need to adjust that file accordingly:

```
  - content: |
      192.168.56.21  kubemaster01
      192.168.56.31  kubenode01
      192.168.56.32  kubenode02
      192.168.56.33  kubenode03
    path: /etc/hosts
    append: true
```

### Bridged (network access)

Although setting up a bridged network interface could use a manually configured IP, etc, it's easier to use DHCP with staticly assigned  IPs based on pre-defined MAC addresses.  You'll benefit from all that DHCP offers, obtain predicatable address, and may not have to make any changes to `cloud-init.yaml`. 

Therefore, this bridged example we make the assumption that you've configured your DCHP to assigned predictable IPs to the following MAC addresses.

Here is a sample of MAC addresses assigned to the VM devices:

| Node | VM Name | hostname | MAC |
| ---- | ---- | --- | --- |
| master | `kubemaster01` | `kubemaster01` | `08AB00000021` |
| node 1 | `kubenode01` | `kubenode01` | `08AB00000031` |
| node 2 | `kubenode02` | `kubenode01` | `08AB00000032` |
| node 3 | `kubenode03` | `kubenode01` | `08AB00000033` |

You can change the MAC prefix in `Vagrantfile-bridge`:

```
MAC = "08AB000000"
```

Master node MACs are in the range of `08AB000000[21-29]`:

```
MASTER_MAC_START = 20
node.vm.network "public_network", bridge: "#{BRIDGE_INTERFACE}", :mac=> "#{MAC}#{MASTER_MAC_START + i}"
```

And worker nodes MACs are in the range of `08AB000000[31-39]`:

```
NODE_MAC_START = 30
node.vm.network "public_network", bridge: "#{BRIDGE_INTERFACE}", :mac=> "#{MAC}#{NODE_MAC_START + i}"
```

Lastly, update `cloud-init.yaml` to use the static addresses you configured in DHCP server.


## Create and start the VMs using vagrant

Not, let's bring up our VMs:

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

Run `kubeadm init` using the IP address of the masternode:

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

Next, some file setup on `kubemaster01` (all masters):

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Exit the vm and copy the kubernetes config to your host and workstation.

**WARNING Be sure to not step on your existing kubeconfig!**

Verify you don't already have a `~/.kube/config` file:
```
ls ~/.kube/
```

If you do, modify the following command to save the copied config to another location.  Otherwise, create the `~/.kube` directory and copy your cluster's config.

```
mkdir ~/.kube
vagrant scp kubemaster01:~/.kube/config ~/.kube/config
```

# Configure worker nodes (`kubenode*`)

Lastly, join the worker nodes to the master.  Run the following on each `kubenode*` instance (note, this command comes from the output of the command we ran earlier, which we took note of):

```
sudo kubeadm join 192.168.56.21:6443 --token t1wb6k.x8pdo6qxlibz3r2i --discovery-token-ca-cert-hash sha256:14db5e4b61e127a5a6afb72a056bb351d2b5a27984aed1070ce78d207265a78f 
```

# Install CNI (and Service Mesh)

## Cilium (recommended)

[Cilium](https://cilium.io/) provides CNI and Service Mesh functionality, and is highly recommended.  Learn more about Cilium [here](https://docs.cilium.io/en/stable/).

### Cilium pre-setup

I don't beleive Helm is required for Cilium installation, but if it is you can install via brew.

```
brew install helm
```

If you're installing Cilium (reommended) instead of Weave Net, you'll need to install it's cli.

First install golang:
```
sudo apt install golang-go
```

### Cilium cli

cilium cli is the utility used to configure Cilium in your cluster.

Install using either brew (older version) or direct binary (latest version): 

```
brew install cilium-cli
```

Latest binary (instructions available from their GitHub [repo](https://github.com/cilium/cilium-cli)):

```
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
GOOS=$(go env GOOS)
GOARCH=$(go env GOARCH)
curl -L --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-${GOOS}-${GOARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-${GOOS}-${GOARCH}.tar.gz.sha256sum
sudo tar -C /usr/local/bin -xzvf cilium-${GOOS}-${GOARCH}.tar.gz
rm cilium-${GOOS}-${GOARCH}.tar.gz{,.sha256sum}
```

Verify installation:

```
cilium version --client
```

### Hubble

Hubble is the observability layer of Cilium and can be used to obtain cluster-wide visibility into the network and security layer of your Kubernetes cluster.

Next, install [Hubble](https://docs.cilium.io/en/stable/gettingstarted/hubble_setup/), which is part of the Cilium ecosystem:

Enable Hubble:

```
cilium hubble enable
cilium hubble enable -ui
```

To run the cilium connectivity test, you'll need to enable hubble port-forwarding.  

```
$ cilium hubble port-forward&
[1] 250053
```

Now, install cilium:

```
cilium install
```

After a  few minutes you can check the status:

```
$ cilium status
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    disabled (using embedded mode)
 \__/¯¯\__/    Hubble Relay:       OK
    \__/       ClusterMesh:        disabled

Deployment             cilium-operator    Desired: 1, Ready: 1/1, Available: 1/1
Deployment             hubble-relay       Desired: 1, Ready: 1/1, Available: 1/1
Deployment             hubble-ui          Desired: 1, Ready: 1/1, Available: 1/1
DaemonSet              cilium             Desired: 4, Ready: 4/4, Available: 4/4
Containers:            hubble-relay       Running: 1
                       hubble-ui          Running: 1
                       cilium             Running: 4
                       cilium-operator    Running: 1
Cluster Pods:          9/9 managed by Cilium
Helm chart version:    
Image versions         cilium             quay.io/cilium/cilium:v1.15.6@sha256:6aa840986a3a9722cd967ef63248d675a87add7e1704740902d5d3162f0c0def: 4
                       cilium-operator    quay.io/cilium/operator-generic:v1.15.6@sha256:5789f0935eef96ad571e4f5565a8800d3a8fbb05265cf6909300cd82fd513c3d: 1
                       hubble-relay       quay.io/cilium/hubble-relay:v1.15.6@sha256:a0863dd70d081b273b87b9b7ce7e2d3f99171c2f5e202cd57bc6691e51283e0c: 1
                       hubble-ui          quay.io/cilium/hubble-ui:v0.13.0@sha256:7d663dc16538dd6e29061abd1047013a645e6e69c115e008bee9ea9fef9a6666: 1
                       hubble-ui          quay.io/cilium/hubble-ui-backend:v0.13.0@sha256:1e7657d997c5a48253bb8dc91ecee75b63018d16ff5e5797e5af367336bc8803: 1
```

With everything looking good, run your test:

```
cilium connectivity test
```

Unless you installed extension like ingress, many tests will be skipped.  If you are following this guide with not extra extensions to your cluster, you should see the test complete with the last line similiar to:

```
cilium connectivity test
```

## Weave Net (Alternative CNI)

Weave Net is good if you just want networking in your cluster, without any of the extras like service meshes, etc.

See the [new Weave Net](https://rajch.github.io/weave/) for more information.

From within `kubemaster01`:

```
vagrant@kubemaster01:~$ kubectl apply -f https://reweave.azurewebsites.net/k8s/v1.30/net.yaml
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
