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


* git clone this repo
* ```cd  marinesms/website```
* One time installation ```npm install```
* Start watch with ```npm start```
