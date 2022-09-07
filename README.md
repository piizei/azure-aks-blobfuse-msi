# azure-aks-blobfuse-msi
AKS CSI/Blob extension configuration with Managed Identity

## Set up AKS
```
az group create --name fusetest --location westeurope

az aks create -g fusetest -n fusetest --enable-managed-identity --node-count 1  --generate-ssh-keys

az feature register --name EnableBlobCSIDriver --namespace Microsoft.ContainerService

az provider register -n Microsoft.ContainerService

az aks update --enable-blob-driver -n fusetest -g fusetest
```

Check that the storage class for blob-driver is available with:
```kubectl get sc```

(make sure you have connection to right cluster i.e. (```az aks get-credentials --name fusetest --resource-group fusetest --overwrite-existing```)

you should see ```azureblob-fuse-premium``` in the response

## Create storage
```
az storage account create --name fusetest --resource-group fusetest --kind BlobStorage --access-tier hot
az storage container create --name fusecontainer --account-name fusetest --auth-mode login
```

## Setup RBAC for identities

Next find out the managed-identity of the kubelet-identity:

_Note: This query does not work if you have assigned multiple identities. Then use the one you want_
```
clusterid=$(az aks list --resource-group fusetest --query "[].{id:identityProfile.kubeletidentity.objectId}[0].id" --out tsv)
```
And the id of your blob-storage resource group
```
rgid=$(az group show  -n fusetest --query "id" --out tsv)
```
add kubelet sp-id as Storage Blob Data Contributor to resource-group of the blob-storage

_Note: Contributor does not work_
```
az role assignment create --assignee $clusterid --role 'Storage Blob Data Contributor' --scope $rgid
```


## Setup Kubernetes resources

Apply persistent-volume.yml
```
kubectl apply -f persistent-volume.yml
```
check that volume got created.
```kubectl get pv```

Apply persistent-volume-claim
```
kubectl apply -f persistent-volume-claim.yml
```
Check that it got created
```kubectl get pvc```

Install the test application (nginx)
```
kubectl apply -f deployment.yml
```
Check it works:
```kubectl get pods```

(wait until its done)

Thats it!

You should now see the mount with 
```kubectl exec -it nginx-blob -- df -h```


## Notes

If you need to add extra parameters to persistent-volume.yml volumeAttributes, you can find the docs here https://docs.microsoft.com/en-us/azure/aks/azure-csi-blob-storage-static?tabs=secret