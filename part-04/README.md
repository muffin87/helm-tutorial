# Part IV: Creating your own chart

In this part, we will create a basic `Chart` from scratch. The first thing we
need to look at is the structure of a `Chart`.

## Chart structure

You need to create a folder structure like this, starting with a folder named
like your application:

```bash
myAwesomeApp/
    Chart.yaml              # A YAML file containing information about the chart.
    LICENSE                 # OPTIONAL: A plain text file containing the license for the chart.
    README.md               # OPTIONAL: A human-readable README file.
    requirements.yaml       # OPTIONAL: A YAML file listing dependencies for the chart.
    vaules.yaml             # The default configuration values for this chart.
    charts/                 # OPTIONAL: A directory containing any charts upon which this chart depends.
    templates/              # A directory of templates that, when combined with values, will generate valid Kubernetes manifest files.
    templates/NOTES.txt     # OPTIONAL: A plain text file containing short usage notes
```

## Building a Chart

For our demo, we will package an application called [Ghost](https://github.com/tryghost).

```
Ghost is a free and open source blogging platform written in JavaScript and 
distributed under the MIT License, designed to simplify the process of online 
publishing for individual bloggers as well as online publications.
```

Ghost needs a place to persist data. Things like pictures and other binary data
will be placed on the filesystem. Other data needs to be stored
in a database. Since we want to start simple, we will use SQLite. We will also
not build the Docker image from scratch. Instead we will use the one available
on the [Docker Hub](https://hub.docker.com/_/ghost/).

_Note: I always recommend to build your own images when you run them in production.
You never know exactly what's inside an image! See [Backdoor Software](https://www.bleepingcomputer.com/news/security/17-backdoored-docker-images-removed-from-docker-hub/)_

First let's create the folder structure (somewhere on your local filesystem).

```bash
mkdir -p ghost/templates
cd ghost
touch Chart.yaml README.md values.yaml templates/NOTES.txt
```

Next you need to open this dir with an editor of your choice.
We start with editing the `Chart.yaml` file:

````yaml
name: ghost
home: https://ghost.org/de/
appVersion: 1.24.8
version: 1.0.0
description: Ghost is a free and open source blogging platform written in JavaScript and
  distributed under the MIT License, designed to simplify the process of online
  publishing for individual bloggers as well as online publications.
sources:
- https://hub.docker.com/_/ghost/
- https://github.com/TryGhost/Ghost
maintainers:
- name: Fabian Müller
  email: fabian.mueller@example.com
icon: http://www.juliosblog.com/content/images/2016/12/Ghost_icon.png
````

In this file we will collect the basic metadata for the chart. Every chart needs
a `name` a `version` and a `description`. With `version` we reference to the
version of the Chart. In addition there is the `appVersion` which reflects the
version of the packaged software. The `home` key is optional but useful
to point to the official website of an application. In addition to that, you can
define links to `sources`. It can also very helpful to define maintainers
of a chart, so you can contact that person in case of questions or
improvements of a specific chart. The `icon` is only important when you publish
the chart to a web frontend which lets you browse charts.

We will skip the `README.md` and the `values.yaml` for now. First let's jump
into the `templates` folder and create the Kubernetes resources we need. At this
point some basic Kubernetes knowledge kicks in. We know that Ghost is no
stateless application (we need to store things). Therefore we choose a 
[StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
as a base and not a Deployment. The next question we need to answer is, should
other services or users access Ghost. Since it's a blog software, I think we
can answer this with YES. Users should be able to access Ghost. So we need at
least a Service object and in most cases also an Ingress object.

````bash
cd templates
touch ghost_statefulset.yaml ghost_service.yaml ghost_ingress.yaml
````

Let's first setup the ingress:

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: {{ .Release.Name }}
  namespace: {{ .Release.Namespace }}

spec:
  rules:
  - host: "{{ .Values.ingressUrl }}.{{ .Values.global.domain }}"
    http:
      paths:
      - backend:
          serviceName: {{ .Release.Name }}
          servicePort: {{ .Values.statefulset.ports.port }}

```

This looks like a normal Kubernetes manifest file right? The only difference you
will notice is that we use Go templating to make the manifest file
dynamic. What does that mean exactly? The variables we define here, will be
replaced dynamically when the chart is rendered.

```yaml
name: {{ .Release.Name }}
```

`{{ .Release.Name }}` will be replaced with the values of the command line flag
`--name blog`. When the template is rendered you will see:

```yaml
name: blog
```

`Release` is another, so called, built-in object. For a list of built-in object please
take a look at [Build-In objects](https://docs.helm.sh/chart_template_guide/#built-in-objects).

The `Values` object is referencing the `values.yaml` file in the top level folder
of the chart. We will take a look at this later. At this point you only need to
know that every variable starting with `Values` is taken from `values.yaml`.

Now let's setup the Service object:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}
  namespace: {{ .Release.Namespace }}

  labels:
    app: {{ template "ghost.fullname" . }}
    heritage: {{ .Release.Service }}
    release: {{ .Release.Name }}
    chart: "{{ .Chart.Name }}-{{ .Chart.Version }}"
    component: {{ .Values.statefulset.labels.component }}

spec:
  selector:
    app: {{ template "ghost.fullname" . }}
    component: {{ .Values.statefulset.labels.component }}

  ports:
    - name: {{ .Values.statefulset.ports.name }}
      port: {{ .Values.statefulset.ports.port }}
      protocol: {{ .Values.statefulset.ports.protocol }}
      targetPort: {{ .Values.statefulset.ports.targetPort }}

```

In the Service object you will find the `Release` object and as well as some
vars starting with `Values`. There is a new object, the `Chart` object, that is
referencing the `Chart.yaml` file. There is another new part, which looks like
this:

```yaml
app: {{ template "ghost.fullname" . }}
```

With the `template` function, you can dynamically include complex blocks. It's
like an include function. This function can be helpful, when you need a block
multiple times. It also helps you clean up the yaml files and make them easier
to read through. In our case we will call the template with the name 
`ghost.fullname`. Helm will try to lookup a block with that name in 
the `_helpers.tpl` file located in the `templates` dir. Let's create this file
now:

```bash
touch templates/_helpers.tpl
```

And add the following template:

```gotemplate
{{/*
Create a default fully qualified app name.
We truncate at 63 chars because some Kubernetes name fields are limited to this (by the DNS naming spec).
If release name contains chart name it will be used as a full name.
*/}}
{{- define "ghost.fullname" -}}
{{- if .Values.fullnameOverride -}}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" -}}
{{- else -}}
{{- $name := default .Chart.Name .Values.nameOverride -}}
{{- if contains $name .Release.Name -}}
{{- .Release.Name | trunc 63 | trimSuffix "-" -}}
{{- else -}}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" -}}
{{- end -}}
{{- end -}}
{{- end -}}
```

As you can see, we use the keyword `define` with the name of the template and use
`end` to tell the interpreter where it ends. Notice how the template name is
prefixed with the name of our chart, to make sure the template is not
accidentialy overwritten. This template makes sure that the
app name will not reach more than 63 chars. For more information about these
named templates please check [here](https://docs.helm.sh/chart_template_guide/#named-templates).

The last object we need is a StatefulSet:

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

Nothing new here. The only file left is the `values.yaml`:

```yaml
# This is a YAML-formatted file.
# Declare variables to be passed into your template.

# Global var section
global:
  imagePullPolicy: "Always"                         # The Kubernetes image pull policy
  domain: "example"                                 # The domain to use
  timezone: "Europe/Berlin"                         # The default timezone

# Ghost section

# general section
ingressUrl: "myawesomeblog"                       # The Ingress URL to use

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
    initialDelaySeconds: 30                       # The initial delay before the readiness probe starts
    timeoutSeconds: 5                             # The timeout for the readiness probe

  # Liveness probe
  liveness:
    initialDelaySeconds: 30                       # The initial delay after the readiness probe has finished
    timeoutSeconds: 5                             # The timeout for liveness probe
    periodSeconds: 10                             # The interval in which the liveness probe should be executed

  # Port section
  ports:
    name: web                                     # The name of the port
    protocol: TCP                                 # The protocol to use
    port: 2368                                    # The port number to use
    targetPort: 2368                              # The target port for the service object

```

Here you will find all variables used in this chart.
Accessing the variables follows the normal YAML rules. When you want to use
for example the name of the port (`name: web`) you would reference it like this:

```gotemplate
{{ .Values.ghost.statefulset.ports.name }}
```

_Note: You can find the working chart in `part-04/templates/ghost`. If you face
problems with creating the chart, you can also take a look there and find
out what went wrong with your solution._

Now let's deploy our chart. I recommend using the `--debug` flag to enable
verbose output. In addition to that you can initially use `--dry-run` to render
the template to stdout instead of deploying it. This can be very helpful if
you want to check if the templates render correctly.

```
helm install --name blog --namespace=tools ./ghost --debug
```

```
[debug] Created tunnel using local port: '57598'

[debug] SERVER: "127.0.0.1:57598"

[debug] Original chart version: ""
[debug] CHART PATH: /Users/mll0005f/Documents/repository/github.com/helm-tutorial/part-04/templates/ghost

NAME:   blog
REVISION: 1
RELEASED: Mon Jul 23 19:52:22 2018
CHART: ghost-1.0.0
USER-SUPPLIED VALUES:
{}

COMPUTED VALUES:
global:
  domain: example
  imagePullPolicy: Always
  timezone: Europe/Berlin
ingressUrl: myawesomeblog
statefulset:
  dockerImage: ghost
  dockerTag: 1.24.8-alpine
  labels:
    component: frontend
  liveness:
    initialDelaySeconds: 30
    periodSeconds: 10
    timeoutSeconds: 5
  ports:
    name: web
    port: 2368
    protocol: TCP
    targetPort: 2368
  readiness:
    initialDelaySeconds: 30
    timeoutSeconds: 5
  replicas: 1
  resources:
    limits:
      cpu: 200m
      mem: 200Mi
    requests:
      cpu: 200m
      mem: 200Mi

HOOKS:
MANIFEST:

---
# Source: ghost/templates/ghost_service.yaml
apiVersion: v1
kind: Service
metadata:
  name: blog
  namespace: tools

  labels:
    app: blog-ghost
    heritage: Tiller
    release: blog
    chart: "ghost-1.0.0"
    component: frontend

spec:
  selector:
    app: blog-ghost
    component: frontend

  ports:
    - name: web
      port: 2368
      protocol: TCP
      targetPort: 2368
---
# Source: ghost/templates/ghost_statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: blog
  namespace: tools

  labels:
    app: blog-ghost
    heritage: Tiller
    release: blog
    chart: "ghost-1.0.0"
    component: "frontend"

spec:
  serviceName: blog
  replicas: 1

  selector:
    matchLabels:
      app: blog-ghost
      component: frontend

  template:
    metadata:
      labels:
        app: blog-ghost
        heritage: Tiller
        release: blog
        chart: "ghost-1.0.0"
        component: frontend

    spec:
      containers:
      - name: ghost

        image: "ghost:1.24.8-alpine"
        imagePullPolicy: "Always"

        readinessProbe:
          httpGet:
            path: /
            port: 2368
          initialDelaySeconds: 30
          timeoutSeconds: 5

        livenessProbe:
          httpGet:
            path: /
            port: 2368
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5

        resources:
          requests:
            cpu: 200m
            memory: 200Mi
          limits:
            cpu: 200m
            memory: 200Mi

        ports:
        - name: web
          protocol: TCP
          containerPort: 2368

        volumeMounts:
        - mountPath: /var/lib/ghost/content
          name: content

  volumeClaimTemplates:
  - metadata:
      name: content
      namespace: tools
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: standard
      resources:
        requests:
          storage: 500Mi
---
# Source: ghost/templates/ghost_ingress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: blog
  namespace: tools

spec:
  rules:
  - host: "myawesomeblog.example"
    http:
      paths:
      - backend:
          serviceName: blog
          servicePort: 2368
LAST DEPLOYED: Mon Jul 23 19:52:22 2018
NAMESPACE: tools
STATUS: DEPLOYED

RESOURCES:
==> v1/Service
NAME  TYPE       CLUSTER-IP    EXTERNAL-IP  PORT(S)   AGE
blog  ClusterIP  10.107.65.68  <none>       2368/TCP  0s

==> v1/StatefulSet
NAME  DESIRED  CURRENT  AGE
blog  1        0        0s

==> v1beta1/Ingress
NAME  HOSTS                  ADDRESS  PORTS  AGE
blog  myawesomeblog.example  80       0s
```

As you can see in the output above, we get all manifest files rendered correctly.
When you face an error, there are often only two potential sources:

1) You template is wrong. For example, you reference a var which is not in
the `values.yaml`. Or you got a syntax error inside your templates. These
kind of error messages will come directly from Helm.
2) Your manifest file is not valid because you have forgotten a field in the
manifest which is required in order to deploy it. In this case the error message
will come from the Kubernetes API.

In some cases it's very hard to understand the error and get a feeling which
component sent the error message. My recommendation is to take a closer look at
the debug output. It's also worth a try to write the debug output to a file and
try to deploy the different resources with `kubectl`.

Now let's see if our blog is running by taking a look at the deployed resources:

```bash
kubectl -n tools get all -l "app=blog-ghost"
```

```
NAME    REVISION        UPDATED                         STATUS          CHART           NAMESPACE
blog    1               Sat Jul 21 09:18:33 2018        DEPLOYED        ghost-1.24.8    tools

NAME         READY     STATUS    RESTARTS   AGE
pod/blog-0   1/1       Running   0          14m

NAME           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/blog   ClusterIP   10.98.111.137   <none>        2368/TCP   26m

NAME                    DESIRED   CURRENT   AGE
statefulset.apps/blog   1         1         26m
```

That looks very good. Now let's try to access the blog by port forwarding
the web port:

```bash
kubectl -n tools port-forward blog-0 2368:2368
```

You should now be able to see the blog when you open yout browser and
enter `localhost:2368`.

In [Part V](../part-05/README.md) we will take a look at how to manipulate
releases.
