# Part III: Deploy your first chart

Now that we have everything in place you can deploy the first chart. To start
very simple we will take an already packaged and tested chart. To get one of
these `Charts` we will need to introduce the Helm repo management.

## Helm Repos

To easily manage share and reuse already packaged `Charts` the Helm Devs
developed repositories. For now we can define a `Charts` as a bundle of 
information necessary to create an instance of a Kubernetes application. A `Chart`
normally contains a lot of YAML based files. To better handle them as a unit
Helm packages and compress these files. We can say that Helm produces a shipment
artifact, which is normally stored as `<ChartName>.tar.gz`. To serve these
packaged `Charts` a simple web server with an special index file is used (In
[Part VI](../part-06/README.md) we will go into detail about that).

With the `helm repo` command you can easily manage repositories. You can list,
add, update and remove repo. By default Helm also setups some default repos.

```
helm repo list
NAME    URL
stable  https://kubernetes-charts.storage.googleapis.com
local   http://127.0.0.1:8879/charts
```

As you see two repos are already there. `stable` is the alias for the official
Helm repo. `local` as an alias when you want to host your own little repo (this
is mainly for development only use cases). We will take a look at the `stable`
repo. Currently you can't browse or explore a repo from the command line.
You need to check if the repo has a web frontend or something else. For the
`stable` repo you can just check out [Github](https://github.com/kubernetes/charts/tree/master/stable).

## Your first deployment

The first demo application you will deploy is [Dokuwiki](https://www.dokuwiki.org/dokuwiki).
Dokuwiki is a very simple open source wiki software. Before you can install it
I recommend to update the `stable` repo with:

```
helm repo update
Hang tight while we grab the latest from your chart repositories...
...Skip local chart repository
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈ Happy Helming!⎈
```

Let try the Helming:

```
helm install --name wiki stable/dokuwiki
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

So what happened? We have told `helm` that we want to install something.
When we deploy something with Helm we create a so called `Release`. This is
required and also a very good idea to give your release a name. We did this with
`--name wiki`. And the last part of the command specifies the path and the name
of the `Chart` we want to use `stable/dokuwiki`. 

The output will always contain some basic status information like the name of
the release, when the deployment happened, the Namespace where it was deployed
to and the status of the release.
After that helm will list all Kubernetes objects / resources which where created
with this release.
The last part contains additional information added by the author of the `Chart`
to get you started with the application you deployed. 

In this Minikube case we can't follow that instructions blindly. When you try to
follow step 1 you will notice that the result will be empty. This is because the
author added `type: LoadBalancer` in the service object for the wiki. Minikube
don't support that feature out of the box. When you list the service you will
see:

```
kubectl -n tools get svc -o wide
NAME            TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE       SELECTOR
tiller-deploy   ClusterIP      10.97.201.183    <none>        44134/TCP                    1d        app=helm,name=tiller
wiki-dokuwiki   LoadBalancer   10.102.226.119   <pending>     80:30597/TCP,443:32240/TCP   9h        app=dokuwiki
```

The external-ip is in a pending state. Anyways we can help us here with a simple
trick by using something called `port-forward`. For this we need to know the name
of the Pod we want to access:

```
kubectl -n tools get pods -o wide
NAME                             READY     STATUS    RESTARTS   AGE       IP           NODE
tiller-deploy-66998d5d74-9d2gc   1/1       Running   1          1d        172.17.0.4   minikube
wiki-dokuwiki-6bb9fbbb67-m97r4   1/1       Running   0          9h        172.17.0.5   minikube
```

To start the forwarding you can enter this following command. But be aware
that this command don't finished until you cancel it. When you cancel the
forwarding will stop working.

```
kubectl -n tools port-forward  wiki-dokuwiki-6bb9fbbb67-m97r4 8080:80
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

What happens here is that you get network access to the Pod TCP port 80.
Everything you send to 127.0.0.1 port 8080 will be redirect or forwarded to the
Pod. Our API server and the Kubelet will help us with this. For more details
checkout [this](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/).

_Note: When you want to forward the service to a port in range from 1 to 1024 you
will need to execute this command with privileged permissions._

Open a browser and hit the URL http://localhost:8080. You should now see your
wiki. To login you can now follow step 2 of the instructions.

## Manage Releases

Now that you installed a `Chart` we should take a look how you can manage your
releases. A very important feature is how you can find and see what releases
are currently deployed. You can do that by executing:

```
helm list
NAME    REVISION        UPDATED                         STATUS          CHART           NAMESPACE
wiki    1               Wed Jul 18 21:43:30 2018        DEPLOYED        dokuwiki-2.0.3  default
```

We see the name of the releases, the revision number (we will cover this more
deeply in [Part IV](../part-05/README.md)), when the last updated happened, that
status of the release, which `Chart` and version was deployed and in which
Namespace.

Let's assume you deployed to the wrong Namespace. We don't want the wiki to live
in the `default` Namespace. We want to in the `tools` Namespace. A good practise
to delete and redeploy the wiki right? ;)

To delete a release you need to now the name of the release. In our case we know
the name it's `wiki` and you also now how to look up all release. So that
should be a problem anymore. So let's delete the release with:

```
helm del wiki --purge
release "wiki" deleted
```

With the `--purge` option you make sure that really ALL resources of that release
should be deleted. Did that work? Let's do the `list` operation again:

```
helm ls
```

That seems to be empty. Let's check with `kubectl`.

```
kubectl get all --all-namespaces
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

So no wiki anymore? Yes it's gone! Okay fine let's redeploy it to the `tools`
Namespace.

```
helm install --name wiki --namespace tools stable/dokuwiki
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

The command stays pretty much the same except that you now set the Namespace
with the `--namespace` flag. If you like you can perform the same actions we did
when we first deployed the service (check the status of the Pods and do the 
port forwarding).

Now it's time to move on. In [Part IV](../part-04/README.md) you will create
a `Chart` from scratch. Don't worry you will repeat the management of `Charts`.
