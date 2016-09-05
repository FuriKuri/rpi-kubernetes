# Kubernetes on Raspberry PI

## Current todos / problems
- [ ] Start first services
- [ ] The cluster do not survive a reboot :(

## Preparations
### Install OS
I used [HypriotOS](http://blog.hypriot.com/) in version 1.0.1. For an easy installation you can use [flash](https://github.com/hypriot/flash). Just insert your SD card and run the following command from your command line.

```
$ flash --hostname moby-dock-1 https://github.com/hypriot/image-builder-rpi/releases/download/v1.0.1/hypriotos-rpi-v1.0.1.img.zip
```
Repeat this for your other RPIs. Do not forget to increase the hostname.

In addtion I configured my DHCP server, that the RPIs get the following IPs:

* 192.168.0.200
* 192.168.0.201
* 192.168.0.202
* 192.168.0.203

To connect to your RPIs use pirate as username and hypriot as password.

```
$ ssh pirate@192.168.0.200
```

## Create cluster
### Master
To create the kubernetes cluster, I used [kube-deploy](https://github.com/kubernetes/kube-deploy)

First, connect to your first RPI over ssh (use **priate** as username and **hypriot** as password)

```
$ ssh pirate@192.168.0.200
```

When you are connected, checkout kube-deploy deploy and start the ```master.sh```

```
$ git clone https://github.com/kubernetes/kube-deploy
$ cd kube-deploy/docker-multinode
$ sudo ./master.sh
```

After this, the script will start some docker container. When the script is finished, you will see something like this:

```
+++ [0905 21:22:43] Launching Kubernetes master components...
d06915c03700ef43c3629a00ca772d5e5278d4fc3c91a428135fc3912058304e
+++ [0905 21:22:44] Done. It may take about a minute before apiserver is up.
```

To interact with kubernetes, we need to install kubectl:
```
$ sudo su

# If you use another kubernetes version, use this URL template https://storage.googleapis.com/kubernetes-release/release/v[KUBECTL_VERSION]/bin/linux/arm/kubectl
$ curl -sSL https://storage.googleapis.com/kubernetes-release/release/v1.3.6/bin/linux/arm/kubectl > /usr/local/bin/kubectl
$ chmod +x /usr/local/bin/kubectl
$ exit
```

Now the main kubernetes container will start some pods. This can take a while (not only a minute). When this is finished, ```kubectl get nodes``` should return the node:

```
$ kubectl get nodes
NAME            STATUS    AGE
192.168.0.200   Ready     3m
```

The dashboard will be available over [http://192.168.0.200:8080/ui/](http://192.168.0.200:8080/ui/).

### Worker
Now it's time to start the worker nodes. For this we need to connect to our RPIs 2-4. (repeat the following steps for the node 192.168.0.201, 192.168.0.202, 192.168.0.203)

```
$ ssh pirate@192.168.0.201
```

On the worker nodes we need to checkout the same project and use the ```worker.sh``` script. Before we execute this script, we need to set the MASTER_IP.

```
$ git clone https://github.com/kubernetes/kube-deploy
$ cd kube-deploy/docker-multinode
$ export MASTER_IP=192.168.0.200
$ sudo -E ./worker.sh
```

After this, ```kubectl get nodes``` on our master node will show all three worker nodes
```
# On our master node 192.168.0.200
$ kubectl get nodes
NAME            STATUS    AGE
192.168.0.200   Ready     12m
192.168.0.201   Ready     8m
192.168.0.202   Ready     8m
192.168.0.203   Ready     8m
```

## Troubleshooting
At the moment the cluster will not work, if the master node will be rebooted. I found no clean way to get the master node working after a reboot. The only way, which is working for me, is to reinitiate the master node.
I also have to clean ```/var/lib/kubelet``` otherwise the dashboard pod will not start again. This reinitiate only has to be done on the master node.

```
$ sudo ./master.sh
+++ [0905 21:21:56] K8S_VERSION is set to: v1.3.6
+++ [0905 21:21:56] ETCD_VERSION is set to: 2.2.5
+++ [0905 21:21:56] FLANNEL_VERSION is set to: v0.6.1
+++ [0905 21:21:56] FLANNEL_IPMASQ is set to: true
+++ [0905 21:21:56] FLANNEL_NETWORK is set to: 10.1.0.0/16
+++ [0905 21:21:56] FLANNEL_BACKEND is set to: udp
+++ [0905 21:21:57] RESTART_POLICY is set to: unless-stopped
+++ [0905 21:21:57] MASTER_IP is set to: localhost
+++ [0905 21:21:57] ARCH is set to: arm
+++ [0905 21:21:57] IP_ADDRESS is set to: 192.168.0.200
+++ [0905 21:21:57] USE_CNI is set to: false
+++ [0905 21:21:57] USE_CONTAINERIZED is set to: false
+++ [0905 21:21:57] --------------------------------------------
+++ [0905 21:21:57] Killing docker bootstrap...
+++ [0905 21:22:23] Killing all kubernetes containers...
ade9924666eb
9c5bd3a1a2b5
d7adc8dd5e36
86a78fff7988
389af71d80ab
4a2ac8ea9c48
5ffc181fff07
5fc47e950199
7b2546053ec7
cecc6d390def
dd6b8b79f19d
Do you want to clean /var/lib/kubelet? [Y/n] Y
+++ [0905 21:22:28] Launching docker bootstrap...
+++ [0905 21:22:30] Launching etcd...
4357db694ce6aa7a18d0c5d6d8a95b15c5b61202460be15dbf3457fe0bed7d61
+++ [0905 21:22:36] Launching flannel...
{"action":"set","node":{"key":"/coreos.com/network/config","value":"{ \"Network\": \"10.1.0.0/16\", \"Backend\": {\"Type\": \"udp\"}}","modifiedIndex":4,"createdIndex":4}}
acf4b02c9385f208992fd6861a656470a74aecbee50f866b01a212cd466bf052
+++ [0905 21:22:39] FLANNEL_SUBNET is set to: 10.1.2.1/24
+++ [0905 21:22:39] FLANNEL_MTU is set to: 1472
+++ [0905 21:22:39] Restarting main docker daemon...
+++ [0905 21:22:43] Restarted docker with the new flannel settings
+++ [0905 21:22:43] Launching Kubernetes master components...
d06915c03700ef43c3629a00ca772d5e5278d4fc3c91a428135fc3912058304e
+++ [0905 21:22:44] Done. It may take about a minute before apiserver is up.
```
