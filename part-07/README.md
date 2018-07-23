# Part VII: Dependency management

In this part we will take a look at dependency management. With Helm you can
define dependencies which should be deployed together with your application. 
This can be really helpful, when you have a bunch of applications which need to
work together. Instead of creating a very big chart, which contains every 
application, you can try to split them up and create multiple charts.

_Note: It's very hard to give you a clear threshold when to split up your charts.
An indicator could be when the `values.yaml` gets very complex and too long._

## Requirements

When you define dependencies you need to create a file called `requirements.yaml`.
We will go through the process of setting up such a file, of course, with an
example: Let's take a look back at our Ghost application. In the first part we
ran our blog software with a SQLite database. Imagine you are now facing the
problem that the traffic on your running blog instance is very high and the
latency increases. After taking a look in your monitoring system, you notice
that the database is the bottleneck (high IO on the SQLite partition).
You now have a few options like using faster storage or try to add a cache.
For our example we will try to switch the database from SQLite to MySQL. With
MySQL you can use Master Slave replications which scale much better.
So we need to modify our Ghost Chart. First we will create a
`requirements.yaml` file:

```bash
touch requirements.yaml ghost/
```

And add the following lines to that file:

```yaml
dependencies:
- name: mysql
  version: "0.8.2"
  repository: "https://kubernetes-charts.storage.googleapis.com/"
```

You need to know the name of the Chart you want to include as well as the 
version and the repository where to find it. For MySQL you can look up this info
[here](https://github.com/helm/charts/blob/master/stable/mysql/Chart.yaml).

To match the new requirements we will need to change some things. Mainly we need
to tell Ghost to connect to our MySQL Database:

_Note: You may wonder why we don't convert the Statefulset to a Deployment since
we have a external database to persist our data. Ghost stores not all
information into the database. Images and other uploads will remain on disk. In
our case in the PersistentVolume of the Statefulset._

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: {{ .Release.Name }}
  namespace: {{ .Release.Namespace }}

  labels:
    app: {{ template "ghost.fullname" . }}
    heritage: {{ .Release.Service }}
    release: {{ .Release.Name }}
    chart: "{{ .Chart.Name }}-{{ .Chart.Version }}"
    component: "{{ .Values.statefulset.labels.component }}"

spec:
  serviceName: {{ .Release.Name }}
  replicas: {{ .Values.statefulset.replicas }}

  selector:
    matchLabels:
      app: {{ template "ghost.fullname" . }}
      component: {{ .Values.statefulset.labels.component }}

  template:
    metadata:
      labels:
        app: {{ template "ghost.fullname" . }}
        heritage: {{ .Release.Service }}
        release: {{ .Release.Name }}
        chart: "{{ .Chart.Name }}-{{ .Chart.Version }}"
        component: {{ .Values.statefulset.labels.component }}

    spec:
      containers:
      - name: {{ .Values.statefulset.dockerImage }}

        image: "{{ .Values.statefulset.dockerImage }}:{{ .Values.statefulset.dockerTag }}"
        imagePullPolicy: "{{ .Values.global.imagePullPolicy }}"

        env:
        - name: "database__client"
          value: "{{ .Values.databaseClient }}"
        - name: "database__connection__host"
          value: "{{ .Release.Name }}-mysql"
        - name: "database__connection__user"
          value: "{{ .Values.mysql.mysqlUser }}"
        - name: "database__connection__password"
          value: "{{ .Values.mysql.mysqlPassword }}"
        - name: "database__connection__database"
          value: "{{ .Values.mysql.mysqlDatabase }}"

        readinessProbe:
          httpGet:
            path: /
            port: {{ .Values.statefulset.ports.port }}
          initialDelaySeconds: {{ .Values.statefulset.readiness.initialDelaySeconds }}
          timeoutSeconds: {{ .Values.statefulset.readiness.timeoutSeconds }}

        livenessProbe:
          httpGet:
            path: /
            port: {{ .Values.statefulset.ports.port }}
          initialDelaySeconds: {{ .Values.statefulset.liveness.initialDelaySeconds }}
          periodSeconds: {{ .Values.statefulset.liveness.periodSeconds }}
          timeoutSeconds: {{ .Values.statefulset.liveness.timeoutSeconds }}

        resources:
          requests:
            cpu: {{ .Values.statefulset.resources.requests.cpu }}
            memory: {{ .Values.statefulset.resources.requests.mem }}
          limits:
            cpu: {{ .Values.statefulset.resources.limits.cpu }}
            memory: {{ .Values.statefulset.resources.limits.mem }}

        ports:
        - name: {{ .Values.statefulset.ports.name }}
          protocol: {{ .Values.statefulset.ports.protocol }}
          containerPort: {{ .Values.statefulset.ports.port }}

        volumeMounts:
        - mountPath: /var/lib/ghost/content
          name: content

  volumeClaimTemplates:
  - metadata:
      name: content
      namespace: {{ .Release.Namespace }}
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: standard
      resources:
        requests:
          storage: 500Mi

```

We added a bunch of environment variables to configure
the database connection for Ghost (see `env` section). All available env vars 
are listed [here](https://docs.ghost.org/docs/config#section-running-ghost-with-config-env-variables).

We also need to add some things to our `values.yaml`:

```yaml
# This is a YAML-formatted file.
# Declare variables to be passed into your template.

# Global var section
global:
  imagePullPolicy: "Always"                         # The Kubernetes image pull policy
  domain: "example"                                 # The domain to use

# Ghost section

# general section
ingressUrl: "myawesomeblog"                       # The Ingress URL to use
databaseClient: "mysql"                           # The database to use

# statefulset section
statefulset:
  replicas: 1                                     # The number of replicas when deployed
  dockerImage: "ghost"                            # The docker image to use
  dockerTag: "1.24.8-alpine"                      # The docker tags to use

  # Label section
  labels:
    component: "frontend"                         # The component label to use

  # Resource section
  resources:
    requests:
      mem: "200Mi"                                # The initial amount of memory ghost requests
      cpu: "200m"                                 # The initial amount of cpu ghost requests
    limits:
      mem: "200Mi"                                # The maximum amount of memory ghost can consume before it will get killed
      cpu: "200m"                                 # The maximum amount of cpu ghost can consume

  # Readiness probe
  readiness:
    initialDelaySeconds: 120                      # The initial delay before the readiness probe starts
    timeoutSeconds: 5                             # The timeout for the readiness probe

  # Liveness probe
  liveness:
    initialDelaySeconds: 120                      # The initial delay after the readiness probe has finished
    timeoutSeconds: 5                             # The timeout for liveness probe
    periodSeconds: 10                             # The interval in which the liveness probe should be executed

  # Port section
  ports:
    name: web                                     # The name of the port
    protocol: TCP                                 # The protocol to use
    port: 2368                                    # The port number to use
    targetPort: 2368                              # The target port for the service object

# MySQL section
mysql:

  # General
  mysqlUser: "ghost"                                # The username to use
  mysqlPassword: "myAwesomeBlog"                    # The password for the user
  mysqlDatabase: "ghost"                            # The database to use

```

We need to fill the environment vars mentioned in the Deployment. The
interesting part is located at the end of the file. There you see a new section
called `mysql`. This is used to actually set the user, password and database
in the mysql chart. Remember we included the mysql chart
as a dependency under the name `mysql`. To find the right variables to override
I recommend to take a look at the readme page of the [stable repo section](https://github.com/helm/charts/tree/master/stable/mysql#configuration).
This step can be really confusing since we override the values of another chart
in our Ghost chart.

_Note: I recommend to only override values in the top level chart and not in any
subchart._

_Note: I included all templates for this section in `part-07/templates`. When
you face problems feel free to take a look at them._

Before we can deploy our chart with dependencies we need to fetch them.
Helm also offers a command for this which is called `dependency update`.

```
helm dependency update ghost
```

```
Hang tight while we grab the latest from your chart repositories...
...Unable to get an update from the "local" chart repository (http://127.0.0.1:8879/charts):
        Get http://127.0.0.1:8879/charts/index.yaml: dial tcp 127.0.0.1:8879: connect: connection refused
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈Happy Helming!⎈
Saving 1 charts
Downloading mysql from repo https://kubernetes-charts.storage.googleapis.com/
Deleting outdated charts
```

Helm will try to find the `requirements.yaml` and resolves all listed
dependencies by downloading the corresponding charts. To verify what Helm
really did, we can take a look into our charts directory:

```
ls -al ghost/
total 32
drwxr-xr-x  9 mll0005f  631410114   288 22 Jul 19:12 .
drwxr-xr-x  3 mll0005f  631410114    96 22 Jul 18:30 ..
-rw-r--r--  1 mll0005f  631410114   515 22 Jul 18:27 Chart.yaml
-rw-r--r--  1 mll0005f  631410114     0 22 Jul 18:27 README.md
drwxr-xr-x  3 mll0005f  631410114    96 22 Jul 19:12 charts
-rw-r--r--  1 mll0005f  631410114   235 22 Jul 19:12 requirements.lock
-rw-r--r--  1 mll0005f  631410114   113 22 Jul 18:33 requirements.yaml
drwxr-xr-x  7 mll0005f  631410114   224 22 Jul 19:08 templates
-rw-r--r--  1 mll0005f  631410114  2628 22 Jul 19:09 values.yaml
```

We also find a new file called `requirements.lock` which contains the following data:

```yaml
dependencies:
- name: mysql
  repository: https://kubernetes-charts.storage.googleapis.com/
  version: 0.8.2
digest: sha256:8314e90806de6a8f2b6937679fdca7f13f172926d60e3c6838b0dd1f2c431c68
generated: 2018-07-22T19:12:20.36788332+02:00
```

With this lock file, Helm keeps track of pur dependencies.
Besides some metadata it also contains a sha256 hash which can be used to
verify the integrity of the downloaded package.
there is also another new thing in our chart folder. A folder called `charts`:

```
ls -al ghost/charts
```

```
total 24
drwxr-xr-x  3 mll0005f  631410114    96 22 Jul 19:12 .
drwxr-xr-x  9 mll0005f  631410114   288 22 Jul 19:12 ..
-rw-r--r--  1 mll0005f  631410114  8278 22 Jul 19:12 mysql-0.8.2.tgz
```

Helm will there place all charts which are listed as a dependency. Now we can
try to deploy our blog again. The command stays the same:

```
helm install --name blog --namespace=tools ./ghost
```

```
NAME:   blog
LAST DEPLOYED: Sun Jul 22 20:22:28 2018
NAMESPACE: tools
STATUS: DEPLOYED

RESOURCES:
==> v1/PersistentVolumeClaim
NAME        STATUS  VOLUME                                    CAPACITY  ACCESS MODES  STORAGECLASS  AGE
blog-mysql  Bound   pvc-30c309f0-8ddc-11e8-a620-08002779694e  8Gi       RWO           standard      0s

==> v1/Service
NAME        TYPE       CLUSTER-IP      EXTERNAL-IP  PORT(S)   AGE
blog-mysql  ClusterIP  10.102.155.20   <none>       3306/TCP  0s
blog        ClusterIP  10.103.242.207  <none>       2368/TCP  0s

==> v1beta1/Deployment
NAME        DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
blog-mysql  1        1        1           0          0s

==> v1/Deployment
blog  1  1  1  0  0s

==> v1beta1/Ingress
NAME  HOSTS                  ADDRESS  PORTS  AGE
blog  myawesomeblog.example  80       0s

==> v1/Pod(related)
NAME                         READY  STATUS             RESTARTS  AGE
blog-mysql-764569f779-wxnvk  0/1    Init:0/1           0         0s
blog-6fd6cd54b9-czsb9        0/1    ContainerCreating  0         0s

==> v1/Secret
NAME        TYPE    DATA  AGE
blog-mysql  Opaque  2     0s

==> v1/ConfigMap
NAME             DATA  AGE
blog-mysql-test  1     0s
```

When you take a look a the deployed resources you will see that the MySql DB
was deployed along with Ghost. Let's see if Ghost can access the database
by viewing the logs:

```
kubectl -n tools logs blog-6fd6cd54b9-czsb9 -f
[2018-07-22 17:27:39] INFO Creating table: posts
[2018-07-22 17:27:39] INFO Creating table: users
[2018-07-22 17:27:39] INFO Creating table: posts_authors
[2018-07-22 17:27:39] INFO Creating table: roles
[2018-07-22 17:27:39] INFO Creating table: roles_users
[2018-07-22 17:27:39] INFO Creating table: permissions
[2018-07-22 17:27:39] INFO Creating table: permissions_users
[2018-07-22 17:27:39] INFO Creating table: permissions_roles
[2018-07-22 17:27:39] INFO Creating table: permissions_apps
[2018-07-22 17:27:39] INFO Creating table: settings
[2018-07-22 17:27:39] INFO Creating table: tags
[2018-07-22 17:27:39] INFO Creating table: posts_tags
[2018-07-22 17:27:40] INFO Creating table: apps
[2018-07-22 17:27:40] INFO Creating table: app_settings
[2018-07-22 17:27:40] INFO Creating table: app_fields
[2018-07-22 17:27:40] INFO Creating table: clients
[2018-07-22 17:27:40] INFO Creating table: client_trusted_domains
[2018-07-22 17:27:40] INFO Creating table: accesstokens
[2018-07-22 17:27:40] INFO Creating table: refreshtokens
[2018-07-22 17:27:40] INFO Creating table: subscribers
[2018-07-22 17:27:40] INFO Creating table: invites
[2018-07-22 17:27:40] INFO Creating table: brute
[2018-07-22 17:27:40] INFO Creating table: webhooks
[2018-07-22 17:27:40] INFO Model: Tag
[2018-07-22 17:27:41] INFO Model: Client
[2018-07-22 17:27:41] INFO Model: Role
[2018-07-22 17:27:41] INFO Model: Permission
[2018-07-22 17:27:43] INFO Model: User
[2018-07-22 17:27:48] INFO Model: Post
[2018-07-22 17:27:50] INFO Relation: Role to Permission
[2018-07-22 17:27:52] INFO Relation: Post to Tag
[2018-07-22 17:27:52] INFO Relation: User to Role
[2018-07-22 17:28:13] INFO Finished database migration!
[2018-07-22 17:28:59] WARN Theme's file locales/en.json not found.
[2018-07-22 17:29:01] INFO Ghost is running in production...
[2018-07-22 17:29:01] INFO Your blog is now available on http://localhost:2368/
[2018-07-22 17:29:01] INFO Ctrl+C to shut down
[2018-07-22 17:29:01] INFO Ghost boot 47.103s
```

That looks very good. If you want, you can verify it by accessing the blog.
To complete this part of the tutorial, we will delete the blog again.

```
helm del blog --purge
```

```
release "blog" deleted
```
