# Cloud Native Guide

# Continous Integration & Delivery

Drone is our CI/CD tool, see [Issue #2](https://github.com/linxuyalun/devops/issues/2) to learn about why we choose it.

## Configure and run Drone server in single machine

The process of configuring and running Drone server in single machine can be seen on this [page](https://docs.drone.io/installation/github/single-machine/).

The main difference between the tutorial and our project is we use `docker-compose` to manage docker containers:

`docker-compose.yml`:

```yaml
version: '2'

services:
  drone-server:
    image: drone/drone:1.0.0
    ports:
      - 8081:80
    volumes:
      - ./:/data
      - /var/run/docker.sock:/var/run/docker.sock
    restart: always
    environment:
      - DRONE_SERVER_HOST=${DRONE_SERVER_HOST}
      - DRONE_SERVER_PROTO=${DRONE_SERVER_PROTO}
      - DRONE_TLS_AUTOCERT=false
      - DRONE_RUNNER_CAPACITY=3
      - DRONE_LOGS_PRETTY=true
      - DRONE_LOGS_COLOR=true
      - DRONE_USER_CREATE=username:linxuyalun,admin:true
      # GitHub Config
      - DRONE_GITHUB_SERVER=https://github.com
      - DRONE_GITHUB_CLIENT_ID=${DRONE_GITHUB_CLIENT_ID}
      - DRONE_GITHUB_CLIENT_SECRET=${DRONE_GITHUB_CLIENT_SECRET}
```

All sensitive information has been stored in `.env`.

To start the drone server, run:

```
docker-compose -f $FILE_NAME up -d
```

Ideally, when go to `www.your-drone-server.com`, Drone will first ask for authorization from GitHub, and then synchronous all repositories from the corresponding account:

![](img/1.png)

To continuously integrate, a `.drone.yml` is required in a repo, here's an example of `.drone.yml`:

```yaml
kind: pipline
name: demo

steps:
- name: node1
  image: node:11.12.0
  commands:
  - echo "this is testing"

- name: node2
  image: node:11.12.0
  commands:
  - sleep 10
  - echo "sleep 10"
```

Ideally, every time a member push or pull request, GitHub will activate a web hook created by Drone server. Drone server then integrates the project base on the information from `.drone.yml` and show the activity feed on Drone UI:![](img/2.jpg)

There are some details when configure and run Drone server. This [guide](https://discourse.drone.io/t/nothing-happens-when-i-push-code-no-builds-or-builds-stuck-in-pending/3424) troubleshoot the scenario where code is pushed and nothing happens in Drone. 

Now just play with it! To make full use of Drone, plugins are needed. Go through the [market page](http://plugins.drone.io/) and choose whatever you want  🥴 

## CI/CD in practice

Assume that we gonna develop a front end application, the following content gives you a guide of how to devops.

### Volume cache

When we write an application, it's really common that we need to install vast dependencies to support our application, which costs several minutes. In order to reduce this part costs, we can apply [**volume cache**](http://plugins.drone.io/drillster/drone-volume-cache/) in our CI/CD.

This plugin requires Volume configuration. This means your repository Trusted flag must be enabled. So we need to set our account as admin first — amend our `docker-compose.yml`:

```yaml
version: '2'

services:
  drone-server:
    image: drone/drone:1.0.0
    ports:
      - 8081:80
    volumes:
      - ./:/data
      - /var/run/docker.sock:/var/run/docker.sock
    restart: always
    environment:
      - DRONE_SERVER_HOST=${DRONE_SERVER_HOST}
      - DRONE_SERVER_PROTO=${DRONE_SERVER_PROTO}
      - DRONE_TLS_AUTOCERT=false
      - DRONE_RUNNER_CAPACITY=3
      - DRONE_LOGS_PRETTY=true
      - DRONE_LOGS_COLOR=true
      # Add admin account
      - DRONE_USER_CREATE=username:linxuyalun,admin:true
      # GitHub Config
      - DRONE_GITHUB_SERVER=https://github.com
      - DRONE_GITHUB_CLIENT_ID=${DRONE_GITHUB_CLIENT_ID}
      - DRONE_GITHUB_CLIENT_SECRET=${DRONE_GITHUB_CLIENT_SECRET}
```

See [full privilege](https://docs.drone.io/administration/user/admins/) of an admin here.

Restart the container:

```bash
> docker-compose -f docker-compose.yml up -d
Recreating devops_drone-server_1
```

Then we can go to a specific repo and setting it as trusted:

![](img/4.png)

Modify `.drone.yml`:

```yaml
kind: pipline
name: molecule

steps:
- name: restore-cache
  image: drillster/drone-volume-cache
  volumes:
  - name: cache
    path: /cache
  settings:
    restore: true
    mount:
    - ./node_modules
    - ./yarn-cache

- name: install
  image: node:11.13.0
  commands:
  - echo "Install dependencies 🧐🤪🤪"
  - yarn --version
  - yarn cache dir
  - yarn config set cache-folder /drone/src/yarn-cache
  - yarn install --pure-lockfile
  - echo "Install successfully 🏰🏰🏰"

- name: rebuild-cache
  image: drillster/drone-volume-cache
  volumes:
  - name: cache
    path: /cache
  settings:
    rebuild: true
    mount:
    - ./node_modules
    - ./yarn-cache

volumes:
  - name: cache
    host:
      path: /root/tmp/cache
```

To learn the meaning of the above params, see [here](http://plugins.drone.io/drillster/drone-volume-cache/).

### SCP

The [SCP plugin](http://plugins.drone.io/appleboy/drone-scp/) copy files and artifacts to target host machine via SSH. 

Because this is a front end application, there is no need to upload source files. We can first build the source code and then just upload these static files and other related configuration files.

```yaml
kind: pipline
name: molecule

steps:
- name: restore-cache
  image: drillster/drone-volume-cache
  volumes:
  - name: cache
    path: /cache
  settings:
    restore: true
    mount:
    - ./node_modules
    - ./yarn-cache

- name: install
  image: node:11.13.0
  commands:
  - echo "Install dependencies 🧐🤪🤪"
  - yarn --version
  - yarn cache dir
  - yarn config set cache-folder /drone/src/yarn-cache
  - yarn install --pure-lockfile
  - echo "Install successfully 🏰🏰🏰"

- name: build
  image: node:11.13.0
  commands:
  - echo "Build the application ⛑⛑⛑"
  - yarn cache dir
  - yarn build
  - echo "Build successfully 🗿🗿🗿"

- name: scp
  image: appleboy/drone-scp
  settings:
    host: example.host.com
    username: root
    password:
      from_secret: ssh_password
    target: /root/deploy/${DRONE_REPO}
    source:
    - build
    - Dockerfile
    - nginx.conf
    rm: true

- name: rebuild-cache
  image: drillster/drone-volume-cache
  volumes:
  - name: cache
    path: /cache
  settings:
    rebuild: true
    mount:
    - ./node_modules
    - ./yarn-cache

volumes:
  - name: cache
    host:
      path: /root/tmp/cache
```

Here in our `scp` pipeline, we also send `Dockerfile` and `nginx.conf`. The remote server will use these two files to launch a container of our application.

### SSH

Use the [SSH plugin](http://plugins.drone.io/appleboy/drone-ssh/) to execute commands on a remote server. The process is very similar to **SCP**, and the contents are also easy to understand.

```yaml
kind: pipline
name: molecule

steps:
- name: restore-cache
  image: drillster/drone-volume-cache
  volumes:
  - name: cache
    path: /cache
  settings:
    restore: true
    mount:
    - ./node_modules
    - ./yarn-cache

- name: install
  image: node:11.13.0
  commands:
  - echo "Install dependencies 🧐🤪🤪"
  - yarn --version
  - yarn cache dir
  - yarn config set cache-folder /drone/src/yarn-cache
  - yarn install --pure-lockfile
  - echo "Install successfully 🏰🏰🏰"

- name: build
  image: node:11.13.0
  commands:
  - echo "Build the application ⛑⛑⛑"
  - yarn cache dir
  - yarn build
  - echo "Build successfully 🗿🗿🗿"

- name: scp
  image: appleboy/drone-scp
  settings:
    host: example.host.com
    username: root
    password:
      from_secret: ssh_password
    rm: true
    target: /root/deploy/${DRONE_REPO}
    source:
    - build
    - Dockerfile
    - nginx.conf

- name: ssh
  image: appleboy/drone-ssh
  settings:
    host: example.host.com
    username: root
    password:
      from_secret: ssh_password
    script:
      - cd /root/deploy/${DRONE_REPO}
      - docker build -t xuylin/react-app .
      - docker rm -f react-app-demo
      - docker run -d --name react-app-demo -p 8088:80 xuylin/react-app

- name: rebuild-cache
  image: drillster/drone-volume-cache
  volumes:
  - name: cache
    path: /cache
  settings:
    rebuild: true
    mount:
    - ./node_modules
    - ./yarn-cache

volumes:
  - name: cache
    host:
      path: /root/tmp/cache
```

And here is the `Dockerfile` and `nginx.conf` in case of need:

`Dockerfile`:

```dockerfile
FROM nginx

COPY build/ /usr/share/nginx/html

COPY nginx.conf /etc/nginx/conf.d/default.conf
```

`nginx.conf`:

```
server {
    listen 80;
    server_name localhost;
    location / {
        root   /usr/share/nginx/html;
        index  index.html;
        try_files $uri /index.html;
    }
    location ~* /app.*\.(js|css|png|jpg)$
    {
        access_log off;
        expires    365d;
    }
    location ~* /app.*\.(?:manifest|appcache|html?|xml|json)$
    {
        expires    -1;
    }
}
```

If there is no error in your pipeline, you may see the following content:

![](img/5.png)

And from now on, you build an automatic devops pipeline successfully!

### Build the application in Docker (optional)

If you wouldn't like to build the application in Drone, you can build it just in the Dockerfile.

First, `Dockerfile` needs revision.

```dockerfile
# Stage 0, based on Node.js, to build and compile the frontend
FROM node:11.13.0 as build-stage

WORKDIR /app

COPY package.json /app/

RUN yarn install

COPY ./ /app/

RUN yarn build

# Stage 1 based on Nginx, to have only the compiled app, ready for production with Nginx
FROM nginx

COPY --from=build-stage /app/build/ /usr/share/nginx/html

COPY --from=build-stage /app/nginx.conf /etc/nginx/conf.d/default.conf
```

Modify our `scp ` in `.drone.yml`:

```yaml
kind: pipline
name: molecule

steps:
- name: restore-cache
  image: drillster/drone-volume-cache
  volumes:
  - name: cache
    path: /cache
  settings:
    restore: true
    mount:
    - ./node_modules
    - ./yarn-cache

- name: install
  image: node:11.13.0
  commands:
  - echo "Install dependencies 🧐🤪🤪"
  - yarn --version
  - yarn cache dir
  - yarn config set cache-folder /drone/src/yarn-cache
  - yarn install --pure-lockfile
  - echo "Install successfully 🏰🏰🏰"

- name: scp
  image: appleboy/drone-scp
  settings:
    host: example.host.com
    username: root
    password:
      from_secret: ssh_password
    rm: true
    target: /root/deploy/${DRONE_REPO}
    source:
    - src
    - public
    - package.json
    - Dockerfile
    - nginx.conf

- name: ssh
  image: appleboy/drone-ssh
  settings:
    host: example.host.com
    username: root
    password:
      from_secret: ssh_password
    script:
      - cd /root/deploy/${DRONE_REPO}
      - docker build -t xuylin/react-app .
      - docker rm -f react-app-demo
      - docker run -d --name react-app-demo -p 8088:80 xuylin/react-app

- name: rebuild-cache
  image: drillster/drone-volume-cache
  volumes:
  - name: cache
    path: /cache
  settings:
    rebuild: true
    mount:
    - ./node_modules
    - ./yarn-cache

volumes:
  - name: cache
    host:
      path: /root/tmp/cache
```

This is also an options of CI/CD, but it's really hard to say which one is better. For me, I prefer the former, which looks more general.

# Scheduling & Orchestration

As the only graduated scheduling and orchestration project in CNCF, k8s is no wonder the first choice. Learn basic knowledge of Kubernetes [here](https://kubernetes.io/docs/tutorials/).

## Start a Kubernetes single-node cluster

To start a Kubernetes single node cluster, several methods can be used.

### Minikube

Minikube implements a local Kubernetes cluster on macOS, Linux, and Windows. Its [goal](https://github.com/kubernetes/minikube/blob/master/docs/contributors/principles.md) is to enable fast local development and to support all Kubernetes features that fit. 

#### Installation

See [Install Minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/) (**Do notice the prerequisites of different OS**). You can also download minikube from its [GitHub repo](https://github.com/kubernetes/minikube/releases), which just released the latest version Minikube v1.0.0 recently.

#### Quick start

If your server is not in China, run the following command is totally enough:

```shell
minikube start
```

However, because of the firewall, it's really hard to download resource from `k8s.gcr.io`. So you must using Minikube with an HTTP Proxy.

```shell
# macOS and Linux
export HTTP_PROXY=http://<proxy hostname:port>
export HTTPS_PROXY=http://<proxy hostname:port>
export NO_PROXY=localhost,127.0.0.1,10.96.0.0/12,192.168.99.0/24,192.168.39.0/24

# Windows
set HTTP_PROXY=http://<proxy hostname:port>
set HTTPS_PROXY=http://<proxy hostname:port>
set NO_PROXY=localhost,127.0.0.1,10.96.0.0/12,192.168.99.1/24,192.168.39.0/24


minikube start --docker-env=HTTP_PROXY=$HTTP_PROXY \
			   --docker-env HTTPS_PROXY=$HTTPS_PROXY \
               --docker-env NO_PROXY=$NO_PROXY
```

#### Troubleshooting

1. Do set `HTTPS_PROXY=http://<proxy hostname:port>` instead of `HTTPS_PROXY=https://<proxy hostname:port>`, or you will get an error of `"tls: oversized record received with length 20527"`. See this [issue](https://github.com/kubernetes/kubernetes/issues/15839).
2. If you can't download `minikube-v1.0.0.iso`, you can download [it](https://storage.googleapis.com/minikube/iso/minikube-v1.0.0.iso) manually and place this file in `.minikube/cache/iso`. Run `minikube start` again.
3. Any problems of HTTP Proxy? This [doc](https://github.com/kubernetes/minikube/blob/master/docs/http_proxy.md) may help you.
4. If you are a Windows user and you want to start minikube with Hyper-V, following this [guide](https://medium.com/@JockDaRock/minikube-on-windows-10-with-hyper-v-6ef0f4dc158c).

### Docker-for-Desktop

Docker Desktop is the fastest and simplest way to get a Kubernetes cluster running on your desktop machine, while still giving you the freedom to choose Docker Swarm if you prefer. If you are a macOS user or Windows user, you can start a k8s single node cluster by Docker.

![](img/3.png)

Choose `Enable Kubernetes` and `Apply` it, a k8s single-node cluster will then be installed and started. Again, don't forget to set proxy!

### microk8s

#### Quick start

Microk8s is a single package of k8s that installs on Linux. It’s not elastic, but it is on rails. Use it for offline development, prototyping, testing, or use it on a VM as a small, cheap, reliable k8s for CI/CD.

Follow this [guide](https://tutorials.ubuntu.com/tutorial/install-a-local-kubernetes-with-microk8s#0) to install microk8s.

A proxy is needed, see [how to deploy behind a proxy](https://github.com/ubuntu/microk8s#deploy-behind-a-proxy).

Enable `dns` and `dashboard`:

`microk8s.enable dns dashboard`

If everything goes well, the following messages will be shown when run `microk8s.inspect`

```
Inspecting services
  Service snap.microk8s.daemon-containerd is running
  Service snap.microk8s.daemon-apiserver is running
  Service snap.microk8s.daemon-proxy is running
  Service snap.microk8s.daemon-kubelet is running
  Service snap.microk8s.daemon-scheduler is running
  Service snap.microk8s.daemon-controller-manager is running
  Service snap.microk8s.daemon-etcd is running
  Copy service arguments to the final report tarball
Inspecting AppArmor configuration
Gathering system info
  Copy network configuration to the final report tarball
  Copy processes list to the final report tarball
  Copy snap list to the final report tarball
  Inspect kubernetes cluster

Building the report tarball
  Report tarball is at /var/snap/microk8s/492/inspection-report-20190331_105134.tar.gz
```

To check the deployment progress of our addons, run `microk8s.kubectl get all --all-namespaces`:

```
NAMESPACE     NAME                                                  READY   STATUS    RESTARTS   AGE
kube-system   pod/heapster-v1.5.2-6b5d7b57f9-rgjzx                  4/4     Running   0          11h
kube-system   pod/kube-dns-6bfbdd666c-s6q65                         3/3     Running   2          11h
kube-system   pod/kubernetes-dashboard-6fd7f9c494-p6zrq             1/1     Running   0          13h
kube-system   pod/monitoring-influxdb-grafana-v4-78777c64c8-rjrcl   2/2     Running   0          13h

NAMESPACE     NAME                           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
default       service/kubernetes             ClusterIP   10.152.183.1    <none>        443/TCP             13h
kube-system   service/heapster               ClusterIP   10.152.183.30   <none>        80/TCP              13h
kube-system   service/kube-dns               ClusterIP   10.152.183.10   <none>        53/UDP,53/TCP       13h
kube-system   service/kubernetes-dashboard   ClusterIP   10.152.183.67   <none>        443/TCP             13h
kube-system   service/monitoring-grafana     ClusterIP   10.152.183.35   <none>        80/TCP              13h
kube-system   service/monitoring-influxdb    ClusterIP   10.152.183.47   <none>        8083/TCP,8086/TCP   13h

NAMESPACE     NAME                                             READY   UP-TO-DATE   AVAILABLE   AGE
kube-system   deployment.apps/heapster-v1.5.2                  1/1     1            1           13h
kube-system   deployment.apps/kube-dns                         1/1     1            1           13h
kube-system   deployment.apps/kubernetes-dashboard             1/1     1            1           13h
kube-system   deployment.apps/monitoring-influxdb-grafana-v4   1/1     1            1           13h

NAMESPACE     NAME                                                        DESIRED   CURRENT   READY   AGE
kube-system   replicaset.apps/heapster-v1.5.2-5c5498f57c                  0         0         0       13h
kube-system   replicaset.apps/heapster-v1.5.2-6b5d7b57f9                  1         1         1       11h
kube-system   replicaset.apps/heapster-v1.5.2-89b48dff                    0         0         0       11h
kube-system   replicaset.apps/kube-dns-6bfbdd666c                         1         1         1       13h
kube-system   replicaset.apps/kubernetes-dashboard-6fd7f9c494             1         1         1       13h
kube-system   replicaset.apps/monitoring-influxdb-grafana-v4-78777c64c8   1         1         1       13h
```

Make sure that all pods are in the "Running" state.

See cluster information with `microk8s.kubectl cluster-info`:

```
Kubernetes master is running at https://127.0.0.1:16443
Heapster is running at https://127.0.0.1:16443/api/v1/namespaces/kube-system/services/heapster/proxy
KubeDNS is running at https://127.0.0.1:16443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Grafana is running at https://127.0.0.1:16443/api/v1/namespaces/kube-system/services/monitoring-grafana/proxy
InfluxDB is running at https://127.0.0.1:16443/api/v1/namespaces/kube-system/services/monitoring-influxdb:http/proxy
```

#### Troubleshooting

1. If you deploy microk8s on a remote machine, username and password are required when visit the machine. Right now this username/password are random strings created at MicroK8s install time. You should be able to add more uses in `/var/snap/microk8s/current/credentials/basic_auth.csv`. You can read more [here](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#static-password-file).
2. If your pod's status is not "RUNNING", for example, but "ImagePullBackOff". You can run `microk8s.kubectl describe pod <pod-id>` to find out what's happening. More information about [how to debug “ImagePullBackOff”?](https://stackoverflow.com/questions/34848422/how-to-debug-imagepullbackoff)

## Kubernetes demo

You can learn about `kubectl` document which was just rewritten in k8s 1.14 [here](https://kubectl.docs.kubernetes.io/).

All files in the following demos can be found [here](../k8s-demo).

### Demo 01 - Secrets

**Aim**: Learn about how to mount secret file into containers.

`customization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml

namespace: default

secretGenerator:
- name: db-secrets
  files:
  - "secret/username"
  - "secret/password"
  type: Opaque
```

`deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: helloworld-deployment
  labels:
    app: helloworld
spec:
  replicas: 3
  selector:
    matchLabels:
      app: helloworld
  template:
    metadata:
      labels:
        app: helloworld
    spec:
      containers:
      - name: k8s-demo
        image: wardviaene/k8s-demo
        ports:
        - name: nodejs-port
          containerPort: 3000
        volumeMounts:
        - name: cred-volume
          mountPath: /etc/creds
          readOnly: true
      volumes:
      - name: cred-volume
        secret:
          secretName: db-secrets
```

Run Apply on directories containing `kustomization.yaml` files using `-k`

```
kubectl apply -k .
```

The following messages shows that you created the pods successfully

```
secret/db-secrets-k87mg5mgmk created
deployment.apps/helloworld-deployment created
```

Get pods:

```
NAME                                     READY   STATUS    RESTARTS   AGE
helloworld-deployment-586989ffdd-kvqs7   1/1     Running   0          22s
helloworld-deployment-586989ffdd-l6kr9   1/1     Running   0          22s
helloworld-deployment-586989ffdd-wcpwc   1/1     Running   0          22s
```

Then describe one pod:

```bash
kubectl describe pod helloworld-deployment-586989ffdd-kvqs7
```

```
# Output
Name:               helloworld-deployment-586989ffdd-kvqs7
Namespace:          default
Priority:           0
PriorityClassName:  <none>
Node:               alan/172.18.51.81
Start Time:         Fri, 03 May 2019 11:00:11 +0800
Labels:             app=helloworld
                    pod-template-hash=586989ffdd
Annotations:        <none>
Status:             Running
IP:                 10.1.1.30
Controlled By:      ReplicaSet/helloworld-deployment-586989ffdd
Containers:
  k8s-demo:
    Container ID:   containerd://beecf1fe98abbec7dd22ca733269e17099fca6b55a3bebf32e2b6d78c199b7d4
    Image:          wardviaene/k8s-demo
    Image ID:       docker.io/wardviaene/k8s-demo@sha256:2c050f462f5d0b3a6430e7869bcdfe6ac48a447a89da79a56d0ef61460c7ab9e
    Port:           3000/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Fri, 03 May 2019 11:00:18 +0800
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /etc/creds from cred-volume (ro)
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-b5hpj (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
  Volumes:
  cred-volume:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  db-secrets-k87mg5mgmk
    Optional:    false
  default-token-b5hpj:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-b5hpj
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  39s   default-scheduler  Successfully assigned default/helloworld-deployment-586989ffdd-kvqs7 to alan
  Normal  Pulling    38s   kubelet, alan      Pulling image "wardviaene/k8s-demo"
  Normal  Pulled     32s   kubelet, alan      Successfully pulled image "wardviaene/k8s-demo"
  Normal  Created    32s   kubelet, alan      Created container k8s-demo
  Normal  Started    32s   kubelet, alan      Started container k8s-demo
```

You can see that we successfully mount the secrets files into containers:

```
Mounts:
  /etc/creds from cred-volume (ro)
  /var/run/secrets/kubernetes.io/serviceaccount from default-token-b5hpj (ro)
```

One important thing is Kubernetes also uses the Secretes in the volumes to share the Kubernetes credentials:

```
/var/run/secrets/kubernetes.io/serviceaccount from default-token-b5hpj (ro)
```

This is a volume that is already mounted and is done by Kubernetes itself to share the secrets within the pod so the pod can access the API.

So let's start a shell:

```
kubectl exec -it helloworld-deployment-586989ffdd-kvqs7 -- /bin/bash
root@helloworld-deployment-586989ffdd-kvqs7:/app# cd ../etc/creds
root@helloworld-deployment-586989ffdd-kvqs7:/etc/creds# ls
password  username
root@helloworld-deployment-586989ffdd-kvqs7:/etc/creds# cat password
password
root@helloworld-deployment-586989ffdd-kvqs7:/etc/creds# cat username
root
```

### Demo 02 - Running Apllication Using the Secrets

**Aim**: Set up an apllication(e.g. WordPress) using the secrets.

In this demo, it does not involve stateful containers yet, that means that whenever you put data in this WordPress, it's not going to be persistent. 

`kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml

namespace: default

secretGenerator:
- name: wordpress-secrets
  files:
  - "secret/db-password"
  type: Opaque
```

`deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress-deployment
  labels:
    app: wordpress
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
      - name: wordpress
        image: wordpress:4-php7.0
        ports:
        - name: http-port
          containerPort: 80
        env:
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: wordpress-secrets
              key: db-password
        - name: WORDPRESS_DB_HOST
          value: 127.0.0.1
      - name: mysql
        image: mysql:5.7
        ports:
        - name: mysql-port
          containerPort: 3306
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: wordpress-secrets
              key: db-password
```

We'd like to access the WordPress, thus we set up a Service and expose the port (e.g. 31000).

`service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name:  wordpress-service
spec:
  selector:
    app: wordpress
  type: NodePort
  ports:
  - name: wordpress-service
    port: 31000
    nodePort: 31000
    targetPort: http-port
    protocol: TCP
```

Apply the above files:

```bash
kubectl apply -k .
```

And we set up the WordPress successfully!

```
> kubectl get pods
NAME                                    READY   STATUS    RESTARTS   AGE
wordpress-deployment-5d8449fcdb-7s48d   2/2     Running   0          15m
```

Then we can play with it - choose a language, register, write a post and so on. However, do remember these data is all not persistent because the containers we created are cattles.

### Demo 03 - Service Discovery

**Aim**: Connect different services by service discovery

`kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- database-deployment.yaml
- database-service.yaml
- helloworld-deployment.yaml
- helloworld-service.yaml

namespace: default

secretGenerator:
- name: helloworld-secrets
  files:
  - "secret/username"
  - "secret/password"
  - "secret/rootPassword"
  - "secret/database"
  type: Opaque 
```

`database-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database-deployment
  labels:
    app: database
spec:
  replicas: 1
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
      - name: mysql
        image: mysql:5.7
        ports:
        - name: mysql-port
          containerPort: 3306
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: helloworld-secrets
              key: rootPassword
        - name: MYSQL_USER
          valueFrom:
            secretKeyRef:
              name: helloworld-secrets
              key: username
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: helloworld-secrets
              key: password
        - name: MYSQL_DATABASE
          valueFrom:
            secretKeyRef:
              name: helloworld-secrets
              key: database
```

`database-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: database-service # service name
spec:
  selector:
    app: database
  type: NodePort
  ports:
  - name: database-service
    port: 3306
    targetPort: mysql-port
    protocol: TCP
```

`helloworld-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: helloworld-deployment
  labels:
    app: helloworld
spec:
  replicas: 3
  selector:
    matchLabels:
      app: helloworld
  template:
    metadata:
      labels:
        app: helloworld
    spec:
      containers:
      - name: k8s-demo
        image: wardviaene/k8s-demo
        command: ["node", "index-db.js"]
        ports:
        - name: nodejs-port
          containerPort: 3000
        env:
        - name: MYSQL_HOST
          value: database-service		# service descovery, thanks to DNS
        - name: MYSQL_USER
          value: root
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: helloworld-secrets
              key: rootPassword
        - name: MYSQL_DATABASE
          valueFrom:
            secretKeyRef:
              name: helloworld-secrets
              key: database
```

`helloworld-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name:  helloword-service
spec:
  selector:
    app: helloword
  type: NodePort
  ports:
  - name: helloword-service
    port: 3000
    nodePort: 31001
    protocol: TCP
```

Apply the above files:

```bash
kubectl apply -k .
```

To make sure that the  application has connected to database, we can check the logs:

```
kubectl logs helloworld-deployment-6bfd7b6df6-sz2r4
```

Here's the output, indicating connecting to database successfully:

```
Example app listening at http://:::3000
Connection to db established
```

### Demo 04 - Volumes

**Aim**: Say hello to volumes

We have already see `Volumes` in demo 1, now let's learn more about it.

Kubernetes [supports several types of Volumes](https://kubernetes.io/docs/concepts/storage/volumes/#types-of-volumes). In our demo, we will use `hostPath` volume, which mounts a file or directory from the host node’s filesystem into your Pod.   

To make the demo easy, we won't apply `kustomization.yaml` this time, one file is pretty enough.

`volume.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello-volume
spec:
  containers:
  - image: busybox
    name: busybox1
    command:
    - /bin/sh
    - -c
    - "while true; do echo hello; sleep 10; done"
    volumeMounts:
    - name: hello-volume
      mountPath: /test-pd
  - image: busybox
    name: busybox2
    command:
    - /bin/sh
    - -c
    - "while true; do echo hello; sleep 10; done"
    volumeMounts:
    - name: hello-volume
      mountPath: /test-pd
  volumes:
  - name: hello-volume
    hostPath:
      path: /root/data	# need to be created in advance
```

```
kubectl apply -f volume.yaml
```

Now we can see two pods have been set up:

```
NAME                        READY   STATUS    RESTARTS   AGE
hello-volume                2/2     Running   0          17m
```

Note that `/root/data` on your local machine and `/test-pd` in your containers share the data:

```bash
> touch test
> echo hello >> test
> cat test
	hello

> kubectl exec -it hello-volume sh
  Defaulting container name to busybox1.
  Use 'kubectl describe pod/hello-volume -n default' to see all of the containers in this pod.
  / # cd /test-pd/
  /test-pd # more test
  hello
  /test-pd # echo goodbye >> test
  /test-pd # exit

> cat test
	hello
	goodbye
```

