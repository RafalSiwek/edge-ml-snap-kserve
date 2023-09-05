# ML-on-Edge delivery with Snap, Microk8s and KServe on Ubuntu Core

Deliver your ML artifact inference to edge devices using Snapcraft

# Prepare edge environment

## Edge OS

Run Ubuntu Core by

- [Booting on edge device](https://ubuntu.com/core/docs/getting-started)
- [Running Multipass VM](https://ubuntu.com/server/docs/virtualization-multipass)

Or run this on normal Ubuntu Server

## MicroK8s

Install Microk8s:

```bash
# The list of available releases can be found under: 
# https://snapcraft.io/microk8s
snap install microk8s --channel=1.28-strict/stable
sudo usermod -a -G snap_microk8s $USER
mkdir -p ~/.kube
sudo chown -f -R $USER ~/.kube
newgrp snap_microk8s
# Alias kubectl and helm:
sudo snap alias microk8s.kubectl kubectl
sudo snap alias microk8s.helm helm
# Microk8s is not started by default after installation. 
# To start MicroK8s run:
sudo microk8s start
# Enable extensions
microk8s enable metallb:10.64.140.43-10.64.140.49
microk8s enable registry
```

Result:

```bash
$ microk8s status
microk8s is running
high-availability: no
  datastore master nodes: 127.0.0.1:19001
  datastore standby nodes: none
addons:
  enabled:
    cert-manager         # (core) Cloud native certificate management
    dns                  # (core) CoreDNS
    ha-cluster           # (core) Configure high availability on the current node
    helm                 # (core) Helm - the package manager for Kubernetes
    helm3                # (core) Helm 3 - the package manager for Kubernetes
    hostpath-storage     # (core) Storage class; allocates storage from host directory
    metallb              # (core) Loadbalancer for your Kubernetes cluster
    registry             # (core) Private image registry exposed on localhost:32000
    storage              # (core) Alias to hostpath-storage add-on, deprecated
  disabled:
    cis-hardening        # (core) Apply CIS K8s hardening
    community            # (core) The community addons repository
    dashboard            # (core) The Kubernetes dashboard
    host-access          # (core) Allow Pods connecting to Host services smoothly
    ingress              # (core) Ingress controller for external access
    mayastor             # (core) OpenEBS MayaStor
    metrics-server       # (core) K8s Metrics Server for API access to service metrics
    minio                # (core) MinIO object storage
    observability        # (core) A lightweight observability stack for logs, traces and metrics
    prometheus           # (core) Prometheus operator for monitoring and logging
    rbac                 # (core) Role-Based Access Control for authorisation
    rook-ceph            # (core) Distributed Ceph storage using Rook
```

## KServe

```bash
# Download the quick_install shell script
curl -s "https://raw.githubusercontent.com/kserve/kserve/release-0.11/hack/quick_install.sh"
```

```bash
# Edit the file:
$ nano quick_install.sh

...
export ISTIO_VERSION=1.17.2
export KNATIVE_SERVING_VERSION=knative-v1.10.1
export KNATIVE_ISTIO_VERSION=knative-v1.10.0
export KSERVE_VERSION=v0.11.0
export CERT_MANAGER_VERSION=v1.3.0
# Add pointer to the snapped MicroK8s config file"
export KUBECONFIG=/var/snap/microk8s/current/credentials/client.config
```

```bash
# Run the KServe installation script
cat quick_install.sh | sudo bash
```

Check for the installation completeness

```bash
$ kubectl get pods --all-namespaces
NAMESPACE            NAME                                        READY   STATUS    RESTARTS   AGE
kube-system          calico-node-cxnms                           1/1     Running   0          7m16s
kube-system          coredns-864597b5fd-kfgq5                    1/1     Running   0          7m15s
kube-system          calico-kube-controllers-77bd7c5b-btfk4      1/1     Running   0          7m15s
kube-system          hostpath-provisioner-7df77bc496-7wdgq       1/1     Running   0          7m4s
metallb-system       controller-5c6b6c8447-jvjfg                 1/1     Running   0          7m2s
container-registry   registry-6c9fcc695f-5d6wp                   1/1     Running   0          7m4s
istio-system         istiod-57b55446f6-4vq54                     1/1     Running   0          6m25s
metallb-system       speaker-rq6wk                               1/1     Running   0          7m2s
istio-system         istio-ingressgateway-5b6899ddcc-w46xv       1/1     Running   0          6m15s
knative-serving      domain-mapping-5ffd4df948-mbw5b             1/1     Running   0          6m1s
cert-manager         cert-manager-cainjector-7c8bcfdd69-kmdkw    1/1     Running   0          5m57s
knative-serving      autoscaler-657cb48c96-sgxbt                 1/1     Running   0          6m2s
knative-serving      net-istio-webhook-55c8775bfd-xdttn          1/1     Running   0          6m
knative-serving      domainmapping-webhook-859df874cb-r6ml6      1/1     Running   0          6m1s
knative-serving      net-istio-controller-79dc5cdb78-hxxp5       1/1     Running   0          6m
knative-serving      controller-5649857ccc-q5ptg                 1/1     Running   0          6m1s
knative-serving      webhook-74b6f5cf75-mkj2h                    1/1     Running   0          6m1s
knative-serving      activator-7f86fb77f8-fjh8v                  1/1     Running   0          6m2s
cert-manager         cert-manager-5799666d46-9dvsz               1/1     Running   0          5m57s
cert-manager         cert-manager-webhook-6dd97d9768-szqgq       1/1     Running   0          5m57s
kserve               kserve-controller-manager-d754ccd4c-qllmw   2/2     Running   0          5m39s
```

## Insecure registry setup

Get the registry service details

```bash
$ kubectl get svc -n container-registry
NAME       TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
registry   NodePort   10.152.183.90   <none>        5000:32000/TCP   7h29m
$ kubectl describe svc registry -n container-registry
Name:                     registry
Namespace:                container-registry
Labels:                   app=registry
Annotations:              <none>
Selector:                 app=registry
Type:                     NodePort
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.152.183.90
IPs:                      10.152.183.90
Port:                     registry  5000/TCP
TargetPort:               5000/TCP
NodePort:                 registry  32000/TCP
Endpoints:                10.1.11.6:5000
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

Create a MicroK8s `cert.d` record file:

```bash
sudo mkdir -p /var/snap/microk8s/current/args/certs.d/<registry svc ClusterIP>:5000
sudo touch /var/snap/microk8s/current/args/certs.d/<registry svc ClusterIP>:5000/hosts.toml
```

Edit the `/var/snap/microk8s/current/args/certs.d/<registry svc ClusterIP>:5000/hosts.toml` file:

```bash
# /var/snap/microk8s/current/args/certs.d/<registry svc ClusterIP>:5000/hosts.toml
server = "http://<registry svc ClusterIP>:5000"

[host."<registry svc ClusterIP>:5000"]
capabilities = ["pull", "resolve"]
```

# On the ML model build machine

## Train a ML model

Run the [wine_rater_model_train](wine-rater/wine_rater_model_train.ipynb) notebook in the [ml directory](ml).

## Build docker KServe Inference images for `arm64` and `amd64` and export them to tarrbals

Make sure you have [docker installed and configured](configured)

```bash
# For amd64 target device
docker buildx build --output type=docker -t kserve-wine-rater-amd64 ml --platform linux/amd64
docker save kserve-wine-rater-amd64 -o images/kserve-wine-rater-amd64.tar

# For arm64 target device
docker buildx build --output type=docker -t kserve-wine-rater-arm64 ml --platform linux/arm64
docker save kserve-wine-rater-arm64 -o images/kserve-wine-rater-arm64.tar
```

## Test images locally

Depending on the hosts architecture run:

```bash
docker load --input images/kserve-wine-rater-<your-arch>.tar

docker run --rm -it -p 8080:8080 kserve-wine-rater-<your-arch>
```

Once the container is running, in a separate terminal request a example prediction:

```bash
curl -H "Content-Type: application/json" \
    -d '{"inputs": [{"name": "input1","shape": [1,11],"datatype": "FP32","data": [[5.6,0.31,0.37,1.4,0.074,12.0,96.0,0.9954,3.32,0.58,9.2]]}]}' \
  http://localhost:8080/v2/models/wine-rater/infer
```

An example output will be:

```
{"model_name":"wine-rater","model_version":null,"id":"5622b70a-6604-40ca-9cdc-38fc300bc93c","parameters":null,"outputs":[{"name":"output-0","shape":[1],"datatype":"FP64","parameters":null,"data":[5.288068083678879]}]}%
```

## Build Snaps

Run the build script

```bash
snapcraft
```

Result

```bash
$ snapcraft
Launching instance...
Executed: skip pull copy-script (already ran)
Executed: skip pull crane (already ran)
Executed: skip build copy-script (already ran)
Executed: skip build crane (already ran)
Executed: skip stage copy-script (already ran)
Executed: skip stage crane (already ran)
Executed: skip prime copy-script (already ran)
Executed: skip prime crane (already ran)
Executed parts lifecycle
Generated snap metadata
Created snap package wine-rater_0.0.1_amd64.snap
Launching instance...
Executed: skip pull copy-script (already ran)
Executed: skip pull crane (already ran)
Executed: skip build copy-script (already ran)
Executed: skip build crane (already ran)
Executed: skip stage copy-script (already ran)
Executed: skip stage crane (already ran)
Executed: skip prime copy-script (already ran)
Executed: skip prime crane (already ran)
Executed parts lifecycle
Generated snap metadata
Created snap package wine-rater_0.0.1_arm64.snap
```

Release Snaps

```bash
# Login to Snap Store
snapcraft login

# Register the Snap name as private
snapcraft register --private wine-rater

# Release the snaps
snapcraft upload --release=<your release> wine-rater_0.0.1_arm64.snap
snapcraft upload --release=<your release> wine-rater_0.0.1_amd64.snap
```

## Deploy ML model on edge

Download the ML InferenceService container Snap

```bash
# Login to Snap Store
sudo snap login

# Install the wine-rater InferenceService container Snap form the given channel
sudo snap install wine-rater --channel <required channel>
```

Check if the containers are available:

```bash
$ curl -X GET http://localhost:32000/v2/_catalog
{"repositories":["kserve-wine-rater"]}
$ curl -X GET http://localhost:32000/v2/kserve-wine-rater/tags/list
{"name":"kserve-wine-rater","tags":["<chosen version>"]}
```

Deploy the InferenceService

```bash
kubectl apply -f - <<EOF
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: wine-rater
spec:
  predictor:
    containers:
      - name: kserve-container
        image: <registry svc ClusterIP>:5000/kserve-wine-rater:<the version>
EOF
```

Result:

```bash
$ kubectl get inferenceservices
NAME         URL                                     READY   PREV   LATEST   PREVROLLEDOUTREVISION   LATESTREADYREVISION          AGE
wine-rater   http://wine-rater.default.example.com   True           100                              wine-rater-predictor-00001   47s
```

Test the model:

```bash
export MODEL_NAME=wine-rater
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
export SERVICE_HOSTNAME=$(kubectl get inferenceservice $MODEL_NAME -o jsonpath='{.status.url}' | cut -d "/" -f 3)

curl \
  -H "Host: ${SERVICE_HOSTNAME}" \
  -H "Content-Type: application/json" \
  -d '{"inputs": [{"name": "input1","shape": [1,11],"datatype": "FP32","data": [[5.6,0.31,0.37,1.4,0.074,12.0,96.0,0.9954,3.32,0.58,9.2]]}]}' \
  http://${INGRESS_HOST}:${INGRESS_PORT}/v2/models/${MODEL_NAME}/infer
```

Result:

```bash
$ curl \
  -H "Host: ${SERVICE_HOSTNAME}" \
  -H "Content-Type: application/json" \
  -d '{"inputs": [{"name": "input1","shape": [1,11],"datatype": "FP32","data": [[5.6,0.31,0.37,1.4,0.074,12.0,96.0,0.9954,3.32,0.58,9.2]]}]}' \
  http://${INGRESS_HOST}:${INGRESS_PORT}/v2/models/${MODEL_NAME}/infer
{"model_name":"wine-rater","model_version":null,"id":"f00ad461-e7f6-48b2-a707-f3a3a7bc604b","parameters":null,"outputs":[{"name":"output-0","shape":[1],"datatype":"FP64","parameters":null,"data":[5.288068083678879]}]}
```
