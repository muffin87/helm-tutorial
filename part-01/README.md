# Part I: Setting up a local dev environment

A very simple approach to get started is [Minikube](https://github.com/kubernetes/minikube).
Minikube is a single node Kubernetes cluster which should be sufficient for our
use case here.

_Note: Since we only need a working Kubernetes API you could also use any other
cluster you like. I only tested this tutorial on K8S version 1.9.X, 1.10.X and 
1.11.X._

Before you can install Minikube, you need to install the Kubernetes command line
client called `kubectl`. You can get it [here](https://kubernetes.io/docs/tasks/tools/install-kubectl/).
You need `kubectl` because Minikube will setup a so called `context` during startup.
A `context` is like a pointer and can be very helpful when you need to manage
multiple Kubernetes cluster. You can switch between the `context` and point ot
another cluster. In addition Helm will also use your `kubectl` config file to
find your cluster...

After that please follow the instructions on the 
[Minikube Github Page](https://github.com/kubernetes/minikube) and install 
it now. Please play attention to the dependencies of Minikube. When you are done, 
open a terminal and start Minikube.

```bash
minikube start
Starting local Kubernetes v1.10.0 cluster...
Starting VM...
Downloading Minikube ISO
 160.27 MB / 160.27 MB [============================================] 100.00% 0s
Getting VM IP address...
Moving files into cluster...
Downloading kubeadm v1.10.0
Downloading kubelet v1.10.0
Finished Downloading kubelet v1.10.0
Finished Downloading kubeadm v1.10.0
Setting up certs...
Connecting to cluster...
Setting up kubeconfig...
Starting cluster components...
Kubectl is now configured to use the cluster.
Loading cached images from config file.
```

After the start up you should check if you can interact with the Kubernetes API.

```bash
kubectl get node
NAME       STATUS    ROLES     AGE       VERSION
minikube   Ready     master    47s       v1.10.0
```

Additionally take a look at the running Pods.

```bash
kubectl get pods -o wide --all-namespaces
NAMESPACE     NAME                                    READY     STATUS    RESTARTS   AGE       IP           NODE
kube-system   etcd-minikube                           1/1       Running   0          6m        10.0.2.15    minikube
kube-system   kube-addon-manager-minikube             1/1       Running   0          6m        10.0.2.15    minikube
kube-system   kube-apiserver-minikube                 1/1       Running   0          5m        10.0.2.15    minikube
kube-system   kube-controller-manager-minikube        1/1       Running   0          6m        10.0.2.15    minikube
kube-system   kube-dns-86f4d74b45-dbtdx               3/3       Running   0          6m        172.17.0.2   minikube
kube-system   kube-proxy-c9t76                        1/1       Running   0          6m        10.0.2.15    minikube
kube-system   kube-scheduler-minikube                 1/1       Running   0          5m        10.0.2.15    minikube
kube-system   kubernetes-dashboard-5498ccf677-x876c   1/1       Running   0          5m        172.17.0.3   minikube
kube-system   storage-provisioner                     1/1       Running   0          5m        10.0.2.15    minikube
```

To make this tutorial a little bit more realistic you will now create some
[Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/).
A Namespace is a logic unit and I like to tread them as environments. I have
prepared some Namespaces to create them just execute:

```bash
kubectl apply -f part-01/templates/namespaces.yml
namespace/dev created
namespace/test created
namespace/prod created
namespace/tools created
```

Verify that everything worked as expected:

```bash
kubectl get namespaces
NAME          STATUS    AGE
default       Active    12m
dev           Active    41s
kube-public   Active    11m
kube-system   Active    12m
prod          Active    41s
test          Active    41s
tools         Active    41s
```

Now you are ready to go on with [Part 2](../part-02/README.md).
