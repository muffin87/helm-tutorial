# Part II: Installing helm

When you followed [Part I](../part-01/README.md) you should have a running dev
environment. In this part we will setup Helm. But first let's take a look
which components Helm has.

## Helm in a Handbasket

(Taken from the official [Helm Repo](https://github.com/kubernetes/helm)):

Helm is a tool that streamlines installing and managing Kubernetes applications.
Think of it like apt/yum/homebrew for Kubernetes.

+ Helm has two parts: a client `helm` and a server `tiller`.
+ Tiller runs inside of your Kubernetes cluster, and manages releases
(installations) of your charts.
+ Helm runs on your laptop, CI/CD, or wherever you want it to run.
+ Charts are Helm packages that contain at least two things:
    * A description of the package (`Chart.yaml`)
    * One or more templates, which contain Kubernetes manifest files
* Charts can be stored on disk, or fetched form remote chart repositories (like
Debian or RedHat packages).

## Installation

You first need the client `helm`. Since Helm is written in GoLang you simply need
to download the static compiled binaries and place them somewhere on your disk
and be sure that they have the rights that you can execute them. In addition
you can also find a Helm package for Homebrew and Chocolatey.

Homebrew:

```bash
brew install kubernetes-helm
```

Chocolaety:

```bash
choco install kubernetes-helm
```

For other Linux / Unix distributions please download the binaries on the 
[latest Release page](https://github.com/kubernetes/helm/releases/latest).

Next step would be to install / setup the server component `tiller`. You don't
need any additional things since `tiller` will be installed over `helm`.
You just would run: `helm init`. This would install the `tiller` in the 
`kube-system` by default. My personal recommendation is to use another Namespace
for Helm since `kube-system` is in most cases very sensible. Good for us that
the Helm Dev implemented a command line flag `--tiller-namespace` to define
a special Namespace.

*Note: I would like to point out that you MUST always aim to secure your tiller
in a production environment. For this tutorial we start off with a insecure
version. How to secure the tiller on different levels and will be touched in 
[Part VIII: Multi-tenancy installation](../part-08/README.md).*

```bash
helm init --tiller-namespace tools --debug
```

This will setup the `tiller` in the `tools` Namespace. I have also added the
`--debug` command that we have a little more more output what's going one.

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: helm
    name: tiller
  name: tiller-deploy
  namespace: tools
spec:
  replicas: 1
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: helm
        name: tiller
    spec:
      containers:
      - env:
        - name: TILLER_NAMESPACE
          value: tools
        - name: TILLER_HISTORY_MAX
          value: "0"
        image: gcr.io/kubernetes-helm/tiller:v2.9.1
        imagePullPolicy: IfNotPresent
        livenessProbe:
          httpGet:
            path: /liveness
            port: 44135
          initialDelaySeconds: 1
          timeoutSeconds: 1
        name: tiller
        ports:
        - containerPort: 44134
          name: tiller
        - containerPort: 44135
          name: http
        readinessProbe:
          httpGet:
            path: /readiness
            port: 44135
          initialDelaySeconds: 1
          timeoutSeconds: 1
        resources: {}
status: {}
---
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: helm
    name: tiller
  name: tiller-deploy
  namespace: tools
spec:
  ports:
  - name: tiller
    port: 44134
    targetPort: tiller
  selector:
    app: helm
    name: tiller
  type: ClusterIP
status:
  loadBalancer: {}
...
$HELM_HOME has been configured at /Users/mll0005f/.helm.

Tiller (the Helm server-side component) has been installed into your Kubernetes Cluster.

Please note: by default, Tiller is deployed with an insecure 'allow unauthenticated users' policy.
For more information on securing your installation see: https://docs.helm.sh/using_helm/#securing-your-helm-installation
Happy Helming!
```

What Helm does is simply apply an [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
which is the simplest high level API object in Kubernetes. We see that the
manifest files contains your wanted Namespace `namespace: tools`. We also see
that the Deployment uses a tiller image `image: gcr.io/kubernetes-helm/tiller:v2.9.1`
and we have a liveness probe which will check if our tiller is alive.

The seconds manifest file (after the `---`) is a [service](https://kubernetes.io/docs/concepts/services-networking/service/) 
object. A service object is an abstraction for your Pods. This follows the
famous [Pets vs Cattle analogy](http://cloudscaling.com/blog/cloud-computing/the-history-of-pets-vs-cattle/).
This basically means when we talk to our `tiller` we will always talk to the
service object (which will redirect our request to the ``tiller` Pod). We would
never talk to the Pod directly.

Let's check out if we see something inside our Kubernetes cluster:

```bash
kubectl -n tools get all -l name=tiller -o wide
NAME                                 READY     STATUS    RESTARTS   AGE       IP           NODE
pod/tiller-deploy-66998d5d74-9d2gc   1/1       Running   0          23m       172.17.0.4   minikube

NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE       SELECTOR
service/tiller-deploy   ClusterIP   10.97.201.183   <none>        44134/TCP   23m       app=helm,name=tiller

NAME                            DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE       CONTAINERS   IMAGES                                 SELECTOR
deployment.apps/tiller-deploy   1         1         1            1           23m       tiller       gcr.io/kubernetes-helm/tiller:v2.9.1   app=helm,name=tiller

NAME                                       DESIRED   CURRENT   READY     AGE       CONTAINERS   IMAGES                                 SELECTOR
replicaset.apps/tiller-deploy-66998d5d74   1         1         1         23m       tiller       gcr.io/kubernetes-helm/tiller:v2.9.1   app=helm,name=tiller,pod-template-hash=2255481830
```

There is the Deployment as well as the Service. The Replicaset and the Pod belongs
to the Deployment.

So now our `tiller` is running but how do you communicate with it? When you don't
use the default Namespace we need to tell your client where to find your `tiller`.

```bash
helm version
Client: &version.Version{SemVer:"v2.9.1", GitCommit:"20adb27c7c5868466912eebdf6664e7390ebe710", GitTreeState:"clean"}
```

You will notice that the command hangs. So let's try to define the `tiller` 
Namespace.

````bash
export TILLER_NAMESPACE=tools
helm version
Client: &version.Version{SemVer:"v2.9.1", GitCommit:"20adb27c7c5868466912eebdf6664e7390ebe710", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.9.1", GitCommit:"20adb27c7c5868466912eebdf6664e7390ebe710", GitTreeState:"clean"}
````

Now we have an answer from our server component. 

_Note: I would recommend to set this the `TILLER_NAMESPACE` env var (in your
`.bashrc`, `.zshrc` and so on) so that you don't have to set this manually every
time you launch a new shell._

So are we now finished already? Almost ... there is this little thing called
[Role-based access control (RBAC)](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
in our way. With RBAC you control who can do what. I have prepared a
ClusterRoleBinding for our local dev environment approach. Please follow 
[Part VIII](../part-08/README.md) for a more secure production ready solution.
Please apply / create the ClusterRoleBinding:

```bash
kubectl create -f part-02/templates/tiller-rbac.yml
clusterrolebinding.rbac.authorization.k8s.io/tiller created
```

With this ClusterRoleBinding you give the default service account in the `tools`
Namespace admin rights on cluster level.

Now you are ready to deploy our first `Chart`. We will cover this in
[Part III](../part-03/README.md).
