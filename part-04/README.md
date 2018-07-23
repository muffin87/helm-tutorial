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
    charts/                 # A directory containing any charts upon which this chart depends.
    templates/              # A directory of templates that, when combined with values, will generate valid Kubernetes manifest files.
    tempaltes/NOTES.txt     # OPTIONAL: A plain text file containing short usage notes
```

## Building a Chart

For demo cases we will package an application called [Ghost](https://github.com/tryghost).

```
Ghost is a free and open source blogging platform written in JavaScript and 
distributed under the MIT License, designed to simplify the process of online 
publishing for individual bloggers as well as online publications.
```

Ghost needs a place to persist data. Things like pictures and other binary data
will be place on a filesystem and some other information needs to be stored
in a database. Since we want to start simple we will use SQLLite. We also will
not build the Docker image from scratch. We will use the one which is available
on the [Docker Hub](https://hub.docker.com/_/ghost/).

_Note: I always recommend that you build your own images when you run in production.
You never know exactly what's inside an image! See [Backdoor Software](https://www.bleepingcomputer.com/news/security/17-backdoored-docker-images-removed-from-docker-hub/)_

You first create the folder structure (somewhere on your local filesystem).

```bash
mkdir -p ghost/templates
cd ghost
touch Chart.yaml README.md values.yaml templates/NOTES.txt
```

Next you need to open this dir with an editor of your choice. We start editing 
the `Chart.yaml` file:

```yaml
name: ghost
home: https://ghost.org/de/
version: 1.24.8
description: Ghost is a free and open source blogging platform written in JavaScript and
  distributed under the MIT License, designed to simplify the process of online
  publishing for individual bloggers as well as online publications.
sources:
- https://hub.docker.com/_/ghost/
- https://github.com/TryGhost/Ghost
maintainers:
- name: Max Mustermann
  email: max.mustermann@example.com
icon: http://www.juliosblog.com/content/images/2016/12/Ghost_icon.png
```

In this file we will collect the basic metadata for the chart. Every chart needs
a `name` a `version` and a `description`. The `home` key is option but useful
to point to a official website of the application. In addition you can define
one or more link to `sources`. It can also very helpful to define maintainers
for a chart, so you can contact that person in case of questions for
improvements. The `icon` is only important when you publish the chart to a web
frontend which let you browse charts.

We will skip the `README.md` and the `values.yaml` for now. We will first jump
into the `templates` folder and create the Kubernetes resources we need. At this
point some basic Kubernetes knowledge kicks in. We know that Ghost is no
stateless application (we need to store things). Therefore we choose a 
[StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
as a base and not a Deployment. The next question we need to answer is should
other services or users access Ghost. Since it's a Blog software I think we
can answer with YES. Users should be able to access Ghost. So we need at least
a Service object and in most cases also an Ingress object.

````bash
cd templates
touch ghost_statefulset.yaml ghost_service.yaml ghost_ingress.yaml
````

Let first setup the ingress:

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: {{ .Release.Name }}
  namespace: {{ .Release.Namespace }}

spec:
  rules:
  - host: "{{ .Values.ghost.ingressUrl }}.{{ .Values.global.domain }}"
    http:
      paths:
      - backend:
          serviceName: {{ .Release.Name }}
          servicePort: {{ .Values.ghost.statefulset.ports.port }}
```

That looks like a normal Kubernetes manifest file right? The only difference you
will notice is that we use the GoLang templating to make the manifest file
dynamic. What does that mean? Well we use variables which will be replaced during 
rending time. 

```yaml
name: {{ .Release.Name }}
```

`{{ .Release.Name }}` will be replaced with the values of the command line flag
`--name blog`. When the template is rendered you will see:

```yaml
name: blog
```

`Release` is a so called build-in object. For a list of build-in object please
take a look at [Build-In objects](https://docs.helm.sh/chart_template_guide/#built-in-objects).

The `Values` object is referencing the `values.yaml` file in the top level folder
of the chart. We will take a look at it later. At this point you only need to
know that every variable starting with `Values` is taken from the `values.yaml`.

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
    component: "{{ .Values.ghost.statefulset.labels.component }}"

spec:
  selector:
    app: {{ template "ghost.fullname" . }}
    component: "{{ .Values.ghost.statefulset.labels.component }}"

  ports:
    - name: {{ .Values.ghost.statefulset.ports.name }}
      port: {{ .Values.ghost.statefulset.ports.port }}
      protocol: {{ .Values.ghost.statefulset.ports.protocol }}
      targetPort: {{ .Values.ghost.statefulset.ports.targetPort }}

```

In the Service object you will also find the `Release` object and also some 
vars starting with `Values`. There is a new object, the `Chart` object, that is
referencing the `Chart.yaml` file. Also we see

```yaml
app: {{ template "ghost.fullname" . }}
```

With the `template` function you can dynamically include complex blocks. It's 
like a include function. This function can be helpful, when you need a block 
multiple times. This also helps you clean up the yaml files and make it easier 
to read through. In our case we will call the template with the name 
`ghost.fullname`. Helm will try to lookup a block with that name in 
the `_helpers.tpl` file located in the `templates` dir. We will create that now:

```bash
touch templates/_helpers.tpl
```

And we will insert this template:

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

As you see we use the keyword `define` with the name of the template. and use
`end` to tell the interpreter where it ends. In this case we make sure that the
app name will not reach more than 63 chars. For more information about these
named template please see [here](https://docs.helm.sh/chart_template_guide/#named-templates).

And the last object, the Statefulset:

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
    component: "{{ .Values.ghost.statefulset.labels.component }}"

spec:
  serviceName: {{ .Release.Name }}
  replicas: {{ .Values.ghost.statefulset.replicas }}

  selector:
    matchLabels:
      app: {{ template "ghost.fullname" . }}
      component: "{{ .Values.ghost.statefulset.labels.component }}"

  template:
    metadata:
      labels:
        app: {{ template "ghost.fullname" . }}
        heritage: {{ .Release.Service }}
        release: {{ .Release.Name }}
        chart: "{{ .Chart.Name }}-{{ .Chart.Version }}"
        component: "{{ .Values.ghost.statefulset.labels.component }}"

    spec:
      containers:
      - name: {{ .Values.ghost.statefulset.dockerImage }}

        image: "{{ .Values.ghost.statefulset.dockerImage }}:{{ .Values.ghost.statefulset.dockerTag }}"
        imagePullPolicy: "{{ .Values.global.imagePullPolicy }}"

        readinessProbe:
          httpGet:
            path: /
            port: {{ .Values.ghost.statefulset.ports.port }}
          initialDelaySeconds: {{ .Values.ghost.statefulset.readiness.initialDelaySeconds }}
          timeoutSeconds: {{ .Values.ghost.statefulset.readiness.timeoutSeconds }}

        livenessProbe:
          httpGet:
            path: /
            port: {{ .Values.ghost.statefulset.ports.port }}
          initialDelaySeconds: {{ .Values.ghost.statefulset.liveness.initialDelaySeconds }}
          periodSeconds: {{ .Values.ghost.statefulset.liveness.periodSeconds }}
          timeoutSeconds: {{ .Values.ghost.statefulset.liveness.timeoutSeconds }}

        resources:
          requests:
            cpu: {{ .Values.ghost.statefulset.resources.requests.cpu }}
            memory: {{ .Values.ghost.statefulset.resources.requests.mem }}
          limits:
            cpu: {{ .Values.ghost.statefulset.resources.limits.cpu }}
            memory: {{ .Values.ghost.statefulset.resources.limits.mem }}

        ports:
        - name: {{ .Values.ghost.statefulset.ports.name }}
          protocol: {{ .Values.ghost.statefulset.ports.protocol }}
          containerPort: {{ .Values.ghost.statefulset.ports.port }}

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
ghost:

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

Here you will find all variables used in this chart. The iteration or navigation
through the file will follow the normal YAML rules. When you want to access the
name of the port for example `name: web` you would reference it like:

```gotemplate
{{ .Values.ghost.statefulset.ports.name }}
```

_Note: I included the ghost chart under `part-04/templates/ghost`.If you face 
problems with creation the chart you can also take a look there to find
out what went wrong._

Now we can try to deploy our chart. I recommend that you use the `--debug` flag.
With that flag Helm will print the rendered manifest files to stdout. In addition
you can use `--dry-run` to just render the template and not deploy it. This can
be very helpful when you just want to check if the templates are correctly
renders. 

```
helm install --name blog --namespace=tools ./ghost --debug
[debug] Created tunnel using local port: '51496'

[debug] SERVER: "127.0.0.1:51496"

[debug] Original chart version: ""
[debug] CHART PATH: /Users/mll0005f/Documents/repository/github.com/helm-tutorial/part-04/templates/ghost

NAME:   blog
REVISION: 1
RELEASED: Sat Jul 21 10:56:47 2018
CHART: ghost-1.24.8
USER-SUPPLIED VALUES:
{}

COMPUTED VALUES:
ghost:
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
global:
  domain: example
  imagePullPolicy: Always
  timezone: Europe/Berlin

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
    chart: "ghost-1.24.8"
    component: "frontend"

spec:
  selector:
    app: blog-ghost
    component: "frontend"

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
    chart: "ghost-1.24.8"
    component: "frontend"

spec:
  serviceName: blog
  replicas: 1

  selector:
    matchLabels:
      app: blog-ghost
      component: "frontend"

  template:
    metadata:
      labels:
        app: blog-ghost
        heritage: Tiller
        release: blog
        chart: "ghost-1.24.8"
        component: "frontend"

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
LAST DEPLOYED: Sat Jul 21 10:56:47 2018
NAMESPACE: tools
STATUS: DEPLOYED

RESOURCES:
==> v1/Service
NAME  TYPE       CLUSTER-IP    EXTERNAL-IP  PORT(S)   AGE
blog  ClusterIP  10.98.200.49  <none>       2368/TCP  0s

==> v1/StatefulSet
NAME  DESIRED  CURRENT  AGE
blog  1        1        0s

==> v1beta1/Ingress
NAME  HOSTS                  ADDRESS  PORTS  AGE
blog  myawesomeblog.example  80       0s

==> v1/Pod(related)
NAME    READY  STATUS             RESTARTS  AGE
blog-0  0/1    ContainerCreating  0         0s
```

So you see in the output above we get all rendered manifest files. When you face
an error, there are two potential sources:

1) You template got a problem. For example you reference a var which Helm don't
find in the `values.yaml`. Or you got a syntax error inside your templates. That
kind of error messages will come from Helm.
2) Your manifest file is not valid because you have forgot a field in the manifest
which is required in order to deploy it. In this case the error message
will come from the Kubernetes API.

In some cases it's very hard to find the error and also get a feeling who send
the error message. My recommendation is to take a closer look at the debug output.
It's also worth to write the debug output to a file and try to deploy the single
resources with `kubectl`.

Now let's see if our blog is running by taking a look at the deployed resources
again.

```
kubectl -n tools get all -l "app=blog-ghost"
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
the web port.

```bash
kubectl -n tools port-forward blog-0 2368:2368
```
You should now be able to see the blog when you open a browser and
enter `localhost:2368`.

In [Part V](../part-05/README.md) we will take a look how to manipulate releases.
