# Part VI: Setup our own chart repo

In this part we will setup our own chart repo. This can be really helpful
and also necessary when you want to share your charts with other teams or people.
It's also needed when you have a very complex bundle of applications which
needs to be deployed in one rush. We will cover dependency management in 
[Part 7](../part-07/README.md) of this guide.

## Chart repo

So what's needed to host a chart repo? It's less than you think. You basically
need a web server which serves a `index.yaml` and you need some packaged charts.
Since we use HTTP(S) to communicate with a web server you get your charts by
executing a GET request to the right URL. 

The structure of the a repo is also very simple:

```
charts/
    |
    |- index.yaml
    |
    |- mysql-1.0.0.tgz
    |
    |- mysql-1.0.0.tgz.prov
```

The `.tgz` file is a packaged chart (we will package our Ghost chart after we
setup the repo). And we have a so called [Provenance](https://docs.helm.sh/developing_charts/#helm-provenance-and-integrity)
which helps to prove the integrity of a package. This file is optional and we
will not cover it in this tutorial.

_Note: You should sign your packages in production when you have more than one
team which is consuming your charts. The link provided above will give your
detailed info how to set this up._

When we want to download the mysql chart and our base URL would be 
http://my-chart-repo.example.com/charts, we would do a GET on
http://my-chart-repo.example.com/charts/mysql-1.0.0.tgz. Simple isn't it?

You now wonder what's the `index.yaml` for? Well for some operations you need
some metadata of the charts since the only thing we have on web server side
are the packaged and compressed charts. And we only get the name and the chart
version with them. A operation where we need the `index.yaml` would be `helm
repo update` for example. But what metadata are inside that file?

```yaml
apiVersion: v1
entries:
  alpine:
    - created: 2017-10-06T16:23:20.499543808-06:00
      description: Deploy a basic mysql database
      digest: 515c58e5f79d8b2913a10cb400ebb6fa9c77fe813287afbacf1a0b897cd78727
      home: https://www.mysql.com/de/
      name: mysql
      sources:
      - https://github.com/kubernetes/helm
      urls:
      - https://www.mysql.com/de/
      version: 1.0.0
generated: 2017-10-06T16:23:20.499029981-06:00
```

We have some base metadata here. You should remember this information, they are
taken from the `Chart.yaml`. You can manually create the index files by
executing in the directory where you packaged charts resides.

```bash
helm repo index
```

You see, this is no rocket science. You have many options to host / build your
repo. Luckily there are many solution out. Some of them are:

* Host a static web server which is updated with CI / CD.
* [Use Amazon S3 / S3 Like Storage](https://github.com/hypnoglow/helm-s3)
* [Use Google Cloud Storage](https://docs.helm.sh/developing_charts/#google-cloud-storage)
* [Use Github Pages](https://docs.helm.sh/developing_charts/#github-pages-example)
* [chartmuseum](https://github.com/helm/chartmuseum)
* Commercial Software

The solutions have, of course, pro's and con's. You should check your use case
and your requirements to see what matches. In this tutorial we will use the
`chartmuseum` since I think it's a pretty flexible solution and not so hard
to setup.

## Chartmuseum

[chartmuseum](https://github.com/helm/chartmuseum) is a open source Helm Chart
repository with support for the most common object storage implementations.
It's developed by the Helm community and provides some very nice features such
as basic auth, tls encryption and multi tenancy. 

To install it we will use? Right Helm! To keep it simple we will use local
storage for our dev environment. But feel free to try out every object storage
implementation you like. The official Helm chart say's that you will need to
provide a `custom.yaml` values files to configure the persistence. 

```yaml
env:
  open:
    STORAGE: local
    DISABLE_API: false
    ALLOW_OVERWRITE: true
persistence:
  enabled: true
  accessMode: ReadWriteOnce
  size: 8Gi
  storageClass: standard
```


To install it execute:

```bash
helm install --name helm-repo --namespace tools -f custom.yaml stable/chartmuseum --debug
```

We have a new command line parameter here. The `-f <valuefile>` will provide the
option to specify an additional values file. You can add / override variables
which differ from the standard `values.yaml`. In our case we will override the
storage section to modify it for our needs. In addition we will add the 
`DISABLE_API` flag and set it to false (this is needed to upload charts) and
the `ALLOW_OVERWRITE` in order to push the same version of chart multiple times.

When everything has worked out you should now have a running Pod:

```
kubectl -n tools get all -l "app=chartmuseum"
NAME                                         READY     STATUS    RESTARTS   AGE
pod/helm-repo-chartmuseum-568b9f848d-xmhcr   1/1       Running   0          9m

NAME                            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/helm-repo-chartmuseum   ClusterIP   10.96.244.72   <none>        8080/TCP   9m

NAME                                    DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/helm-repo-chartmuseum   1         1         1            1           9m

NAME                                               DESIRED   CURRENT   READY     AGE
replicaset.apps/helm-repo-chartmuseum-568b9f848d   1         1         1         9m
```

Now let's have a look if the app is responding to a request:

```bash
kubectl -n tools port-forward helm-repo-chartmuseum-568b9f848d-xmhcr 8080:8080
```

Open a browser and enter the URL http://localhost:8080. You should see something
like this:

```
Welcome to ChartMuseum!

If you see this page, the ChartMuseum web server is successfully installed and working.

For online documentation and support please refer to the GitHub project.

Thank you for using ChartMuseum.
```

Now that our repo is ready for use, we can start packaging and uploading our
Ghost chart.

## Package and Upload

A chart can't be uploaded in this raw format. It needs to be packaged. Helm
provides a command for this called `package`. You simply need to tell the
command where you package is located (it needs to find the `Chart.yaml`).

```bash
helm package ghost/
Successfully packaged chart and saved it to: <absolute path to>/ghost-1.24.8.tgz
```

On your local disk you should now see a file called `ghost-1.24.8.tgz`. This
is our packaged chart. Before we can upload it we need to add our repo to your
local helm repos. You can do that by executing:

```bash
helm repo add helm-repo http://localhost:8080
"helm-repo" has been added to your repositories
```

Check if you can see the repo in our repo index:

```
helm repo list
NAME            URL
stable          https://kubernetes-charts.storage.googleapis.com
local           http://127.0.0.1:8879/charts
helm-repo       http://localhost:8080
```

Now we should try to update the repo:

_Note: You need to port-forward the chartmuseum Pod for the rest of this part._

```bash
helm repo update
Hang tight while we grab the latest from your chart repositories...
...Skip local chart repository
...Successfully got an update from the "helm-repo" chart repository
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈ Happy Helming!⎈
```

And we can try to search in our repo with:

```
curl -Lv http://localhost:8080/api/charts | jq '.'
 % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0*   Trying ::1...
* TCP_NODELAY set
* Connected to localhost (::1) port 8080 (#0)
> GET /api/charts HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/7.54.0
> Accept: */*
>
< HTTP/1.1 200 OK
< Content-Type: application/json; charset=utf-8
< X-Request-Id: b6ac6cd9-4266-4424-b995-b44bd8c62138
< Date: Sun, 22 Jul 2018 13:18:22 GMT
< Content-Length: 703
<
{ [703 bytes data]
100   703  100   703    0     0  27801      0 --:--:-- --:--:-- --:--:-- 28120
* Connection #0 to host localhost left intact
{}
```

Seems to be emtpy. So now let's push our chart to our repo (Note: you need
to be in the same folder as your packaged chart or you have to use relative or
absolute paths).

```bash
curl --data-binary "@ghost-1.24.8.tgz" http://localhost:8080/api/charts
{"saved":true}%
```

So we just `POST` to the `api/charts` endpoint with a simple `curl` command.
Now let's have a look again:

```
curl -Lv http://localhost:8080/api/charts | jq '.'
 % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0*   Trying ::1...
* TCP_NODELAY set
* Connected to localhost (::1) port 8080 (#0)
> GET /api/charts HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/7.54.0
> Accept: */*
>
< HTTP/1.1 200 OK
< Content-Type: application/json; charset=utf-8
< X-Request-Id: b6ac6cd9-4266-4424-b995-b44bd8c62138
< Date: Sun, 22 Jul 2018 14:28:33 GMT
< Content-Length: 703
<
{ [703 bytes data]
100   703  100   703    0     0  27801      0 --:--:-- --:--:-- --:--:-- 28120
* Connection #0 to host localhost left intact
{
  "ghost": [
    {
      "name": "ghost",
      "home": "https://ghost.org/de/",
      "sources": [
        "https://hub.docker.com/_/ghost/",
        "https://github.com/TryGhost/Ghost"
      ],
      "version": "1.24.8",
      "description": "Ghost is a free and open source blogging platform written in JavaScript and distributed under the MIT License, designed to simplify the process of online publishing for individual bloggers as well as online publications.",
      "maintainers": [
        {
          "name": "Fabian Müller",
          "email": "fabian.mueller@example.com"
        }
      ],
      "icon": "http://www.juliosblog.com/content/images/2016/12/Ghost_icon.png",
      "urls": [
        "charts/ghost-1.24.8.tgz"
      ],
      "created": "2018-07-22T14:21:38.009387148Z",
      "digest": "9f179591b6f9fa77ce642fe0bcc80919a2ddc3c06148d40d4d2c0eff1cb36a27"
    }
  ]
}
```

We use the same url prefix `/api/charts` but this time just do a `GET` operation.
Chartmuseum will return the `index.yaml` formatted as JSON. To make that output
more human readable I like to use the [jq JSON processor](https://stedolan.github.io/jq/).
We can see that our chart was uploaded and we see our metadata. There is also
another option to perform the upload. A plugin for helm which is called
[helm push](https://github.com/chartmuseum/helm-push). You can install it by
executing:

```bash
helm plugin install https://github.com/chartmuseum/helm-push
Downloading and installing helm-push v0.5.0 ...
https://github.com/chartmuseum/helm-push/releases/download/v0.5.0/helm-push_0.5.0_darwin_amd64.tar.gz
Installed plugin: push
```

Helm will automatically download the right binary for your operation system.
Now you can upload a chart like this:

```bash
helm push --force ghost-1.24.8.tgz helm-repo
Pushing ghost-1.24.8.tgz to helm-repo...
Done.
```

We have successfully pushed our chart. We should try if we can deploy Ghost from
our repo. We first need to update the repo data and than we can deploy the Chart:

```
helm repo update

helm install --name blog --namespace=tools helm-repo/ghost
NAME:   blog
LAST DEPLOYED: Sun Jul 22 16:57:45 2018
NAMESPACE: tools
STATUS: DEPLOYED

RESOURCES:
==> v1/StatefulSet
NAME  DESIRED  CURRENT  AGE
blog  1        1        0s

==> v1beta1/Ingress
NAME  HOSTS                  ADDRESS  PORTS  AGE
blog  myawesomeblog.example  80       0s

==> v1/Pod(related)
NAME    READY  STATUS             RESTARTS  AGE
blog-0  0/1    ContainerCreating  0         0s

==> v1/Service
NAME  TYPE       CLUSTER-IP     EXTERNAL-IP  PORT(S)   AGE
blog  ClusterIP  10.103.227.99  <none>       2368/TCP  0s
```

As you see we use our new repo instead of pointing to the local filesystem.
You can now verify that our release is running. We will finish this part by
cleanup our release.

```
helm del blog --purge
release "blog" deleted
```

In [Part VII](../part-07/README.md) we will look at dependency management.
