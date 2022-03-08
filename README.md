# azureml-with-azure-netapp-files

A repo that shows how to use Azure Machine Learning with Azure NetApp Files.

# Instructions

## Install extensions

First, install the CLI extensions and make sure we can provision Azure NetApp Files:

```console
az extension remove -n azure-cli-ml
az extension add --name ml
az provider register --namespace Microsoft.NetApp --wait
```

## Configuration

Set variables on console session with our configuration:

```console
# Resource group name
rg='aml-anf-test'
location='westeurope'

# VNET details
vnet_name='vnet'
vnet_address_range='10.0.0.0/16'
vnet_aml_subnet='10.0.1.0/24'
vnet_anf_subnet='10.0.2.0/24'

# AML details
workspace_name='aml-anf'

# ANF details
anf_name=anf
pool_name=pool1
```

## Provisioning of services

Create a new Azure Machine Learning Workspace, VNET and Azure NetApp Files.

First, create resource group and set defaults:

```console
az group create -n $rg
az configure --defaults group=$rg workspace=$workspace_name location=$location
```

Provision VNET:

```console
az network vnet create -n $vnet_name --address-prefix $vnet_address_range
az network vnet subnet create --vnet-name $vnet_name -n aml --address-prefixes $vnet_aml_subnet
az network vnet subnet create --vnet-name $vnet_name -n anf --address-prefixes $vnet_anf_subnet --delegations "Microsoft.NetApp/volumes"
```

Provision Azure Machine Learning workspace:

```console
az ml workspace create --name $workspace_name
```

Provision Azure NetApp Files and create a single volume:

```console
az netappfiles account create --name $anf_name
az netappfiles pool create --account-name $anf_name --name $pool_name --size 4 --service-level premium
az netappfiles volume create --account-name $anf_name --pool-name $pool_name --name vol1 --service-level premium --usage-threshold 4096 --file-path "vol1" --vnet $vnet_name --subnet anf --protocol-types NFSv3 --allowed-clients $vnet_aml_subnet --rule-index 1 --unix-read-write true
```

## Resource creation in Azure Machine Learning

Create a new Azure Machine Learning Environment with NFS drivers installed:

```console
az ml environment create --file environment.yml
```

Alternatively, you can base this off an existing environment in Azure Machine Learning.

Next, create a Compute Cluster in the VNET (so that ANF can reach it):

```console
az ml compute create -n cpu-cluster --type amlcompute --min-instances 0 --max-instances 1 --size Standard_F16s_v2 --vnet-name $vnet_name --subnet aml --idle-time-before-scale-down 1800
```

## Run job reading data from Azure NetApp Files

Edit [`train.yml`](train.yml) and adapt the NFS mount path to your IP address. Then kick of AzureML job:

```console
az ml job create -f train.yml --web
```

Once that is working, you can replace [`code/train.py`](code/train.py) with something more meaningful.