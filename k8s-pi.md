# Kubernetes on (vanilla) Raspbian Lite

Yes - you can create a Kubernetes cluster with Raspberry Pis with the default operating system called Raspbian. This means you can carry on using all the tools and packages you're used to with the officially-supported OS.

## Pre-reqs:

* You must use an RPi 2 or 3 for use with Kubernetes
* I'm assuming you're using wired ethernet (Wi-Fi also works, but it's not recommended)

## Master node setup

* Flash Raspbian to a fresh SD card.

You can use [Etcher.io](https://etcher.io) to burn the SD card.

Before booting set up an empty file called `ssh` in /boot/ on the SD card.

Use Raspbian Stretch Lite

> Update: I previously recommended downloading Raspbian Jessie instead of Stretch. At time of writing (3 Jan 2018) Stretch is now fully compatible.

https://www.raspberrypi.org/downloads/raspbian/

* Change hostname

Use the `raspi-config` utility to change the hostname to k8s-master-1 or similar and then reboot.

* Set a static IP address

It's not fun when your cluste breaks because the IP of your master changed. Let's fix that problem ahead of time:

```
cat >> /etc/dhcpcd.conf
```

Paste this block:

```
profile static_eth0
static ip_address=192.168.0.100/24
static routers=192.168.0.1
static domain_name_servers=8.8.8.8
```

Hit Control + D.

Change 100 for 101, 102, 103 etc.

You may also need to make a reservation on your router's DHCP table so these addresses don't get given out to other devices on your network.

* Install Docker

This installs 17.12 or newer.

```
$ curl -sSL get.docker.com | sh && \
sudo usermod pi -aG docker
```

* Disable swap

For Kubernetes 1.7 and newer you will get an error if swap space is enabled.

Turn off swap:

```
$ sudo dphys-swapfile swapoff && \
  sudo dphys-swapfile uninstall && \
  sudo update-rc.d dphys-swapfile remove
```

This should now show no entries:

```
$ sudo swapon --summary
```

* Edit `/boot/cmdline.txt`

Add this text at the end of the line, but don't create any new lines:

```
cgroup_enable=cpuset cgroup_enable=memory
```

> Some people in the comments suggest `cgroup_memory=memory` should now be: `cgroup_memory=1`.

Now reboot - do not skip this step.

* Add repo lists & install kubeadm

```
$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add - && \
  echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list && \
  sudo apt-get update -q && \
  sudo apt-get install -qy kubeadm
```

> I realise this says 'xenial' in the apt listing, don't worry. It still works.


* You now have two new commands installed:
 * kubeadm - used to create new clusters or join an existing one
 * kubectl - the CLI administration tool for Kubernetes

* Initialize your master node:

```
$ sudo kubeadm init --token-ttl=0
```

We pass in `--token-ttl=0` so that the token never expires - do not use this setting in production. The UX for `kubeadm` means it's currently very hard to get a join token later on after the initial token has expired. 

> Optionally also pass `--apiserver-advertise-address=192.168.0.27` with the IP of the Pi.

Note: This step will take a long time, even up to 15 minutes.

After the `init` is complete run the snippet given to you on the command-line:

```
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

This step takes the key generated for cluster administration and makes it available in a default location for use with `kubectl`.

* Now save your join-token

Your join token is valid for 24 hours, so save it into a text file. Here's an example of mine:

```
$ kubeadm join --token 9e700f.7dc97f5e3a45c9e5 192.168.0.27:6443 --discovery-token-ca-cert-hash sha256:95cbb9ee5536aa61ec0239d6edd8598af68758308d0a0425848ae1af28859bea
```

* Check everything worked:

```
$ kubectl get pods --namespace=kube-system
NAME                           READY     STATUS    RESTARTS   AGE                
etcd-of-2                      1/1       Running   0          12m                
kube-apiserver-of-2            1/1       Running   2          12m                
kube-controller-manager-of-2   1/1       Running   1          11m                
kube-dns-66ffd5c588-d8292      3/3       Running   0          11m                
kube-proxy-xcj5h               1/1       Running   0          11m                
kube-scheduler-of-2            1/1       Running   0          11m                
weave-net-zz9rz                2/2       Running   0          5m 
```

You should see the "READY" count showing as 1/1 for all services as above. DNS uses three pods, so you'll see 3/3 for that.

* Setup networking

Install Weave network driver

```
$ kubectl apply -f \
 "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

### Join other nodes

On the other RPis, repeat everything apart from `kubeadm init`.

* Change hostname

Use the `raspi-config` utility to change the hostname to k8s-worker-1 or similar and then reboot.

* Join the cluster

Replace the token / IP for the output you got from the master node:

```
$ sudo kubeadm join --token 1fd0d8.67e7083ed7ec08f3 192.168.0.27:6443
```

You can now run this on the master:

```
$ kubectl get nodes
NAME      STATUS     AGE       VERSION
k8s-1     Ready      5m        v1.7.4
k8s-2     Ready      10m       v1.7.4
```

## Deploy a container

This container will expose a HTTP port and convert Markdown to HTML. Just post a body to it via `curl` - follow the instructions below. 

*function.yml*

```
apiVersion: v1
kind: Service
metadata:
  name: markdownrender
  labels:
    app: markdownrender
spec:
  type: NodePort
  ports:
    - port: 8080
      protocol: TCP
      targetPort: 8080
      nodePort: 31118
  selector:
    app: markdownrender
---
apiVersion: apps/v1beta1 # for versions before 1.6.0 use extensions/v1beta1
kind: Deployment
metadata:
  name: markdownrender
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: markdownrender
    spec:
      containers:
      - name: markdownrender
        image: functions/markdownrender:latest-armhf
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
          protocol: TCP
```

Deploy and test:

```
$ kubectl create -f function.yml
```

Once the Docker image has been pulled from the hub and the Pod is running you can access it via `curl`: 

```
$ curl -4 http://127.0.0.1:31118 -d "# test"
<p><h1>test</h1></p>
```

If you want to call the service from a remote machine such as your laptop then use the IP address of your Kubernetes master node and try the same again.

## Start up the dashboard

The dashboard can be useful for visualising the state and health of your system but it does require the equivalent of "root" in the cluster. If you want to proceed you should first run in a [ClusterRole from the docs](https://github.com/kubernetes/dashboard/wiki/Access-control#admin-privileges).

```
echo -n 'apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
  labels:
    k8s-app: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard
  namespace: kube-system' | kubectl apply -f -
```

This is the development/alternative dashboard which has TLS disabled and is easier to use.

```
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/alternative/kubernetes-dashboard-arm.yaml
```

You can then find the IP and port via `kubectl get svc -n kube-system`. To access this from your laptop you will need to use `kubectl proxy` and navigate to `http://localhost:8001/` on the master, or tunnel to this address with `ssh`.

## Remove the test deployment

Now on the Kubernetes master remove the test deployment:

```
$ kubectl delete -f function.yml
```

### Moving on

Now head back over to the tutorial and deploy OpenFaaS to put the cluster through its paces.
