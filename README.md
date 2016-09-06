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
### Generell setup for all nodes (master and worker)
To create the kubernetes cluster, I use [kubernetes-on-arm](https://github.com/luxas/kubernetes-on-arm). In the first setup I used [kube-deploy](https://github.com/kubernetes/kube-deploy), but kubernetes-on-arm is more comfortable. It installs kbuectl and offers some kubernetes addons. Btw, kubernetes-on-arm uses kube-deploy to create the cluster.

First, connect to your first RPI over ssh (use **priate** as username and **hypriot** as password)

```
$ ssh pirate@192.168.0.200
```

When you are connected, download ```docker-multinode.deb``` in version 0.8.0.

```
$ wget https://github.com/luxas/kubernetes-on-arm/releases/download/v0.8.0/docker-multinode.deb
$ sudo dpkg -i docker-multinode.deb
$ sudo kube-config install
```

```kube-config install``` will ask you some questions. I used the following setup (replace ```moby-dock-1``` to the corresponding host):

```
Which board is this running on? Options: [bananapro, cubietruck, generic, odroid-c2, parallella, pine64, rpi-2, rpi-3, rpi]. rpi-3
Which OS do you have? Options: [archlinux, debian, hypriotos]. hypriotos
What hostname do you want? Defaults to kubepi. moby-dock-1
Which timezone should be set? Defaults to Europe/Helsinki. Europe/Berlin
Which storage driver do you want? Defaults to overlay.
Do you want to create an 1GB swapfile (required for compiling)? N is default [y/N] n
Do you want to reboot now? A reboot is required for Docker to function. Y is default. [Y/n] n
```

### Master

Now install the kubernetes master node.

```
sudo kube-config enable-master
```

After this, the script will start some docker container. When the script is finished, you will see something like this:

```
+++ [0905 21:22:43] Launching Kubernetes master components...
d06915c03700ef43c3629a00ca772d5e5278d4fc3c91a428135fc3912058304e
+++ [0905 21:22:44] Done. It may take about a minute before apiserver is up.
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

On the worker nodes we need to run ```kube-config enable-worker``` with the IP from the master node

```
$ sudo kube-config enable-worker 192.168.0.200
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
I also have to clean ```/var/lib/kubelet``` otherwise the dashboard pod will not start again. This reinitiate only has to be done on the master node. Just run ```sudo kube-config disable``` and ```sudo kube-config enable-master``` and hope everything will work after this ;-)
