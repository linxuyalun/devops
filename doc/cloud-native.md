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
      # GitHub Config
      - DRONE_GITHUB_SERVER=https://github.com
      - DRONE_GITHUB_CLIENT_ID=${DRONE_GITHUB_CLIENT_ID}
      - DRONE_GITHUB_CLIENT_SECRET=${DRONE_GITHUB_CLIENT_SECRET}
      - DRONE_LOGS_PRETTY=true
      - DRONE_LOGS_COLOR=true
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



