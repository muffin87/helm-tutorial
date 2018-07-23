# Part V: Update, Rollback and what's under the hood

In this part we will take a look at what we can do with an already running
release. It's a common use case to manipulate a running release without
deleting and redeploying it as this would result in a downtime of our service.
With Helm you can update and also rollback releases. It makes use of the normal
Kubernetes strategies (rolling updates).

Let's start by updating our blog.

## Update

As an example we choose a very simple update. We want to update the version of
Ghost. This means we need to update the docker tag, which is defined in the 
`values.yaml`. In addition we need to change the Chart version in the
`Chart.yaml`

Now let's try to update our Chart. (I included the updated chart under
`part-05/templates/ghost`).

```
helm upgrade blog ghost/
```

```
Release "blog" has been upgraded. Happy Helming!
LAST DEPLOYED: Sat Jul 21 13:14:43 2018
NAMESPACE: tools
STATUS: DEPLOYED

RESOURCES:
==> v1/Service
NAME  TYPE       CLUSTER-IP     EXTERNAL-IP  PORT(S)   AGE
blog  ClusterIP  10.110.25.206  <none>       2368/TCP  1m

==> v1/StatefulSet
NAME  DESIRED  CURRENT  AGE
blog  1        1        1m

==> v1beta1/Ingress
NAME  HOSTS                  ADDRESS  PORTS  AGE
blog  myawesomeblog.example  80       1m

==> v1/Pod(related)
NAME    READY  STATUS   RESTARTS  AGE
blog-0  1/1    Running  0         1m
```

We use the `upgrade` command followed by the name of the release and the location
where the updated Chart can be found. Since we updated the Docker image,
Kubernetes will handle the update for us. This means it will terminate the Pod
and create a new one with the new Docker image. To verify that our changed
was successfully applied, we can do the following:

```
kubectl -n tools describe pod blog-0 | grep Image
   Image:          ghost:1.24.9-alpine
   Image ID:       docker-pullable://ghost@sha256:6aa66615f623a35289ab5f3f9026617fb2f2ab07d9d45f2091dad9381139a3ab
``` 

That looks very good. You could now try to access the blog again, to check if
the change didn't break anything.

_Note: The rolling update for a Deployment or a Statefulset will
not be triggered by every value you change in the manifest files. For example
if you change only a label your Pods will not be recreated automatically.
You would have to trigger the update manually by deleting the corresponding Pods._

So we did an update. But imagine you have an update which is causing trouble and
you need to go back to the state where everything was fine. This operation is
called rollback. Before we can execute the rollback, we need some background
information on how Helm manages the state of releases.

## What's under the hood?

So what magic does Helm use to keep track of releases. We can once again list
the Helm releases with:

```bash
helm list
```

```
NAME    REVISION        UPDATED                         STATUS          CHART           NAMESPACE
blog    2               Sat Jul 21 13:44:01 2018        DEPLOYED        ghost-1.24.9    tools
```

Let's focus on the revision part. Since we did an update from the blog release
we now have 2 revisions. Helm stores each revision as a ConfigMap in the
tiller namespace.

```bash
kubectl -n tools get configmap
```

```
NAME      DATA      AGE
blog.v1   1         19m
blog.v2   1         18m
```

As you see, we have two ConfigMaps for the blog release. `blog.v1` will have all
information about the first revision and `blog.v2` for the second one. So what's
inside these ConfigMaps:

```bash
kubectl -n tools get configmap blog.v1 -o yaml
```

```
apiVersion: v1
data:
  release: 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
kind: ConfigMap
metadata:
  creationTimestamp: 2018-07-21T11:42:45Z
  labels:
    MODIFIED_AT: "1532173441"
    NAME: blog
    OWNER: TILLER
    STATUS: SUPERSEDED
    VERSION: "1"
  name: blog.v1
  namespace: tools
  resourceVersion: "18965"
  selfLink: /api/v1/namespaces/tools/configmaps/blog.v1
  uid: 2f3b322d-8cdb-11e8-81ac-08002779694e
```

As you see we have a data field called `release` where Helm stores hashed
information about the release. In addition we see a lot of metadata about the
release such as the name and the version. We should now test if these
information let us rollback to the first revision.

## Rollback

For this use case Helm implemented a rollback function. We'll simply need to
execute the following command:

```
helm rollback blog 1
```

```
Rollback was a success! Happy Helming!
```

So we used the `rollback` command with the name of the release and the revision
number we want to rollback to. In our case `1` for the first release. Let's 
check if we are using the old docker image again:

```
kubectl -n tools describe pod blog-0 | grep Image
```

```
    Image:          ghost:1.24.8-alpine
    Image ID:       docker-pullable://ghost@sha256:0078e645794188d4d1d6c63455bce697a8eaf3a14435323b150a8b40ed2e03a6
```

So that worked. We did successfully roll back our release. Let's clean up our
blog.

```bash
helm del blog --purge
```

```
release "blog" deleted
```

Now that we can create and install Charts and also handle releases, we should
take a look at how to package and host Charts. This will be explained 
in [Part VI](../part-06/README.md).
