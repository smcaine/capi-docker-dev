# capi-docker-dev

Deploy a management cluster into kind and a workload cluster using the CAPD provider and flux.

## Steps

### Pre-requisites

- install clusterctl
- install kind
- install kubectl
- install docker
- install fluxctl

### Deploy Management Cluster

Run the following commands in yoru terminal:

```
cat > kind-cluster-with-extramounts.yaml <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraMounts:
    - hostPath: /var/run/docker.sock
      containerPath: /var/run/docker.sock
EOF
```

Apply this config:

```
kind create cluster --image kindest/node:v1.23.1 --config kind-cluster-with-extramounts.yaml
```

Initialize the management cluster:

```

# Initialize the management cluster
clusterctl init --infrastructure docker
```

Wait for it to complete (you should see the following message):

```
Your management cluster has been initialized successfully!

You can now create your first workload cluster by running the following:

  clusterctl generate cluster [name] --kubernetes-version [version] | kubectl apply -f -
```

The CAPI cluster resources have already been generated [here](./clusters/my-cluster/). Lets deploy flux to the cluster and let that manage the install.

To do this though, yoiu cna run this command:

```
clusterctl generate cluster capi-quickstart --flavor development \
  --kubernetes-version v1.23.1 \
  --control-plane-machine-count=1 \
  --worker-machine-count=1 \
  > ./clusters/my-cluster/capi-quickstart.yaml
```

Deploy flux to the cluster:

```
export GITHUB_TOKEN=<add github token here>

export GITHUB_USER=<add github username here>

flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=capi-docker-dev \
  --branch=main \
  --path=./clusters/my-cluster \
  --personal
```

Wait for the workload cluster deployment to complete:

```
kubectl get kubeadmcontrolplane -A --watch  
```

Once initialized, get the workload kubeconfig:

```
clusterctl get kubeconfig capi-quickstart > capi-quickstart.kubeconfig
```
When using using Docker Desktop on macOS or Docker Desktop (Docker Engine works fine) on Linux, you’ll need to take a few extra steps to get the kubeconfig for a workload cluster created with the Docker provider.
```
# Point the kubeconfig to the exposed port of the load balancer, rather than the inaccessible container IP.
sed -i -e "s/server:.*/server: https:\/\/$(docker port capi-quickstart-lb 6443/tcp | sed "s/0.0.0.0/127.0.0.1/")/g" ./capi-quickstart.kubeconfig
```
