# VGPU with TKGm

This doc outlines a process for using NVIDIA vGPU capabilties with TKGm(TKG w/standalone management cluster). **This process is not officially supported by VMware**. There are two ways of using VGPU capabilties, multi instance and time slicing. This doc will cover both.


## Install ESX Drivers for NVAIE

Follow the NVIDIA docs to install the NVAIE drivers. There is nothing unique to TKG about this process.

1. Install the VIBs - [docs here](https://docs.nvidia.com/ai-enterprise/latest/quick-start-guide/index.html#installing-grid-vgpu-manager-vmware-vsphere)
2. Disable ECC if your GPU does not support it - [docs here](https://docs.nvidia.com/ai-enterprise/latest/quick-start-guide/index.html#disabling-enabling-ecc-memory)
3. If your GPU supports MIG and you want to use it, it needs to be enabled at the esx level. - [docs here](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/#enable-mig-mode)


**Notes**
* If you have previsously had this gpu set to passthrough be sure to disable that
* `nvidia-smi -q` is very helpful to get info about ECC


## Configure the graphics settings in vCenter

In this case we do not want to enable passthrough we just want to enable the "shared direct" settings

1. navigate to the different tabs and set the gaprhics setting to "shared direct" -  [docs here](https://docs.nvidia.com/ai-enterprise/latest/quick-start-guide/index.html#changing-default-gpu-mode-vmware-vsphere)


## Setup vGPU profile based TKG templates

Becuase CAPV does not support vGPU today we need to work around this by creating some vSphere templates that already have the gpu profiles added as pci devices. CAPv support passthrough for GPU by adding a pci device, however the way that adding a vGPU works in vCenter has a slightly different api so this can't be done thropugh CAPV today.


**If you are running TKG 1.x you will first need to create an EFI ubuntu image [using BYOI](https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/1.6/vmware-tanzu-kubernetes-grid-16/GUID-build-images-linux.html). In TKG 2.x EFI templates are shipped with the product so we can just use one of those.**

### Get the list of available profiles

Get a list of the profiles that are available for your GPU.These will differ based on your GPU model. 

1. Convert the template to a VM and place it on the host with a GPU. 
2. Edit the VM settings and add a PCI device.
3. The PCI device section should give you an option for `NVIDIA GRID vGPU` choose that and click the dropdown

![](images/2023-05-12-13-23-36.png)

4. Take screenshot of this list for the next step. 


**Notes**

* You can find out what the different profiles mean through the nvidia docs. For the example above with a T4 the different types of vgpu are [here](https://docs.nvidia.com/grid/10.0/grid-vgpu-user-guide/index.html#vgpu-types-tesla-t4). Since this use case is with TKG fro traning workloads we can see on the doc the best profiles for that are the `c` profiles.


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

### TKG 1.6 new cluster

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
      gpu-workers: true 
    taints: []
```


### TKG 1.6 existing cluster

### TKG 2.1 new cluster


## deploy GPU operator

## done

