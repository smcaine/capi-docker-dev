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
kind create cluster --config kind-cluster-with-extramounts.yaml
```

Initialize the management cluster:

```
export CLUSTER_TOPOLOGY=true

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

export GITHUB_TOKEN=<add github token here>
export GITHUB_USER=<add github username here>

Deploy flux to the cluster:

```
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