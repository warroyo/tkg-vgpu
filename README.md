# VGPU with TKGm

This doc outlines a process for using NVIDIA vGPU capabilties with TKGm(TKG w/standalone management cluster). **This process is not officially supported by VMware**. There are two ways of using VGPU capabilties, multi instance and time slicing. This doc will cover both.


## Install ESX Drivers for NVAIE

Follow the NVIDIA docs to install the NVAIE drivers. There is nothing unique to TKG about this process.

Ensure that the version you install aligns with the version of the operator you will be using in the steps further down.

1. Install the VIBs - [docs here](https://docs.nvidia.com/ai-enterprise/latest/quick-start-guide/index.html#installing-grid-vgpu-manager-vmware-vsphere)
2. Disable ECC if your GPU does not support it - [docs here](https://docs.nvidia.com/ai-enterprise/latest/quick-start-guide/index.html#disabling-enabling-ecc-memory)
3. If your GPU supports MIG and you want to use it, it needs to be enabled at the esx level. - [docs here](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/#enable-mig-mode)


**NOTES:**
* If you have previsously had this gpu set to passthrough be sure to disable that
* `nvidia-smi -q` is very helpful to get info about ECC


## Configure the graphics settings in vCenter

In this case we do not want to enable passthrough we just want to enable the "shared direct" settings

1. navigate to the different tabs and set the gaprhics setting to "shared direct" -  [docs here](https://docs.nvidia.com/ai-enterprise/latest/quick-start-guide/index.html#changing-default-gpu-mode-vmware-vsphere)


## Setup vGPU profile based TKG templates

Becuase CAPV does not support vGPU today we need to work around this by creating some vSphere templates that already have the gpu profiles added as pci devices. CAPv support passthrough for GPU by adding a pci device, however the way that adding a vGPU works in vCenter has a slightly different api so this can't be done thropugh CAPv today.


**If you are running TKG 1.x you will first need to create an EFI ubuntu image [using BYOI](https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/1.6/vmware-tanzu-kubernetes-grid-16/GUID-build-images-linux.html). In TKG 2.x EFI templates are shipped with the product so we can just use one of those.**

### Get the list of available profiles

Get a list of the profiles that are available for your GPU.These will differ based on your GPU model. 

1. Convert the template to a VM and place it on the host with a GPU. 
2. Edit the VM settings and add a PCI device.
3. The PCI device section should give you an option for `NVIDIA GRID vGPU` choose that and click the dropdown

![](images/2023-05-12-13-23-36.png)

4. Take screenshot of this list for the next step. 


**NOTES:**

* You can find out what the different profiles mean through the nvidia docs. For the example above with a T4 the different types of vgpu are [here](https://docs.nvidia.com/grid/10.0/grid-vgpu-user-guide/index.html#vgpu-types-tesla-t4). Since this use case is with TKG for traning workloads we can see on the doc the best profiles for that are the `c` profiles.


### Create vGPU profile templates

This step is how we work around the limitation of CAPv today mentioned above. Since this is for AI workloads we will onyl create templates for the profiles that meet that type. In the case of this example it will be 16c, 8c, and 4c. If you are using mig the profiles will look different you can see [this doc](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/#a100-profiles) for details. 

For each relevant profile take the following steps.

1. Copy the EFI based TKG template to a new template and rename it to something meaninful with the profile name in it. ex. `ubuntu-2004-efi-kube-v1.23.8-t4-4c`
2. Convert the newly created template to a VM and place it on a host that has the GPU available. 
3. Edit the settings on the VM and add the PCI device with the GPU profile that matches the name you gave the template and click ok to exit the settings. 
![](images/2023-05-12-14-49-10.png)
4. convert the VM back to a template.



## Create a nodepool with GPU

This is a slightly different process based on which version of TKg you are on and whether or not you are adding a nodepool to an existing cluster. The sections below are broken out by scenario.

### TKG 1.6 - new cluster

This approach makes use of an overlay to be able to add nodepools mainly due to some extra config that is needed that the tanzu cli nodepools command doesn't support today. The overlay and docs are [here](https://github.com/warroyo/tkg-overlays/tree/main/vsphere/nodepools#usage-for-new-clusters) but this will outline the steps as well.

1. Create a new directory on the workstation you use to create TKg clusters. 
   ```
   mkdir -p ~/.config/tanzu/tkg/providers/ytt/03_customizations/nodepools
   ``` 
2. Copy the `nodepools_values.yml` and the `nodepools.yml` from the above linked repo into the new `nodepools` directory. 
3. Create/modify your new cluster yaml file. This should be the file you are passing to the tanzu cli to create the cluster. Official docs for creating a workload cluster config file are [here](https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/1.6/vmware-tanzu-kubernetes-grid-16/GUID-tanzu-k8s-clusters-index.html#create-a-workload-cluster-configuration-file-1.). You can also find the full workload cluster template [here](https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/1.6/vmware-tanzu-kubernetes-grid-16/GUID-tanzu-k8s-clusters-vsphere.html).
4. Add the Nodepool settings to the cluster yaml file. Below is an example.The full option set is [here](https://github.com/warroyo/tkg-overlays/blob/main/vsphere/nodepools/cluster_config.yml) anything that is omitted will be inherited from the main cluster values. The important pieces here are the `template` and the `customVMX`. The `64bitMMIOSizeGB` shoudl be calcuated based off of your GPU size. 
```yaml
EXTRA_NODE_POOLS: true
extrapools: |
  - name: gpu-pool
    replicas: 1
    template: <full path to template we created>
    customVMX: "pciPassthru.use64BitMMIO=true,pciPassthru.64bitMMIOSizeGB=<GIB>"
    labels:
      gpu-workers: 'true' 
    taints: []
```

5. Create a cluster as you typically would. This config above will create a nodepool in the cluster called gpu-pool

```
tanzu create cluster -f <workload-ckluster-yaml>
```


6. validate the node pool is created

```bash
tanzu cluster node-pool list <cluster-name>
  NAME      NAMESPACE  PHASE    REPLICAS  READY  UPDATED  UNAVAILABLE  
  gpu-pool  default    Running  1         1      1        0            
  md-0      default    Running  1         1      1        0            
  md-1      default    Running  1         1      1        0            
  md-2      default    Running  1         1      1        0          
```

7. Validate that the gpu nodes attached the right device in vsphere.





### TKG 1.6 - existing cluster

Adding a new node pool is done using the same overlays that are used in the above step for creating a new cluster.

1. Follow steps 1-4 from the section above on creating a new cluster.

2. Change into the TKg mgmt cluster context

3. Run the following command. This command will run generate the yaml to create a new cluster but by adding `ADD_POOLS=true` it will only generate the nodepool yaml and apply that to the existing cluster.

```bash
ADD_POOLS=true tanzu cluster create -f <workload-ckluster-yaml> --dry-run | kubectl apply -f-
```

4. validate the node pool is created

```bash
tanzu cluster node-pool list <cluster-name>
  NAME      NAMESPACE  PHASE    REPLICAS  READY  UPDATED  UNAVAILABLE  
  gpu-pool  default    Running  1         1      1        0            
  md-0      default    Running  1         1      1        0            
  md-1      default    Running  1         1      1        0            
  md-2      default    Running  1         1      1        0          
```



### TKG 2.1 - new cluster


## deploy GPU operator

The GPU operator will be deployed from the NVAIE catalog. Prior to starting this step make sure you have your NGC API Token to access the private registry.

In this guide we are deploying the `v22.9.1` operator helm chart. This is due to the version alignment between the ESX Drivers and the client drivers that get deployed by the operator. You will need to be sure that you are pulling the right version of the operator and updating the script below according to what your environment needs. 

The script below takes a very standard approach to deploying the helm chart. If you need to modify any values of the chart you will need to pull down the chart values and update the values prior to deploying.

1. export your NGC api token. 

```
export NGC_API_KEY='token-here'
````

2. Get the client configuartion token from the license server. we will need to create a file called `client_configuration_token.tok` it is very important to name this exactly that. The operator requires this naming. Place this file in the working directory where you will run the below script. This file will be add into a config map by the script.


3. copy the below content into a file and make it executable

```bash
#!/bin/bash
echo "creating namespace gpu-operator…"
kubectl create namespace gpu-operator

echo "ensuring the empty gridd.conf file exists…"
touch gridd.conf

echo "creating configmap licensing-config …"
kubectl create configmap licensing-config -n gpu-operator --from-file=gridd.conf --from-file=client_configuration_token.tok 

export REGISTRY_SECRET_NAME=ngc-secret
export PRIVATE_REGISTRY=nvcr.io/nvaie

echo "creating ngc-secret…"
kubectl create secret docker-registry ${REGISTRY_SECRET_NAME} \
    --docker-server=${PRIVATE_REGISTRY} \
    --docker-username='$oauthtoken' \
    --docker-password=${NGC_API_KEY} \
    -n gpu-operator


echo "adding the helm repo nvaie…"
helm repo add nvaie https://helm.ngc.nvidia.com/nvaie \
  --username='$oauthtoken' --password=${NGC_API_KEY} \
  && helm repo update


#echo "doing the helm fetch…"

helm fetch  https://helm.ngc.nvidia.com/nvaie/charts/gpu-operator-3-0-v22.9.1.tgz --username='$oauthtoken' --password=${NGC_API_KEY} 


echo "installing GPU operator version"
helm install --wait gpu-operator gpu-operator-3-0-v22.9.1.tgz -n gpu-operator --set driver.repository=nvcr.io/nvaie --set operator.repository=nvcr.io/nvaie --set driver.imagePullPolicy=Always

echo DONE
```

4. Validate that the operator installed. 

```
kubectl get pods -n gpu-operator
```

5. Validate that the driver is licensed

```bash
kubectl exec -it ds/nvidia-driver-daemonset -n gpu-operator -- nvidia-smi -q | grep -i license
Defaulted container "nvidia-driver-ctr" out of: nvidia-driver-ctr, k8s-driver-manager (init)
    vGPU Software Licensed Product
        License Status                    : Licensed (Expiry: 2023-5-17 13:14:18 GMT)
```

**NOTES:**

* To troubleshoot licensing, exec into the driver-daemonset pod and search syslog for `nvidia-gridd`. This will show licensing errors.
* If you are using MIG, make sure that you can see the mig devcies when running `nvidia-smi` when exec'd into the driver-daemonset. 