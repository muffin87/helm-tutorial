# Part III: Deploy your first chart

Now that we have everything in place, we can deploy our first Chart. To start
off simple, we will take an already packaged and tested Chart. To get one of
these `Charts` we will need to introduce the Helm repo management.

## Helm Repos

To easily ,manage share and reuse already packaged `Charts` the Helm Devs
developed repositories. For now we can define a `Chart` as a bundle of
information necessary to create an instance of a Kubernetes application. A `Chart`
normally contains a lot of YAML based files. To better handle them as a unit,
Helm packages and compresses these files. Helm produces a shipment
artifact, which is normally stored as `<ChartName>.tar.gz`. To serve these
packaged `Charts`, a simple web server with an special index file is used (In
[Part VI](../part-06/README.md) we will go into detail about that).

With the `helm repo` command, you can easily manage repositories. You can list,
add, update and remove them. Helm also setups some default repositories:

```
helm repo list
```

```
NAME    URL
stable  https://kubernetes-charts.storage.googleapis.com
local   http://127.0.0.1:8879/charts
```

There are two repositories by default:
 * `stable` is the alias for the official Helm repo
 * `local` as an a lias when you want to host your own little repo (this
is mainly for development only use cases).

Let's take a look at the `stable` repo. Currently you can't browse or explore
a repo from the command line. You need to check if the repo has a web frontend
or something else. For the `stable` repo you can just check out
[Github](https://github.com/kubernetes/charts/tree/master/stable) or the
[App Hub](https://hub.kubeapps.com/).

## Your first deployment

The first demo application you will deploy is [Dokuwiki](https://www.dokuwiki.org/dokuwiki).
Dokuwiki is a very simple open source wiki software. Before installing it,
I recommend updating the `stable` repo with:

```
helm repo update
```

```
Hang tight while we grab the latest from your chart repositories...
...Skip local chart repository
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈ Happy Helming!⎈
```

Let try the Helming:

```
helm install --name wiki stable/dokuwiki
```

```
NAME:   wiki
LAST DEPLOYED: Wed Jul 18 21:43:30 2018
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Secret
NAME           TYPE    DATA  AGE
wiki-dokuwiki  Opaque  1     0s

==> v1/PersistentVolumeClaim
NAME                    STATUS   VOLUME    CAPACITY  ACCESS MODES  STORAGECLASS  AGE
wiki-dokuwiki-apache    Pending  standard  0s
wiki-dokuwiki-dokuwiki  Pending  standard  0s

==> v1/Service
NAME           TYPE          CLUSTER-IP    EXTERNAL-IP  PORT(S)                     AGE
wiki-dokuwiki  LoadBalancer  10.100.2.214  <pending>    80:32179/TCP,443:32636/TCP  0s

==> v1beta1/Deployment
NAME           DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
wiki-dokuwiki  1        1        1           0          0s

==> v1/Pod(related)
NAME                            READY  STATUS   RESTARTS  AGE
wiki-dokuwiki-6bb9fbbb67-sx59p  0/1    Pending  0         0s


NOTES:

** Please be patient while the chart is being deployed **

1. Get the DokuWiki URL by running:

** Please ensure an external IP is associated to the wiki-dokuwiki service before proceeding **
** Watch the status using: kubectl get svc --namespace default -w wiki-dokuwiki **

  export SERVICE_IP=$(kubectl get svc --namespace default wiki-dokuwiki -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
  echo http://$SERVICE_IP/

2. Login with the following credentials

  echo Username: user
  echo Password: $(kubectl get secret --namespace default wiki-dokuwiki -o jsonpath="{.data.dokuwiki-password}" | base64 --decode)
```

So, what happened? We told `helm` that we want to install something.
When we deploy with Helm, we create a so called `Release`. We gave our release a
name with `--name wiki` (if you don't supply a release name, Helm will generate
one for you. The last part of the command specifies the path and the name
of the `Chart` we want to use. In this case `stable/dokuwiki`.

The output will always contain some basic status information, like the name of
the release, when the deployment happened, the Namespace where it was deployed
to and the status of the release.
After that Helm will list all Kubernetes objects / resources which where created
with this release.
The last part contains additional information added by the author of the `Chart`
to get you started with the application you just deployed.

In this Minikube example we can't follow these instructions blindly. When you
try to follow step 1, you will notice that the result will be empty. This is
because the author added `type: LoadBalancer` to the service object of the wiki.
Minikube doesn't support that feature out of the box. When you list the services
you will see:

```
kubectl -n tools get svc -o wide
```

```
NAME            TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE       SELECTOR
tiller-deploy   ClusterIP      10.97.201.183    <none>        44134/TCP                    1d        app=helm,name=tiller
wiki-dokuwiki   LoadBalancer   10.102.226.119   <pending>     80:30597/TCP,443:32240/TCP   9h        app=dokuwiki
```

The external-ip is in a pending state. We can use a simple trick to access our 
wiki anyways - `kubectl port-forward`. For this to work, we need to know the 
name of the Pod we want to access:

```bash
kubectl -n tools get pods -o wide
```

```
NAME                             READY     STATUS    RESTARTS   AGE       IP           NODE
tiller-deploy-66998d5d74-9d2gc   1/1       Running   1          1d        172.17.0.4   minikube
wiki-dokuwiki-6bb9fbbb67-m97r4   1/1       Running   0          9h        172.17.0.5   minikube
```

To start the forwarding run the following command:

```
kubectl -n tools port-forward  wiki-dokuwiki-6bb9fbbb67-m97r4 8080:80
```

```
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

Portforwarding is active as long as the command is running. So make sure to not
quit as long as you need to access to the page.

You get network access to the Pods TCP port 80.
Everything you send to 127.0.0.1 port 8080, will be redirected or forwarded to
the Pod. Our API server and the Kubelet will take care of this. For more details
checkout [this](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/).

_Note: When you want to forward a service to a port in the range from 1 to 1024
you will need to execute this command with privileged permissions._

Open a browser and hit the URL http://localhost:8080. You should now see your
wiki. To login you can now follow step 2 of the instructions.

## Manage Releases

Now that you installed a `Chart`, we should take a look how you can manage your
releases. A very important feature is seeing what releases are currently
deployed. You can do that by executing:

```bash
helm list
```

```
NAME    REVISION        UPDATED                         STATUS          CHART           NAMESPACE
wiki    1               Wed Jul 18 21:43:30 2018        DEPLOYED        dokuwiki-2.0.3  default
```

We see the name of the releases, the revision number (we will cover this more
deeply in [Part IV](../part-05/README.md)), when the last update happened, the
status of the release, which `Chart` and version was deployed and in which
Namespace.

Let's assume you deployed to the wrong Namespace. We don't want the wiki to live
in the `default` Namespace. We want it in the `tools` Namespace. Let's delete
and redeploy the wiki! ;)

To delete a release, you need to know the name of the release. In our case, we
know the name is `wiki`. But we also know how to look up releases, so that
should be no problem anymore. Let's delete the release with:

```bash
helm del wiki --purge
```

```
release "wiki" deleted
```

With the `--purge` option, you make sure that really ALL resources of that release
are deleted. Did that work? Let's do the `list` operation again:

```bash
helm ls
```

Seems to be empty. Let's check with `kubectl`.

```
kubectl get all --all-namespaces
```

```
NAMESPACE     NAME                                        READY     STATUS    RESTARTS   AGE
kube-system   pod/etcd-minikube                           1/1       Running   0          1h
kube-system   pod/kube-addon-manager-minikube             1/1       Running   1          1d
kube-system   pod/kube-apiserver-minikube                 1/1       Running   0          1h
kube-system   pod/kube-controller-manager-minikube        1/1       Running   0          1h
kube-system   pod/kube-dns-86f4d74b45-dbtdx               3/3       Running   4          1d
kube-system   pod/kube-proxy-xkdrq                        1/1       Running   0          1h
kube-system   pod/kube-scheduler-minikube                 1/1       Running   0          1h
kube-system   pod/kubernetes-dashboard-5498ccf677-x876c   1/1       Running   3          1d
kube-system   pod/storage-provisioner                     1/1       Running   3          1d
tools         pod/tiller-deploy-66998d5d74-9d2gc          1/1       Running   1          1d

NAMESPACE     NAME                           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
default       service/kubernetes             ClusterIP   10.96.0.1       <none>        443/TCP         1d
kube-system   service/kube-dns               ClusterIP   10.96.0.10      <none>        53/UDP,53/TCP   1d
kube-system   service/kubernetes-dashboard   NodePort    10.104.52.84    <none>        80:30000/TCP    1d
tools         service/tiller-deploy          ClusterIP   10.97.201.183   <none>        44134/TCP       1d

NAMESPACE     NAME                        DESIRED   CURRENT   READY     UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
kube-system   daemonset.apps/kube-proxy   1         1         1         1            1           <none>          1d

NAMESPACE     NAME                                   DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
kube-system   deployment.apps/kube-dns               1         1         1            1           1d
kube-system   deployment.apps/kubernetes-dashboard   1         1         1            1           1d
tools         deployment.apps/tiller-deploy          1         1         1            1           1d

NAMESPACE     NAME                                              DESIRED   CURRENT   READY     AGE
kube-system   replicaset.apps/kube-dns-86f4d74b45               1         1         1         1d
kube-system   replicaset.apps/kubernetes-dashboard-5498ccf677   1         1         1         1d
tools         replicaset.apps/tiller-deploy-66998d5d74          1         1         1         1d
```

So no more wiki? Yes! It's gone! Okay fine, let's redeploy it to the `tools`
Namespace.

```
helm install --name wiki --namespace tools stable/dokuwiki
```

```
NAME:   wiki
LAST DEPLOYED: Wed Jul 18 22:11:07 2018
NAMESPACE: tools
STATUS: DEPLOYED

RESOURCES:
==> v1/Secret
NAME           TYPE    DATA  AGE
wiki-dokuwiki  Opaque  1     1s

==> v1/PersistentVolumeClaim
NAME                    STATUS  VOLUME                                    CAPACITY  ACCESS MODES  STORAGECLASS  AGE
wiki-dokuwiki-apache    Bound   pvc-b501d05b-8ac6-11e8-9bd0-080027ac8d4e  1Gi       RWO           standard      1s
wiki-dokuwiki-dokuwiki  Bound   pvc-b50482f5-8ac6-11e8-9bd0-080027ac8d4e  8Gi       RWO           standard      1s

==> v1/Service
NAME           TYPE          CLUSTER-IP      EXTERNAL-IP  PORT(S)                     AGE
wiki-dokuwiki  LoadBalancer  10.102.226.119  <pending>    80:30597/TCP,443:32240/TCP  0s

==> v1beta1/Deployment
NAME           DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
wiki-dokuwiki  1        1        1           0          0s

==> v1/Pod(related)
NAME                            READY  STATUS   RESTARTS  AGE
wiki-dokuwiki-6bb9fbbb67-m97r4  0/1    Pending  0         0s


NOTES:

** Please be patient while the chart is being deployed **

1. Get the DokuWiki URL by running:

** Please ensure an external IP is associated to the wiki-dokuwiki service before proceeding **
** Watch the status using: kubectl get svc --namespace tools -w wiki-dokuwiki **

  export SERVICE_IP=$(kubectl get svc --namespace tools wiki-dokuwiki -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
  echo http://$SERVICE_IP/

2. Login with the following credentials

  echo Username: user
  echo Password: $(kubectl get secret --namespace tools wiki-dokuwiki -o jsonpath="{.data.dokuwiki-password}" | base64 --decode)
```

The command stays pretty much the same, except that we now set the Namespace
with the `--namespace` flag. If you like you can perform the same actions we did
when we first deployed the service (check the status of the Pods and do the 
port forwarding).

When you are finished exploring the wiki you should delete it again:

```bash
helm del wiki --purge
release "wiki" deleted
```

Now it's time to move on. In [Part IV](../part-04/README.md) you will create
a `Chart` from scratch. Don't worry we will also repeat the management of
`Charts`.
