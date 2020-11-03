---
title: "How to deploy Elixir release to kubernetes, using Helm"
date: 2020-11-01T21:34:57-05:00
draft: true
---

This git repo is a companion to [my blog post about deploying an Elixir app to kubernetes](http://blog.jebelev.com/posts/elixir-helm-gcloud-deploy/).


Elixir has (or at least tended to have) reputation as a hard-to-deploy platform. The goal of this post is to show a way to quickly deploy an Elixir-based service to a kubernetes environment, using only Elixir releases, docker and helm. 

### Use Case: add a phoenix server to existing kubernetes cluster

Let's say you already have a configured kubernetes cluster (I'll be using Google Cloud in this example) and you want to add a web server to it - could be serving some api requests with JSON payloads, or just regular web content. We will go from creating a new mix project all the way to adding a new service on gcloud, with every step documented in this post. In order to keep the post at reasonable length I will not show database or load balancer setup here - those are optionals not needed by every project, and I may address them in a future post.

#### Create a new Phoenix Project: `hello`

```bash
$ mix phx.new hello --no-ecto
* creating hello/config/config.exs
* creating hello/config/dev.exs
* creating hello/config/prod.exs
...

Fetch and install dependencies? [Yn] y
* running mix deps.get
* running mix deps.compile
* running cd assets && npm install && node node_modules/webpack/bin/webpack.js --mode development

We are almost there! The following steps are missing:

    $ cd hello

Start your Phoenix app with:

    $ mix phx.server

You can also run your app inside IEx (Interactive Elixir) as:

    $ iex -S mix phx.server
```

**Generate a web secret (and record it somewhere)**

```bash
$ cd hello
$ mix phx.gen.secret
5aalTEIX45QHhmz7M3uRnTa9S/Ianrt7KsmQCvM1IniOj0IDkcFN6NuJcXQoNoFV
```

**Prepare directory for static files**

```bash
$ mix phx.digest
Check your digested files at "priv/static"
```

**Check that the server starts and serves wed pages.**

```bash
$ mix phx.server
Compiling 13 files (.ex)
Generated hello app
[info] Running HelloWeb.Endpoint with cowboy 2.8.0 at 0.0.0.0:4000 (http)
[info] Access HelloWeb.Endpoint at http://localhost:4000

webpack is watching the files…

Hash: 61704d6726d360a88d02
Version: webpack 4.41.5
Time: 476ms
Built at: 11/01/2020 10:49:29 PM
                Asset       Size  Chunks                   Chunk Names
       ../css/app.css   10.6 KiB     app  [emitted]        app
   ../css/app.css.map   13.4 KiB     app  [emitted] [dev]  app
       ../favicon.ico   1.23 KiB          [emitted]        
../images/phoenix.png   13.6 KiB          [emitted]        
        ../robots.txt  202 bytes          [emitted]        
               app.js   7.89 KiB     app  [emitted]        app
           app.js.map   9.41 KiB     app  [emitted] [dev]  app
Entrypoint app = ../css/app.css app.js ../css/app.css.map app.js.map
[0] multi ./js/app.js 28 bytes {app} [built]
[../deps/phoenix_html/priv/static/phoenix_html.js] 2.21 KiB {app} [built]
[./css/app.scss] 39 bytes {app} [built]
[./js/app.js] 490 bytes {app} [built]
    + 2 hidden modules
Child mini-css-extract-plugin node_modules/css-loader/dist/cjs.js!node_modules/sass-loader/dist/cjs.js!css/app.scss:
    Entrypoint mini-css-extract-plugin = *
    [./node_modules/css-loader/dist/cjs.js!./css/phoenix.css] 10.4 KiB {mini-css-extract-plugin} [built]
    [./node_modules/css-loader/dist/cjs.js!./node_modules/sass-loader/dist/cjs.js!./css/app.scss] 939 bytes {mini-css-extract-plugin} [built]
        + 1 hidden module
```

Pointing your browser at http://localhost:4000 you should see the default `Phoenix Framework` now.

```bash
[info] GET /
[debug] Processing with HelloWeb.PageController.index/2
  Parameters: %{}
  Pipelines: [:browser]
[info] Sent 200 in 6ms
```

### Prepare the project to run as a release

Elixir releases (available since 1.9) allow us to place all the application code into a self-contained directory. Lets configure the project so we can run the server as a release, initially locally.

Edit `config/prod.secret.exs` to replace `use Mix.Config` with `import Config` and uncomment a line that starts the server endpoint:

```bash
--- a/config/prod.secret.exs--- a/config/prod.secret.exs
+++ b/config/prod.secret.exs
@@ -2,7 +2,7 @@
 # from environment variables. You can also hardcode secrets,
 # although such is generally not recommended and you have to
 # remember to add this file to your .gitignore.
-use Mix.Config
+import Config
 
 secret_key_base =
   System.get_env("SECRET_KEY_BASE") ||
@@ -23,7 +23,7 @@ config :hello, HelloWeb.Endpoint,
 # If you are doing OTP releases, you need to instruct Phoenix
 # to start each relevant endpoint:
 #
-#     config :hello, HelloWeb.Endpoint, server: true
+config :hello, HelloWeb.Endpoint, server: true
 #
 # Then you can assemble a release by calling `mix release`.
 # See `mix help release` for more information.
```

**Rename `config/prod.secret.exs` to `config/releases.exs`:**

```bash
mv config/prod.secret.exs config/releases.exs
```

**Edit `config/prod.exs` to remove host and port data and loading of `prod/secret.exs`**

```bash
-- a/config/prod.exs
+++ b/config/prod.exs
@@ -10,7 +10,6 @@ use Mix.Config
 # which you should run after static files are built and
 # before starting your production server.
 config :hello, HelloWeb.Endpoint,
-  url: [host: "example.com", port: 80],
   cache_static_manifest: "priv/static/cache_manifest.json"
 
 # Do not print debug messages in production

@@ -49,7 +49,3 @@ config :logger, level: :info
 #       force_ssl: [hsts: true]
 #
 # Check `Plug.SSL` for all available options in `force_ssl`.
-
-# Finally import the config/prod.secret.exs which loads secrets
-# and configuration from environment variables.
-import_config "prod.secret.exs"
```

**Build a release:**

```bash
$ MIX_ENV=prod mix release
==> gettext
Compiling 1 file (.erl)
...
Generated hello app
* assembling hello-0.1.0 on MIX_ENV=prod
* skipping runtime configuration (config/runtime.exs not found)

Release created at _build/prod/rel/hello!

    # To start your system
    _build/prod/rel/hello/bin/hello start

Once the release is running:

    # To connect to it remotely
    _build/prod/rel/hello/bin/hello remote

    # To stop it gracefully (you may also send SIGINT/SIGTERM)
    _build/prod/rel/hello/bin/hello stop

To list all commands:

    _build/prod/rel/hello/bin/hello
```

**Start the release and check that the server is running fine on port 4000**

```bash
$ SECRET_KEY_BASE="5aalTEIX45QHhmz7M3uRnTa9S/Ianrt7KsmQCvM1IniOj0IDkcFN6NuJcXQoNoFV" _build/prod/rel/hello/bin/hello start
23:43:50.377 [info] Running HelloWeb.Endpoint with cowboy 2.8.0 at :::4000 (http)
23:43:50.377 [info] Access HelloWeb.Endpoint at http://localhost:4000
23:44:04.042 request_id=FkOXS8oRpmuajWsAAAAE [info] GET /
23:44:04.055 request_id=FkOXS8oRpmuajWsAAAAE [info] Sent 200 in 12ms
```

### Package the release as a docker image

**Create `Dockerfile` in the project's root directory**

We will use a two stage build to get a docker image with the minimum footprint. I am adding some niceties like `curl` but feel free to get rid of them if not needed.

```dockerfile
# ---- Build Stage ----
FROM elixir:1.10.4-alpine AS builder

LABEL app="build-hello"

ENV MIX_ENV=prod \
    LANG=C.UTF-8

COPY config ./config
COPY lib ./lib
COPY priv ./priv
COPY mix.exs .
COPY mix.lock .

RUN mix local.hex --force && \
    mix local.rebar --force && \
    mix deps.get && \
    mix deps.compile && \
    mix phx.digest && \
    mix release

# ---- Application Stage ----
FROM alpine:3
RUN apk add --no-cache --update busybox-extras bash openssl curl

ARG GIT_COMMIT
ARG VERSION

LABEL app="hello"
LABEL GIT_COMMIT=$GIT_COMMIT
LABEL VERSION=$VERSION

WORKDIR /app

COPY --from=builder _build .

CMD ["/app/prod/rel/hello/bin/hello", "start"]
```

**Create a docker image:**

```bash
$ docker build -t hello:0.1.0 .
Sending build context to Docker daemon  101.4MB
Step 1/19 : FROM elixir:1.10.4-alpine AS builder
 ---> 6470e7f49afc
...
Successfully tagged hello:0.1.0
```

**Verify that the image is built:**

```bash
$ docker image ls
REPOSITORY                            TAG                 IMAGE ID            CREATED             SIZE
hello                                 0.1.0               429d0fd537bb        2 seconds ago       25.6MB
```

**Now lets run the project as a docker image:**

```bash
$ docker run --publish 4000:4000 -e SECRET_KEY_BASE="5aalTEIX45QHhmz7M3uRnTa9S/Ianrt7KsmQCvM1IniOj0IDkcFN6NuJcXQoNoFV" hello:0.1.0
04:54:46.029 [info] Running HelloWeb.Endpoint with cowboy 2.8.0 at :::4000 (http)
04:54:46.029 [info] Access HelloWeb.Endpoint at http://localhost:4000
04:54:51.077 request_id=FkOX4nBX9opvIJMAAAAG [info] GET /
04:54:51.087 request_id=FkOX4nBX9opvIJMAAAAG [info] Sent 200 in 10ms
```

At this point we have a working docker image, all that is left is to make it run on a kubernetes cluster.

### Copy docker image to Google Cloud Container Registry

Usually docker images are tested and built (for instance, by Google Builder) automatically after every commit on a push to git repo, such as Github, but I'll do this manually here.

**Tag local docker image**

```bash
$ docker tag 429d0fd537bb gcr.io/your-gcloud-project-id/hello
```

**Push docker image to container registry:**

```bash
$ docker push gcr.io/your-gcloud-project-id/hello
The push refers to repository [gcr.io/your-gcloud-project-id/hello]
7345bbdb7ea1: Pushed 
04b5758a1ad3: Pushed 
3c6ab75eac4e: Pushed 
3e207b409db3: Layer already exists 
latest: digest: sha256:289c96a53573c7608e28919ef113a13308230bc36bfca4078fa6a7cf3afe428a size: 1158
```

At this point you can in fact deploy this image manually from the Google Console web interface, but I'll demo using helm from the command line. 

### Create a helm chart

I am using [helm v3](https://helm.sh/) for this post. Lets create the simplest helm chart possible.

**Create a directory to hold chart files**

```bash
$ mkdir -p charts/hello/templates
```

**Create `charts/hello/Chart.yaml`:**

```bash
apiVersion: v2
appVersion: 0.1.0
description: Hello web server Helm Chart
name: hello
version: 0.1.0
```

**Create a kubernets secret to hold value of secret key base:**

```bash
$ kubectl create secret generic hello-secret --from-literal=secret-key-base='5aalTEIX45QHhmz7M3uRnTa9S/Ianrt7KsmQCvM1IniOj0IDkcFN6NuJcXQoNoFV'
secret/hello-secret created

$ kubectl get secret hello-secret
NAME           TYPE     DATA   AGE
hello-secret   Opaque   1      31s
```

**Create `charts/hello/templates/_env.yaml`**

The string will be read from the kubernetes secret and an environment variable `SECRET_KEY_BASE` will be populated on the pod with its value. The same technique can be used for the rest of the configuration, e.g. database connection parameters and the like.

```bash
{{- define "env" -}}
- name: SECRET_KEY_BASE
  valueFrom:
    secretKeyRef:
      name: hello-secret
      key: secret-key-base
{{- end -}}
```

**Create deployment descriptor file, `charts/hello/templates/deployment.yaml`:**

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-app
  labels:
    app: hello
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
      - name: hello
        image: "gcr.io/your-project-id/hello:latest"
        ports:
        - containerPort: 4000
        env:
{{ include "env" . | indent 10 }}
```

**Create service descriptor file, `charts/templates/service.yaml`:**

```bash
apiVersion: v1
kind: Service
metadata:
  name: hello
spec:
  selector:
    app: hello
  ports:
    - protocol: TCP
      port: 80
      targetPort: 4000
```

**Test the chart (dry run, just to make sure the chart is valid):**

```bash
$ helm install --dry-run --debug hello charts/hello
install.go:172: [debug] Original chart version: ""
install.go:189: [debug] CHART PATH: /web/elixir/hello/charts/hello

NAME: hello
LAST DEPLOYED: Mon Nov  2 19:55:11 2020
NAMESPACE: default
STATUS: pending-install
REVISION: 1
TEST SUITE: None
USER-SUPPLIED VALUES:
{}

COMPUTED VALUES:
{}

HOOKS:
MANIFEST:
---
# Source: hello/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: hello
spec:
  selector:
    app: hello
  ports:
    - protocol: TCP
      port: 80
      targetPort: 4000
---
# Source: hello/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-app
  labels:
    app: hello
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
      - name: hello
        image: "gcr.io/your-project-id/hello:latest"
        env:
          - name: SECRET_KEY_BASE
            valueFrom:
              secretKeyRef:
                name: hello-secret
                key: secret-key-base
        ports:
          - containerPort: 4000
```

**Finally, install the chart on gcloud:**

```bash
$ helm install  hello charts/hello
NAME: hello
LAST DEPLOYED: Mon Nov  2 19:57:57 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

**See it in the list of running pods:**

```bash
$ kubectl get pods|grep hello
hello-app-5b6dd6fbf8-hxdgn                               1/1     Running            0          33s

$ kubectl logs -f hello-app-5b6dd6fbf8-hxdgn
00:58:02.586 [info] Running HelloWeb.Endpoint with cowboy 2.8.0 at :::4000 (http)
00:58:02.586 [info] Access HelloWeb.Endpoint at http://localhost:4000

$ kubectl get service|grep hello
hello                                         ClusterIP      10.91.246.47    <none>          80/TCP                                                           25s
```

**Access locally from the running pod:**

```bash
$ kubectl exec hello-app-5b6dd6fbf8-hxdgn -i -t -- curl http://localhost:4000
<!DOCTYPE html>
<html lang="en">
...
</html>
```

**Access from another pod in the cluster:**

```bash
$ kubectl exec another-pod -i -t -- curl http://hello
<!DOCTYPE html>
<html lang="en">
...
</html>
```

This is it, we deployed a Phoenix server on a kubernetes cluster. When you want to redeploy the server, use `helm upgrade` command instead. All of these steps are typically automated so you only need to run a deploy script after committing the changes, and the code is redeployed on the cluster. But it's good to know exactly what is happening underneath all the automation. 





