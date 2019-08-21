# PostgreSQL in Kubernetes Cluster on Azure
Creating a auto-healing and load scaling Kubernetes cluster on Azure. Starts with creating the Azure cluster through to deploying the database.



## Creating the cluster
There are various pages of instruction for creating a cluster in Azure. This is one of them: https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough but these are the basic steps to create the cluster. This assumes that you have created a Azure accoun, have a login to the Azure Portal and have the Azure `az` client command line tools installed.

Start with creating the Resource Group. Adjust the region to suit your location:

`az group create --name k8-group --location australiasoutheast`

Create the Kubernetes cluster. This will spin up 3 nodes which is probably minimum of what you would like in a production environment:
```
az aks create \
    --resource-group k8-group \
    --name postgres-cluster \
    --node-count 3 \
    --enable-addons monitoring \
    --generate-ssh-keys
```

## Connect to the cluster
We used `kubectl` to manage this cluster. If you have already installed `kubectl`, you can ignore next command:

```az aks install-cli```

Connect `kubectl` to the cluster with:

```az aks get-credentials --resource-group k8-group --name postgres-cluster```

Verify the connection with `kubectl get nodes` which should show something like:
```
NAME                       STATUS   ROLES   AGE     VERSION
aks-nodepool1-31718369-0   Ready    agent   6m44s   v1.12.8
```

## Postgres YAML files
You will find all the yaml files you will need in this repo. There are better ways to store your credentials than in the YAML. We added them in here just for the sake of simplicity in this demo. These should go into environment variables and configured as part of your CI/CD pipelines:

* `postgres-configmap.yaml` - Configuration for Postgres database details which you can use to connect to the Pgsql database later.
* `postgres-storage.yaml` - This creates a Persistent shared storage on Azure which the pods in the Postgres cluster will use as a shared storage.
* `postgres-stateful.yaml` - Pulls it all together using Kubernetes Stateful sets. It has 3 replicas initially but will continue to scale the replicas up to a max of 10 replicas. This file also references the other yaml files.
* `postgres-service.yaml` - Creates a load balancer that works with the Kubernetes Master to distribute load to underlying pods. This creates a public ip which would not be ideal in a production environment as you would probably want to use a ClusterIP or NodeIp that is not exposed to the public... but for playing around with from your workstation, this is simpler.

## Deploy the YAML files
Assuming you have pulled this repo down locally, following are the steps for completing the installation:
```
kubectl create -f postgres-configmap.yaml
kubectl create -f postgres-storage.yaml
kubectl create -f postgres-stateful.yaml
kubectl create -f postgres-service.yaml
```

Watch the pod creation with:
```
kubectl get pods
```

Get the Service status and public ip with:
```
kubectl get svc postgres
```

Once the public ip is created, you should be able to connect to the database instance with command-line `psql` or something like the PgAdmin gui interface.

The guys at SeveralNines have a very helpful page with more detail on: https://severalnines.com/blog/using-kubernetes-deploy-postgresql

