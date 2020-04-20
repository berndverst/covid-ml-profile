# COVID-NET on Azure Kubernetes Service (AKS)

This project adapts the COVID-NET inferencing app example created by Weaveworks for FireKube for use with Azure AKS instead.

## Creating an AKS cluster
If you do not already have an AKS cluster, create one using the Azure CLI like so:

```
# Create a resource group
az group create --name covid-net-group --location westus

# Create a cluster with 2 nodes
az aks create --resource-group covid-net-group --name covid-k8s --node-count 2 --enable-addons monitoring --generate-ssh-keys --node-vm-size Standard_D4_v2

# Get credentials for kubectl and set current cluster config
az aks get-credentials -g covid-net-group -n covid-k8s
```

We will need a place to store XRay / CT images. We will be using Azure Blob Storage.

```
# Create a Storage Account for use with Minio
az storage account create --name myminiostorage --location westus --resource-group covid-net-group --sku Standard_LRS --enable-hierarchical-namespace true

# The covid-net app requires a bucket with the name fk-covid
az storage container create --name fk-covid --account-name myminiostorage

export AZ_STORAGE_ACCOUNT_NAME="myminiostorage"
export STORAGE_KEY=$(az storage account keys list --resource-group covid-net-group --account-name myminiostorage --query "[0].value" -o tsv)
```

## Create the `kubeflow` namespace

Technically we do not require Kubeflow to be installed. Creating the namespace is sufficient.

```
kubectl create namespace kubeflow
```

If you wish to install Kubeflow, see the Azure instructions at https://www.kubeflow.org/docs/azure/deploy/install-kubeflow/.


## Install COVID NET Profile for Azure

```bash
# Create Azure Blob Storage Secret
kubectl create secret generic azure-storage-secret --namespace kubeflow --from-literal=accountname=$AZ_STORAGE_ACCOUNT_NAME --from-literal=accountkey=$STORAGE_KEY

# Install Profile
git clone git@github.com:berndverst/covid-net-azure-profile.git
cd covid-net-azure-profile
kubectl apply -f covid-net-profile --namespace kubeflow
```


## Using the app

Upload images to classify to the `fk-covid` bucket of your storage account.
For convenience you can use the Minio service. The username and password are identical with the storage account name and secret.

The Minio file upload UI is public at the following IP address:

```bash
kubectl get svc minio-azure-service -n kubeflow -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

Once you have uploaded some images to the `fk-covid` folder, open the main UI for the app at the following IP address. Select an XRay image and click on **inference** to see the classification result (normal, pneumonia, COVID-19).

```bash
kubectl get svc fk-covid-net-frontend -n kubeflow -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

### References
This project was created in collaboration with Weaveworks and is based on the work by Chanwit Kaewkasi.

https://github.com/chanwit/covid-ml-profile

https://github.com/weaveworks/fk-covid
