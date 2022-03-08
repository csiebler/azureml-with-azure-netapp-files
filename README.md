# azureml-with-azure-netapp-files

A repo that shows how to use Azure Machine Learning with Azure NetApp Files. For storage access, NFSv3 is used, but it can easily be switched to NFSv4.1.

The code created below creates around $40/day of Azure expenses, so please use it wisely and delete your Azure NetApp Files Pool and Volume after testing.

# Instructions

Run the following instructions on Linux/macOS using the Azure CLI.

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

Edit [`train.yml`](train.yml) and adapt the NFS mount path to your IP address. Then kick off the AzureML job:

```console
az ml job create -f train.yml --web
```

## Results

Have a look at the `std_out.log` in the Azure Machine Learning experiment and validate that everything looks reasonable. If it worked, you should see the following `fio` results.

`fio` performing a 4k random reads against Azure NetApp Files:

```
fio-3.1
Starting 4 processes

4krandomreads: (groupid=0, jobs=4): err= 0: pid=40: Tue Mar  8 13:58:58 2022
   read: IOPS=52.8k, BW=206MiB/s (216MB/s)(4096MiB/19842msec)
    slat (usec): min=2, max=618, avg= 7.20, stdev= 4.20
    clat (usec): min=1476, max=29375, avg=9674.39, stdev=503.38
     lat (usec): min=1493, max=29383, avg=9682.65, stdev=503.29
    clat percentiles (usec):
     |  1.00th=[ 8717],  5.00th=[ 9110], 10.00th=[ 9241], 20.00th=[ 9372],
     | 30.00th=[ 9503], 40.00th=[ 9634], 50.00th=[ 9634], 60.00th=[ 9765],
     | 70.00th=[ 9765], 80.00th=[ 9896], 90.00th=[10159], 95.00th=[10290],
     | 99.00th=[10683], 99.50th=[10945], 99.90th=[13566], 99.95th=[17171],
     | 99.99th=[26346]
   bw (  KiB/s): min=50120, max=53568, per=25.00%, avg=52844.53, stdev=566.46, samples=156
   iops        : min=12530, max=13392, avg=13211.10, stdev=141.61, samples=156
  lat (msec)   : 2=0.01%, 4=0.01%, 10=84.11%, 20=15.86%, 50=0.04%
  cpu          : usr=4.93%, sys=25.15%, ctx=820086, majf=0, minf=536
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=0.1%, >=64=100.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.1%
     issued rwt: total=1048576,0,0, short=0,0,0, dropped=0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=128

Run status group 0 (all jobs):
   READ: bw=206MiB/s (216MB/s), 206MiB/s-206MiB/s (216MB/s-216MB/s), io=4096MiB (4295MB), run=19842-19842msec
4krandomreads: (g=0): rw=randread, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=128
```

`fio` performing a 4k random reads against the local disk of the Compute Cluster:

```
fio-3.1
Starting 4 processes
4krandomreads: Laying out IO file (1 file / 1024MiB)
4krandomreads: Laying out IO file (1 file / 1024MiB)
4krandomreads: Laying out IO file (1 file / 1024MiB)
4krandomreads: Laying out IO file (1 file / 1024MiB)

4krandomreads: (groupid=0, jobs=4): err= 0: pid=49: Tue Mar  8 13:59:55 2022
   read: IOPS=21.9k, BW=85.4MiB/s (89.5MB/s)(4096MiB/47974msec)
    slat (usec): min=94, max=189794, avg=177.11, stdev=1554.90
    clat (usec): min=4, max=548000, avg=23207.66, stdev=21237.69
     lat (usec): min=132, max=548181, avg=23385.73, stdev=21353.27
    clat percentiles (msec):
     |  1.00th=[   18],  5.00th=[   18], 10.00th=[   18], 20.00th=[   18],
     | 30.00th=[   19], 40.00th=[   19], 50.00th=[   20], 60.00th=[   20],
     | 70.00th=[   21], 80.00th=[   22], 90.00th=[   26], 95.00th=[   40],
     | 99.00th=[  108], 99.50th=[  178], 99.90th=[  342], 99.95th=[  380],
     | 99.99th=[  388]
   bw (  KiB/s): min= 1552, max=28128, per=25.12%, avg=21964.14, stdev=5547.17, samples=380
   iops        : min=  388, max= 7032, avg=5491.02, stdev=1386.78, samples=380
  lat (usec)   : 10=0.01%, 250=0.01%, 500=0.01%, 750=0.01%, 1000=0.01%
  lat (msec)   : 2=0.01%, 4=0.01%, 10=0.01%, 20=70.05%, 50=26.08%
  lat (msec)   : 100=2.67%, 250=1.00%, 500=0.18%, 750=0.01%
  cpu          : usr=1.67%, sys=9.30%, ctx=1048747, majf=0, minf=545
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=0.1%, >=64=100.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.1%
     issued rwt: total=1048576,0,0, short=0,0,0, dropped=0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=128

Run status group 0 (all jobs):
   READ: bw=85.4MiB/s (89.5MB/s), 85.4MiB/s-85.4MiB/s (89.5MB/s-89.5MB/s), io=4096MiB (4295MB), run=47974-47974msec
4krandomreads: (g=0): rw=randread, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=128
```

Once all of this is working, you can replace [`code/train.py`](code/train.py) with something more meaningful.