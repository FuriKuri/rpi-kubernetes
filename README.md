# Kubernetes on Raspberry PI
![The kube cluster](https://github.com/FuriKuri/rpi-kubernetes/raw/master/img/live.jpg "The kube cluster")

## Current todos / problems
- [x] Start first service
- [x] Start [ping-ping](https://github.com/FuriKuri/ping-pong) game
- [ ] The cluster do not survive a reboot :(

## Hardware

* 4 x Raspberry Pi 3 Model B [link](https://www.amazon.de/gp/product/B01CEFWQFA)
* 3 x Intermediate Stacking Plate [link](https://www.amazon.de/gp/product/B00NB1WQZW)
* 1 x Raspberry Pi case [link](https://www.amazon.de/gp/product/B00NB1WPEE)
* 1 x LAN cable 0,25m [5-pack] [link](https://www.amazon.de/gp/product/B01AXPQZPA)
* 4 x SD card [link](https://www.amazon.de/gp/product/B013UDL5RU)
* 1 x PowerPort 6 [link](https://www.amazon.de/gp/product/B00PTLSH9G)
* 1 x Micro USB cable [4-pack] [link](https://www.amazon.de/gp/product/B016BEVNK4)
* 1 x USB Powered Switch [link](https://www.amazon.de/gp/product/B003CLIOMU)

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

First, connect to your first RPI over ssh (use **pirate** as username and **hypriot** as password)

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

We also have to wait for specific pods, which will be started. This can take awhile, so be patient. Your master is fully ready when the following pods are running:

```
$ kubectl get pods --namespace=kube-system
NAME                                READY     STATUS    RESTARTS   AGE
k8s-master-192.168.0.200            4/4       Running   4          5m
k8s-proxy-192.168.0.200             1/1       Running   0          4m
kube-addon-manager-192.168.0.200    2/2       Running   0          2m
kube-dns-v17.1-s3iu2                3/3       Running   0          1m
kubernetes-dashboard-v1.1.1-km9od   1/1       Running   0          1m
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

## First interaction

For the following step, I used a simple hello world http server: [rpi-hello-kube](https://github.com/FuriKuri/rpi-hello-kube)

### Watch the running pods

The first image shows, that there is no pod running in the ```default``` namespace. The seconds shows the running pods in the namespace ```kube-system```.

![No running pods in default namespace](https://github.com/FuriKuri/rpi-kubernetes/raw/master/img/kube0.png "No running pods in default namespace")
![Running kube-system pods](https://github.com/FuriKuri/rpi-kubernetes/raw/master/img/kube1.png "Running kube-system pods")

### Create the first pod:

```
$ kubectl run hello-kube --image=furikuri/rpi-hello-kube:v1 --expose --port=3000 --labels="name=hello-kube"
service "hello-kube" created
deployment "hello-kube" created
```

### Check deployment

```
$ kubectl get deployments
NAME         DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
hello-kube   1         1         1            0           21s
```

### Check pods

Pod is starting...

```
$ kubectl get pods
NAME                          READY     STATUS              RESTARTS   AGE
hello-kube-1386070109-xi6im   0/1       ContainerCreating   0          35s
```
![Starting pod](https://github.com/FuriKuri/rpi-kubernetes/raw/master/img/service-start.png "Starting pod")

Pod is ready :)

```
$ kubectl get pods
NAME                          READY     STATUS    RESTARTS   AGE
hello-kube-1386070109-xi6im   1/1       Running   0          2m
```
![Running pod](https://github.com/FuriKuri/rpi-kubernetes/raw/master/img/service-running.png "Running pod")

### Check service / access pod (internal)
```
$ kubectl get services hello-kube
NAME         CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
hello-kube   10.0.0.4     <none>        3000/TCP   3m
```

```
$ curl http://10.0.0.4:3000
Hello World!
```

### Check logs

```
$ kubectl logs hello-kube-1386070109-xi6im
Received request for URL: /
```

### Check events

```
$ kubectl get events
LASTSEEN   FIRSTSEEN   COUNT     NAME                          KIND         SUBOBJECT                     TYPE      REASON                    SOURCE                     MESSAGE
22m        22m         1         192.168.0.200                 Node                                       Normal    Starting                  {kubelet 192.168.0.200}    Starting kubelet.
21m        22m         29        192.168.0.200                 Node                                       Normal    NodeHasSufficientDisk     {kubelet 192.168.0.200}    Node 192.168.0.200 status is now: NodeHasSufficientDisk
21m        22m         29        192.168.0.200                 Node                                       Normal    NodeHasSufficientMemory   {kubelet 192.168.0.200}    Node 192.168.0.200 status is now: NodeHasSufficientMemory
21m        21m         1         192.168.0.200                 Node                                       Normal    RegisteredNode            {controllermanager }       Node 192.168.0.200 event: Registered Node 192.168.0.200 in NodeController
7m         7m          1         192.168.0.201                 Node                                       Normal    Starting                  {kubelet 192.168.0.201}    Starting kubelet.
7m         7m          1         192.168.0.201                 Node                                       Normal    NodeHasSufficientDisk     {kubelet 192.168.0.201}    Node 192.168.0.201 status is now: NodeHasSufficientDisk
7m         7m          1         192.168.0.201                 Node                                       Normal    NodeHasSufficientMemory   {kubelet 192.168.0.201}    Node 192.168.0.201 status is now: NodeHasSufficientMemory
7m         7m          1         192.168.0.201                 Node                                       Normal    RegisteredNode            {controllermanager }       Node 192.168.0.201 event: Registered Node 192.168.0.201 in NodeController
7m         7m          1         192.168.0.201                 Node                                       Normal    NodeReady                 {kubelet 192.168.0.201}    Node 192.168.0.201 status is now: NodeReady
7m         7m          1         192.168.0.202                 Node                                       Normal    Starting                  {kubelet 192.168.0.202}    Starting kubelet.
7m         7m          1         192.168.0.202                 Node                                       Normal    NodeHasSufficientDisk     {kubelet 192.168.0.202}    Node 192.168.0.202 status is now: NodeHasSufficientDisk
7m         7m          1         192.168.0.202                 Node                                       Normal    NodeHasSufficientMemory   {kubelet 192.168.0.202}    Node 192.168.0.202 status is now: NodeHasSufficientMemory
7m         7m          1         192.168.0.202                 Node                                       Normal    RegisteredNode            {controllermanager }       Node 192.168.0.202 event: Registered Node 192.168.0.202 in NodeController
7m         7m          1         192.168.0.202                 Node                                       Normal    NodeReady                 {kubelet 192.168.0.202}    Node 192.168.0.202 status is now: NodeReady
7m         7m          1         192.168.0.203                 Node                                       Normal    Starting                  {kubelet 192.168.0.203}    Starting kubelet.
7m         7m          1         192.168.0.203                 Node                                       Normal    NodeHasSufficientDisk     {kubelet 192.168.0.203}    Node 192.168.0.203 status is now: NodeHasSufficientDisk
7m         7m          1         192.168.0.203                 Node                                       Normal    NodeHasSufficientMemory   {kubelet 192.168.0.203}    Node 192.168.0.203 status is now: NodeHasSufficientMemory
7m         7m          1         192.168.0.203                 Node                                       Normal    RegisteredNode            {controllermanager }       Node 192.168.0.203 event: Registered Node 192.168.0.203 in NodeController
7m         7m          1         192.168.0.203                 Node                                       Normal    NodeReady                 {kubelet 192.168.0.203}    Node 192.168.0.203 status is now: NodeReady
4m         4m          1         hello-kube-1386070109-xi6im   Pod                                        Normal    Scheduled                 {default-scheduler }       Successfully assigned hello-kube-1386070109-xi6im to 192.168.0.201
4m         4m          1         hello-kube-1386070109-xi6im   Pod          spec.containers{hello-kube}   Normal    Pulling                   {kubelet 192.168.0.201}    pulling image "furikuri/rpi-hello-kube:v1"
2m         2m          1         hello-kube-1386070109-xi6im   Pod          spec.containers{hello-kube}   Normal    Pulled                    {kubelet 192.168.0.201}    Successfully pulled image "furikuri/rpi-hello-kube:v1"
2m         2m          1         hello-kube-1386070109-xi6im   Pod          spec.containers{hello-kube}   Normal    Created                   {kubelet 192.168.0.201}    Created container with docker id 9eab2e9b4bc4
2m         2m          1         hello-kube-1386070109-xi6im   Pod          spec.containers{hello-kube}   Normal    Started                   {kubelet 192.168.0.201}    Started container with docker id 9eab2e9b4bc4
4m         4m          1         hello-kube-1386070109         ReplicaSet                                 Normal    SuccessfulCreate          {replicaset-controller }   Created pod: hello-kube-1386070109-xi6im
4m         4m          1         hello-kube                    Deployment                                 Normal    ScalingReplicaSet         {deployment-controller }   Scaled up replica set hello-kube-1386070109 to 1
22m        22m         1         moby-dock-1                   Node                                       Normal    Starting                  {kube-proxy moby-dock-1}   Starting kube-proxy.
7m         7m          1         moby-dock-2                   Node                                       Normal    Starting                  {kube-proxy moby-dock-2}   Starting kube-proxy.
7m         7m          1         moby-dock-3                   Node                                       Normal    Starting                  {kube-proxy moby-dock-3}   Starting kube-proxy.
7m         7m          1         moby-dock-4                   Node                                       Normal    Starting                  {kube-proxy moby-dock-4}   Starting kube-proxy.

```

### Check cluster info

```
$ kubectl cluster-info
Kubernetes master is running at http://localhost:8080
KubeDNS is running at http://localhost:8080/api/v1/proxy/namespaces/kube-system/services/kube-dns
kubernetes-dashboard is running at http://localhost:8080/api/v1/proxy/namespaces/kube-system/services/kubernetes-dashboard

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```
### Check kubectl configuration

```
$ kubectl config view
apiVersion: v1
clusters: []
contexts: []
current-context: ""
kind: Config
preferences: {}
users: []
```

### Scale pod
```
$ kubectl scale deployment hello-kube --replicas=3
deployment "hello-kube" scaled
```
![Scale service](https://github.com/FuriKuri/rpi-kubernetes/raw/master/img/service-scale.png "[Scale service")

```
$ kubectl get pods
NAME                          READY     STATUS              RESTARTS   AGE
hello-kube-1386070109-385j6   0/1       ContainerCreating   0          41s
hello-kube-1386070109-xi6im   1/1       Running             0          7m
hello-kube-1386070109-xwr66   0/1       ContainerCreating   0          41s
```

### Access from outside
In generell all service are available over the master. The master proxies the service, so that they are accessible from outside.

```
# theo at toad in ~/GitProjects/rpi-kubernetes on git:master ● [16:08:14]
→ curl -L http://192.168.0.200:8080/api/v1/proxy/namespaces/default/services/hello-kube
Hello World!
```

### Update deployment image

With ```kubectl set image``` for a deployment a new image can be set.

```
$ kubectl set image deployment/hello-kube hello-kube=furikuri/rpi-hello-kube:v2
deployment "hello-kube" image updated
```

After with command the new image will be roll out.

```
$ kubectl get pods
NAME                          READY     STATUS              RESTARTS   AGE
hello-kube-1386070109-6ct6f   1/1       Running             0          5m
hello-kube-1386070109-6zkhk   1/1       Terminating         0          4m
hello-kube-1386070109-p1646   1/1       Terminating         0          4m
hello-kube-1467465822-9y4el   0/1       ContainerCreating   0          1s
hello-kube-1467465822-g6mg2   1/1       Running             0          5s
hello-kube-1467465822-xbyl5   0/1       ContainerCreating   0          4s
```

For more information, how the image will be roll out, ```kubectl describe deployments``` shows the ```RollingUpdateStrategy``` for each deployment.

```
$ kubectl describe deployments
Name:			hello-kube
Namespace:		default
CreationTimestamp:	Sun, 18 Sep 2016 09:59:29 +0200
Labels:			name=hello-kube
Selector:		name=hello-kube
Replicas:		3 updated | 3 total | 3 available | 0 unavailable
StrategyType:		RollingUpdate
MinReadySeconds:	0
RollingUpdateStrategy:	1 max unavailable, 1 max surge
OldReplicaSets:		<none>
NewReplicaSet:		hello-kube-1467465822 (3/3 replicas created)
Events:
  FirstSeen	LastSeen	Count	From				SubobjectPath	Type		Reason			Message
  ---------	--------	-----	----				-------------	--------	------			-------
  18m		18m		1	{deployment-controller }			Normal		ScalingReplicaSet	Scaled up replica set hello-kube-1386070109 to 1
  16m		16m		1	{deployment-controller }			Normal		ScalingReplicaSet	Scaled up replica set hello-kube-1386070109 to 3
  12m		12m		1	{deployment-controller }			Normal		ScalingReplicaSet	Scaled up replica set hello-kube-1467465822 to 1
  12m		12m		1	{deployment-controller }			Normal		ScalingReplicaSet	Scaled down replica set hello-kube-1386070109 to 2
  12m		12m		1	{deployment-controller }			Normal		ScalingReplicaSet	Scaled up replica set hello-kube-1467465822 to 2
  12m		12m		1	{deployment-controller }			Normal		ScalingReplicaSet	Scaled down replica set hello-kube-1386070109 to 1
  12m		12m		1	{deployment-controller }			Normal		ScalingReplicaSet	Scaled up replica set hello-kube-1467465822 to 3
  12m		12m		1	{deployment-controller }			Normal		ScalingReplicaSet	Scaled down replica set hello-kube-1386070109 to 0
```

### Cleanup
```
$ kubectl delete service,deployment hello-kube
service "hello-kube" deleted
deployment "hello-kube" deleted
```

## The ping pong game
For this I created a simple application written in Go. This application is a simplified version of ping pong. We will start three services. Two services will be the client named ```ping``` and ```pong```. Both clients can be request over HTTP and they will return HIT or MISS, according as whether he hits the ball or not. The ```ping-pong-table``` will request both clients until one of them returns MISS.

### Start pods
For our ping pong game, we need to start 3 services, ```ping```, ```pong``` and ```ping-pong-table```. For all services, the image ```furikuri/rpi-ping-pong:latest``` can be used.
```
# ping client
$ kubectl run ping --expose --port=3000 --labels="name=ping" --image=furikuri/rpi-ping-pong:latest -- --hit-chance 85

# pong client
$ kubectl run pong --expose --port=3000 --labels="name=pong" --image=furikuri/rpi-ping-pong:latest -- --hit-chance 85

# ping pong server
$ kubectl run ping-pong-table --expose --port=3000 --labels="name=ping-pong-table" --image=furikuri/rpi-ping-pong:latest -- --mode server
```

Check with ```get pods``` if all needed pods are up.

```
$ kubectl get pods
NAME                              READY     STATUS    RESTARTS   AGE
ping-1834401500-0g10a             1/1       Running   0          9s
ping-pong-table-943113987-vxmtd   1/1       Running   0          8s
pong-2947596008-yocod             1/1       Running   0          9s
```

![Ping pong](https://github.com/FuriKuri/rpi-kubernetes/raw/master/img/ping-pong.png "Ping pong")

### Usage
For internal usage, you can use the service IP from our ```ping-pong-table```.
```
$ kubectl get service ping-pong-table
NAME               CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
ping-pong-table    10.0.0.179   <none>        3000/TCP   5m

$ curl http://10.0.0.179:3000
PING --> HIT
PONG --> HIT
PING --> HIT
PONG --> HIT
PING --> HIT
PONG --> HIT
PING --> HIT
PONG --> HIT
PING --> HIT
PONG --> HIT
PING --> MISS
The winner is PONG
```

For external use, just use the master proxy:

```
# theo at toad in ~/GitProjects/rpi-kubernetes on git:master ✖︎ [18:02:02]
→ curl http://192.168.0.200:8080/api/v1/proxy/namespaces/default/services/ping-pong-table/
PING --> HIT
PONG --> HIT
PING --> HIT
PONG --> HIT
PING --> HIT
PONG --> HIT
PING --> HIT
PONG --> HIT
PING --> HIT
PONG --> MISS
The winner is PING
```

### View logs
```
$ kubectl logs ping-pong-table-943113987-vxmtd
2016/09/06 16:02:03 Start game
2016/09/06 16:02:03 PING --> HIT
2016/09/06 16:02:03 PONG --> HIT
2016/09/06 16:02:03 PING --> HIT
2016/09/06 16:02:03 PONG --> HIT
2016/09/06 16:02:03 PING --> HIT
2016/09/06 16:02:03 PONG --> HIT
2016/09/06 16:02:03 PING --> HIT
2016/09/06 16:02:03 PONG --> HIT
2016/09/06 16:02:03 PING --> HIT
2016/09/06 16:02:03 PONG --> MISS
2016/09/06 16:02:03 The winner is PING
2016/09/06 16:02:03 End game
```


## Troubleshooting
At the moment the cluster will not work, if the master node will be rebooted. I found no clean way to get the master node working after a reboot. The only way, which is working for me, is to reinitiate the master node.
I also have to clean ```/var/lib/kubelet``` otherwise the dashboard pod will not start again. This reinitiate only has to be done on the master node. Just run ```sudo kube-config disable``` and ```sudo kube-config enable-master``` and hope everything will work after this ;-)
