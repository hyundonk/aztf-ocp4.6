# OCP4.6 on Azure 



## Overview

This repo contains how to deploy infrastructure for OpenShift Cloud Platform 4.6 deployment in Azure. This is based on the documentation which is based on ARM template. 

[Installing a cluster on Azure using ARM templates - Installing on Azure | Installing | OpenShift Container Platform 4.6](https://docs.openshift.com/container-platform/4.6/installing/installing_azure/installing-azure-user-infra.html#installation-user-infra-generate_installing-azure-user-infra)

Running OCP on Azure means that you are responsible for managing all the infrastructure such as installing, patching, updating control nodes and application nodes by yourself. If you are looking for running managed OpenShift cluster on Azure instead, please refer https://azure.microsoft.com/en-us/services/openshift/. 



## Prerequisites

Follow the steps below in advance.

### 1) Creating a service principal

https://docs.openshift.com/container-platform/4.6/installing/installing_azure/installing-azure-user-infra.html#installation-azure-service-principal_installing-azure-user-infra



### 2) Obtaining the installation program

https://docs.openshift.com/container-platform/4.6/installing/installing_azure/installing-azure-user-infra.html#installation-obtaining-installer_installing-azure-user-infra

```bash
# Download install file at https://cloud.redhat.com/openshift/install/azure/user-provisioned
$ wegt https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-install-linux.tar.gz
$ tar xvzf openshift-install-linux.tar.gz
# Also download "pull-secret.txt"
```

### 3) Generating an SSH private key and adding it to the agent

https://docs.openshift.com/container-platform/4.6/installing/installing_azure/installing-azure-user-infra.html#ssh-agent-using_installing-azure-user-infra

Note) In the document above, "ed25519" algorithm is used. However, Azure doesn't support ED25519 for now. Use RSA key instead.  (https://docs.microsoft.com/en-us/azure/virtual-machines/linux/mac-create-ssh-keys#supported-ssh-key-formats)

```bash
$ ssh-keygen -t rsa -b 4096 -N '' -f <path>/<file_name> 
$ eval "$(ssh-agent -s)"
$ ssh-add <path>/<file_name> 

```

### 4) Creating the installation files for Azure

https://docs.openshift.com/container-platform/4.6/installing/installing_azure/installing-azure-user-infra.html#installation-initializing_installing-azure-user-infra

```bash
# Create install-config.yaml file
$ mkdir install
$ ./openshift-install create install-config --dir=install
```

Modify 'networking.machineNetwork.cidr' IP address range to your virtual address range.



### 5) Exporting common variables for ARM templates

https://docs.openshift.com/container-platform/4.6/installing/installing_azure/installing-azure-user-infra.html#installation-user-infra-exporting-common-variables-arm-templates_installing-azure-user-infra

```bash
export CLUSTER_NAME=<cluster_name>
export AZURE_REGION=<azure_region>
export SSH_KEY=<ssh_key>
export BASE_DOMAIN=<base_domain>
export BASE_DOMAIN_RESOURCE_GROUP=<base_domain_resource_group>

export KUBECONFIG=<installation_directory>/auth/kubeconfig 
```



### 6) Creating the Kubernetes manifest and Ignition config files

https://docs.openshift.com/container-platform/4.6/installing/installing_azure/installing-azure-user-infra.html#installation-user-infra-generate-k8s-manifest-ignition_installing-azure-user-infra

```bash
$./openshift-install create manifests --dir=./install

# remove Kubernetes manifest files that define the control plane machines
$ rm -f ./install/openshift/99_openshift-cluster-api_master-machines-*.yaml

# Remove the Kubernetes manifest files that define the worker machines:
$ rm -f ./install/openshift/99_openshift-cluster-api_worker-machineset-*.yaml

# Export the infrastructure ID.This is the value of the .status.infrastructureName attribute from the manifests/cluster-infrastructure-02-config.yml file.
$ export INFRA_ID=<infra_id> 

# Export the resource group by using the following command:
$ export RESOURCE_GROUP=<resource_group> 

# Modify networkResourceGroupName" value in 'install/manifests/cluster-infrastructure-02-config.yml' with existing value.
$ export NETWORK_RESOURCE_GROUP=<network_resource_group>
# create the Ignition configuration
$ ./openshift-install create ignition-configs --dir=./install
```



### 7) Creating the Kubernetes manifest and Ignition config files

https://docs.openshift.com/container-platform/4.6/installing/installing_azure/installing-azure-user-infra.html#installation-azure-create-resource-group-and-identity_installing-azure-user-infra

```bash
# Create the resource group. You would need to login first using "az login"
$ az group create --name ${RESOURCE_GROUP} --location ${AZURE_REGION}

# Create an Azure identity for the resource group
$ az identity create -g ${RESOURCE_GROUP} -n ${INFRA_ID}-identity

# Grant the Contributor role to the Azure identity
$ export PRINCIPAL_ID=`az identity show -g ${RESOURCE_GROUP} -n ${INFRA_ID}-identity --query principalId --out tsv`
$ export RESOURCE_GROUP_ID=`az group show -g ${RESOURCE_GROUP} --query id --out tsv`
$ az role assignment create --assignee "${PRINCIPAL_ID}" --role 'Contributor' --scope "${RESOURCE_GROUP_ID}"
```



### 8) Uploading the RHCOS cluster image and bootstrap Ignition config file

https://docs.openshift.com/container-platform/4.6/installing/installing_azure/installing-azure-user-infra.html#installation-azure-user-infra-uploading-rhcos_installing-azure-user-infra

```bash
# Create an Azure storage account to store the VHD cluster image
$ az storage account create -g ${RESOURCE_GROUP} --location ${AZURE_REGION} --name ${CLUSTER_NAME}sa --kind Storage --sku Standard_LRS

# Export the storage account key
$ export ACCOUNT_KEY=`az storage account keys list -g ${RESOURCE_GROUP} --account-name ${CLUSTER_NAME}sa --query "[0].value" -o tsv`

# Choose the RHCOS version to use and export the URL of its VHD to an environment variable
$ export VHD_URL=`curl -s https://raw.githubusercontent.com/openshift/installer/release-4.6/data/data/rhcos.json | jq -r .azure.url`

# Copy the chosen VHD to a blob
$ az storage container create --name vhd --account-name ${CLUSTER_NAME}sa --account-key ${ACCOUNT_KEY}
$ az storage blob copy start --account-name ${CLUSTER_NAME}sa --account-key ${ACCOUNT_KEY} --destination-blob "rhcos.vhd" --destination-container vhd --source-uri "${VHD_URL}"

# Check progress
status="unknown"
while [ "$status" != "success" ]
do
  status=`az storage blob show --container-name vhd --name "rhcos.vhd" --account-name ${CLUSTER_NAME}sa --account-key ${ACCOUNT_KEY} -o tsv --query properties.copy.status`
  echo $status
done

# Create a blob storage container to upload ignition files.
$ az storage container create --name files --account-name ${CLUSTER_NAME}sa --account-key ${ACCOUNT_KEY} --public-access blob

# upload the generated bootstrap.ign file:
$ az storage blob upload --account-name ${CLUSTER_NAME}sa --account-key ${ACCOUNT_KEY} -c "files" -f "./install/bootstrap.ign" -n "bootstrap.ign"
 

```



### 9) Create DNS zones

https://docs.openshift.com/container-platform/4.6/installing/installing_azure/installing-azure-user-infra.html#installation-azure-create-dns-zones_installing-azure-user-infra

```bash
# Create the new public DNS zone in the resource group exported in the BASE_DOMAIN_RESOURCE_GROUP
az network dns zone create -g ${BASE_DOMAIN_RESOURCE_GROUP} -n ${CLUSTER_NAME}.${BASE_DOMAIN}

# Create the private DNS zone in the same resource group as the rest of this deployment
$ az network private-dns zone create -g ${RESOURCE_GROUP} -n ${CLUSTER_NAME}.${BASE_DOMAIN}

```



### 10) Creating a Private DNS link to existing VNET

https://docs.openshift.com/container-platform/4.6/installing/installing_azure/installing-azure-user-infra.html#installation-azure-create-dns-zones_installing-azure-user-infra

```bash
$ export SUBSCRIPTION_ID=<subscription id>
$ export VNET_NAME=<virtual network name>

$ export VNET_RESOURCE_ID="/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${NETWORK_RESOURCE_GROUP}/providers/Microsoft.Network/virtualNetworks/${VNET_NAME}"

# You need to specify Virtual Network Resource ID instead of Virtual Network Name if it is different virtual network
$ az network private-dns link vnet create -g ${RESOURCE_GROUP} -z ${CLUSTER_NAME}.${BASE_DOMAIN} -n ${INFRA_ID}-network-link -v "${VNET_RESOURCE_ID}" -e false
```



### 11) Deploying the RHCOS cluster image for the Azure infrastructure

https://docs.openshift.com/container-platform/4.6/installing/installing_azure/installing-azure-user-infra.html#installation-azure-user-infra-deploying-rhcos_installing-azure-user-infra

```bash
$ export VHD_BLOB_URL=`az storage blob url --account-name ${CLUSTER_NAME}sa --account-key ${ACCOUNT_KEY} -c vhd -n "rhcos.vhd" -o tsv`

$ az deployment group create -g ${RESOURCE_GROUP} \
  --template-file "./install/02_storage.json" \
  --parameters baseName="${INFRA_ID}" \
  --parameters vhdBlobURL="${VHD_BLOB_URL}"
```



### 12) Creating networking and load balancing components in Azure

https://docs.openshift.com/container-platform/4.6/installing/installing_azure/installing-azure-user-infra.html#installation-creating-azure-dns_installing-azure-user-infra

```bash
# Set existing "MASTER_SUBNET_NAME" on which control plane nodes will be deployed.
$ export MASTER_SUBNET_NAME="subnet name"

$ az deployment group create -g ${RESOURCE_GROUP} \
  --template-file "./install/03_infra.json" \
  --parameters privateDNSZoneName="${CLUSTER_NAME}.${BASE_DOMAIN}" \
  --parameters baseName="${INFRA_ID}" \
  --parameters vnetName="${VNET_NAME}" \
  --parameters vnetResourceGroupName="${NETWORK_RESOURCE_GROUP}" \
  --parameters subnetName="${MASTER_SUBNET_NAME}"

# Create an api DNS record in the public zone for the API public load balancer.
$ export PUBLIC_IP=`az network public-ip list -g ${RESOURCE_GROUP} --query "[?name=='${INFRA_ID}-master-pip'] | [0].ipAddress" -o tsv`

$ az network dns record-set a add-record -g ${BASE_DOMAIN_RESOURCE_GROUP} -z ${CLUSTER_NAME}.${BASE_DOMAIN} -n api -a ${PUBLIC_IP} --ttl 60

$ az network dns record-set a add-record -g ${BASE_DOMAIN_RESOURCE_GROUP} -z ${BASE_DOMAIN} -n api.${CLUSTER_NAME} -a ${PUBLIC_IP} --ttl 60
```



### 13) Creating the bootstrap machine in Azure

https://docs.openshift.com/container-platform/4.6/installing/installing_azure/installing-azure-user-infra.html#installation-creating-azure-bootstrap_installing-azure-user-infra

```bash
$ export BOOTSTRAP_URL=`az storage blob url --account-name ${CLUSTER_NAME}sa --account-key ${ACCOUNT_KEY} -c "files" -n "bootstrap.ign" -o tsv`

$ export BOOTSTRAP_IGNITION=`jq -rcnM --arg v "3.1.0" --arg url ${BOOTSTRAP_URL} '{ignition:{version:$v,config:{replace:{source:$url}}}}' | base64 | tr -d '\n'`

$ az deployment group create -g ${RESOURCE_GROUP} \
  --template-file "./install/04_bootstrap.json" \
  --parameters bootstrapIgnition="${BOOTSTRAP_IGNITION}" \
  --parameters sshKeyData="${SSH_KEY}" \
  --parameters baseName="${INFRA_ID}" \
  --parameters vnetName="${VNET_NAME}" \
  --parameters vnetResourceGroupName="${NETWORK_RESOURCE_GROUP}" \
  --parameters subnetName="${MASTER_SUBNET_NAME}"
```



### 14) Creating the control plane machines in Azure

https://docs.openshift.com/container-platform/4.6/installing/installing_azure/installing-azure-user-infra.html#installation-creating-azure-bootstrap_installing-azure-user-infra

```bash
$ export MASTER_IGNITION=`cat ./install/master.ign | base64 | tr -d '\n'`

$ az deployment group create -g ${RESOURCE_GROUP} \
  --template-file "./install/05_masters.json" \
  --parameters masterIgnition="${MASTER_IGNITION}" \
  --parameters sshKeyData="${SSH_KEY}" \
  --parameters privateDNSZoneName="${CLUSTER_NAME}.${BASE_DOMAIN}" \
  --parameters baseName="${INFRA_ID}" \
  --parameters vnetName="${VNET_NAME}" \
  --parameters vnetResourceGroupName="${NETWORK_RESOURCE_GROUP}" \
  --parameters subnetName="${MASTER_SUBNET_NAME}"
 
# Wait until bootstraping ends
$ ./openshift-install wait-for bootstrap-complete --dir=./install --log-level info
```



### 15) Creating the node machines in Azure

https://docs.openshift.com/container-platform/4.6/installing/installing_azure/installing-azure-user-infra.html#installation-creating-azure-bootstrap_installing-azure-user-infra

```bash
$ export WORKER_IGNITION=`cat install/worker.ign | base64 | tr -d '\n'`
$ export NODE_SUBNET_NAME="<node subnet name>"

$ az deployment group create -g ${RESOURCE_GROUP} \
  --template-file "./install/06_workers.json" \
  --parameters workerIgnition="${WORKER_IGNITION}" \
  --parameters sshKeyData="${SSH_KEY}" \
  --parameters baseName="${INFRA_ID}" \
  --parameters vnetName="${VNET_NAME}" \
  --parameters vnetResourceGroupName="${NETWORK_RESOURCE_GROUP}" \
  --parameters subnetName="${NODE_SUBNET_NAME}"
  
  
```



## 











