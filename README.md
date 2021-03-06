# PostgreSQL in Kubernetes Cluster on Azure
Basic steps for creating a Kubernetes cluster on Azure and deploying latest Postgres database configuring for stateful, persistent storage to ensure data is not lost if the single Postgres primary pod falls over.

## Creating the cluster
There are various pages of instruction for creating a cluster in Azure. This is one of them: https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough but these are the basic steps to create the cluster. This assumes that you have:
* Created a Azure account
* Have a login to the Azure Portal
* Have the Azure `az` client command line tools installed
* Have installed psql client tool to access the database.

Start with creating the Resource Group. Adjust the region to suit your location:

`az group create --name k8-group --location australiasoutheast`

Create the Kubernetes cluster. This will spin up 2 nodes:
```
az aks create \
    --resource-group k8-group \
    --name postgres-cluster \
    --node-count 2 \
    --enable-addons monitoring \
    --generate-ssh-keys
```

## Connect to the cluster
I used `kubectl` to manage this cluster. If you have already installed `kubectl`, you can ignore next command:

```az aks install-cli```

Connect `kubectl` to the cluster with:

```az aks get-credentials --resource-group k8-group --name postgres-cluster```

Verify the connection with `kubectl get nodes` which should show something like:
```
NAME                       STATUS   ROLES   AGE     VERSION
aks-nodepool1-31718369-0   Ready    agent   6m44s   v1.12.8
```

## Postgres YAML files
You will find all the yaml files you will need in this repo. . We added them in here just for the sake of simplicity in this demo:

* `psql-config.yaml` - Configuration for Postgres database details which you can use to connect to the Pgsql database.
* `psql-secret.yaml` -  Stores and manage sensitive information. Password in there have been base64 encoded.
* `psql-stateful.yaml` - Pulls it all together using Kubernetes Stateful sets.
* `psql-service.yaml` - Creates the access service with a NodePort access point rather than a external ip (LoadBalancer type). More on how to access this further down.

## Deploy the YAML files
Assuming you have pulled this repo down locally, following are the steps for completing the installation. You can reapply these any number of times to update settings in the YAML files:
```
kubectl apply -f ./psql-config.yaml 
kubectl apply -f ./psql-secret.yaml 
kubectl apply -f ./psql-stateful.yaml 
kubectl apply -f ./psql-service.yaml 
```

This will create the configuration/secre YAML's, start pulling down the Postgres image and create the persistent storage volume. Use get and describe to watch the pod creation:
```
kubectl get pods

NAME          READY   STATUS    RESTARTS   AGE
pg-single-0   1/1     Running   0          42m
```

The Statefulset will propably takes the longest to create depending on the size of the volume defined in psql-stateful.yaml. Watch progress with:
```
kubectl describe statefulset
```

Get the Service status and ClusterIP with:
```
kubectl get svc postgres

NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
pg-single    NodePort    10.0.102.104   <none>        5432:31295/TCP   40m
```

To connect to this instance with something like `psql`, you need to do a port forward which will then enable you to connect to the postgres instances from your local dev machine. Use the NAME in the `kubectl get pods` to create a port-forward. A reasonably secure way to connect to the remote database instance:
```
kubectl port-forward pod/pg-single-0 5432:5432
```

At this stage you should be able to connect to the db as if it was on your localhost:
```
psql -h localhost -U postgres
```

## Resize the Volume Claim
As Kubernetes does not allow changes to volume size of the volumeClaimTemplate after a statefulset has first been applied, you need to edit the pvc after it has been created. Use this command to get the name of the Persistent Volume Claim:

```
kubectl get pvc
```
.. and then edit the pvc:

```
kubectl edit pvc pg-single-claim
```
... and increase the storage size under spec.resources.requests. You may need to kill the Postgres pod to reclaim the new storage size.

## Summary
You can also enable a external IP to the above service which exposes the db instance to the then internet. Less ideal security wise but if you scenario demands it, just change the Type in psql-service.yaml from NodePort to LoadBalancer.

Statefulset has a single pod only as postgres can only have single primary writing to its datastore at any one time. Might feel like that makes it pointless to put this in Kubernetes cluster but if a pod crashes and because the storage in this installation is persistent, the pod will recover very quickly and connect to the existing storage volume. If the underlying node is unavailable, it will create a new pod in second node connected to the existing storage.
