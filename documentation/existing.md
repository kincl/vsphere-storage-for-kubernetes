---
title: Configurations on Existing Kubernetes Cluster
---

## Enabling vSphere Cloud Provider

### Preferred method
If user has deployed Kubernetes cluster on vSphere then use automated way of enabling vSphere Cloud Provider. For more details, please visit [https://github.com/vmware/kubernetes/blob/enable-vcp-uxi/README.md](https://github.com/vmware/kubernetes/blob/enable-vcp-uxi/README.md). If pre-requisites are not applicable, then follow manual steps mentioned below.

**Note: If Kubernetes cluster is spanning across multiple vCenters then please use below manual steps.**

### Manual steps (if needed) to enable vSphere Cloud Provider

#### Step 1: Create a VM folder for Node VMs 

**Note: This is required only if Kubernetes version is 1.8.x or below.**

Create a VM folder. Follow instructions mentioned in this [link](https://docs.vmware.com/en/VMware-vSphere/6.0/com.vmware.vsphere.vcenterhost.doc/GUID-031BDB12-D3B2-4E2D-80E6-604F304B4D0C.html) and move Kubernetes Node VMs to this folder.

#### Step 2: Make sure node VM names are compliant

**Note: This is required only if Kubernetes version is 1.8.x or below.**

Make sure Node VM names must comply with the regex `[a-z](([-0-9a-z]+)?[0-9a-z])?(\.[a-z0-9](([-0-9a-z]+)?[0-9a-z])?)*` If Node VMs does not comply with this regex, rename them and make it compliant to this regex.

  Node VM names constraints:

  * VM names cannot begin with numbers.
  * VM names cannot have capital letters, any special characters except `.` and `-`.
  * VM names cannot be shorter than 3 chars and longer than 63

#### Step 3: Enable disk UUID on Node virtual machines

The disk.EnableUUID parameter must be set to "TRUE" for each Node VM. This step is necessary so that the VMDK always presents a consistent UUID to the VM, thus allowing the disk to be mounted properly.

For each of the virtual machine nodes that will be participating in the cluster, follow the steps below using [GOVC tool](https://github.com/vmware/govmomi/tree/master/govc)

* Set up GOVC environment

        export GOVC_URL='vCenter IP OR FQDN'
        export GOVC_USERNAME='vCenter User'
        export GOVC_PASSWORD='vCenter Password'
        export GOVC_INSECURE=1

* Find Node VM Paths

        govc ls /datacenter/vm/<vm-folder-name>

* Set disk.EnableUUID to true for all VMs

        govc vm.change -e="disk.enableUUID=1" -vm='VM Path'

Note: If Kubernetes Node VMs are created from template VM then `disk.EnableUUID=1` can be set on the template VM. VMs cloned from this template, will automatically inherit this property.

#### Step 4: Create Roles, add Privileges to Roles and assign them to the vSphere Cloud Provider user and vSphere entities

Note: If vCenter Administrator account is specified in the `vsphere.conf` file, then this step can be skipped.

vSphere Cloud Provider interacts with vCenter through the user configured in vSphere cloud config file (`vsphere.conf`) - see below section. The following minimal set of privileges are required by this user to execute relevant operations in vCenter. Please refer [vSphere Documentation Center](https://docs.vmware.com/en/VMware-vSphere/6.5/com.vmware.vsphere.security.doc/GUID-18071E9A-EED1-4968-8D51-E0B4F526FDA3.html) to know about steps for creating a Custom Role, User and Role Assignment.

<table>
<thead>
<tr>
  <th>Roles</th>
  <th>Privileges</th>
  <th>Entities</th>
  <th>Propagate to Children</th>
</tr>
</thead>
<tbody><tr>
  <td>manage-k8s-node-vms</td>
  <td>Resource.AssignVMToPool<br> System.Anonymous<br> System.Read<br> System.View<br> VirtualMachine.Config.AddExistingDisk<br> VirtualMachine.Config.AddNewDisk<br> VirtualMachine.Config.AddRemoveDevice<br> VirtualMachine.Config.RemoveDisk<br> VirtualMachine.Inventory.Create<br> VirtualMachine.Inventory.Delete</td>
  <td>Cluster,<br> Hosts,<br> VM Folder</td>
  <td>Yes</td>
</tr>
<tr>
  <td>manage-k8s-volumes</td>
  <td>Datastore.AllocateSpace<br> Datastore.FileManagement<br> System.Anonymous<br> System.Read<br> System.View</td>
  <td>Datastore</td>
  <td>No</td>
</tr>
<tr>
  <td>k8s-system-read-and-spbm-profile-view</td>
  <td>StorageProfile.View<br> System.Anonymous<br> System.Read<br> System.View</td>
  <td>vCenter</td>
  <td>No</td>
</tr>
<tr>
  <td>ReadOnly</td>
  <td>System.Anonymous<br>System.Read<br>System.View</td>
  <td>Datacenter,<br> Datastore Cluster,<br> Datastore Storage Folder</td>
  <td>Yes</td>
</tr>
</tbody>
</table>

Please note that the user configured for vSphere Cloud Provider will be used by all Kubernetes users on the same cluster. If one single user is not sufficient for different use cases, it is recommended to utilize [Kubernetes RBAC Authorization](https://kubernetes.io/docs/admin/authorization/rbac/) to drive authorization decisions, allowing admins to dynamically configure access policies through the Kubernetes API.

In some cases Administrator may still want to customize the privileges for vSphere Cloud Provider user:

1. SPBM support in vSphere Cloud Provider was introduced in Kubernetes 1.7 release. For 1.6 or older release, remove SPBM related privileges to keep required privileges to a minimal.
2. Kubernetes RBAC mode was stabilized as of 1.8 release. For 1.7 or older release, Administrator may still want to customize the privileges for vSphere Cloud Provider user based on different use cases. It's recommended to assign different roles to the same vCenter user instead of switching to a different vCenter user, in which case Kubelet needs to be restarted, which might be a concern in production environment.


Here are [some examples](https://vmware.github.io/vsphere-storage-for-kubernetes/documentation/vcp-roles.html) to customize roles and privileges for vSphere Cloud Provider user.

#### Step 5: Create the vSphere cloud config file (vsphere.conf).

##### For Kubernetes version 1.9.x and above

vSphere Cloud Provider config file needs to be placed in the shared directory which should be accessible from kubelet, controller-manager, and API server.

**```vsphere.conf```**

```
[Global]
# properties in this section will be used for all specified vCenters unless overriden in VirtualCenter section.

user = "vCenter username for cloud provider"
password = "password"        
port = "443" #Optional
insecure-flag = "1" #set to 1 if the vCenter uses a self-signed cert
datacenters = "list of datacenters where Kubernetes node VMs are present"
        
[VirtualCenter "1.2.3.4"]
# Override specific properties for this Virtual Center.
        user = "vCenter username for cloud provider"
        password = "password"
        # port, insecure-flag, datacenters will be used from Global section.
        
[VirtualCenter "1.2.3.5"]
# Override specific properties for this Virtual Center.
        port = "448"
        insecure-flag = "0"
        # user, password, datacenters will be used from Global section.
        
[Workspace]
# Specify properties which will be used for various vSphere Cloud Provider functionality.
# e.g. Dynamic provisioing, Storage Profile Based Volume provisioning etc.
        server = "IP/FQDN for vCenter"
        datacenter = "Datacenter name"
        folder = "vCenter VM folder path in which dummy VMs should be created."
        default-datastore = "Datastore name" #Datastore to use for provisioning volumes using storage classes/dynamic provisioning
        resourcepool-path = "Resource pool path" # Used fordummy VM creation. Optional

[Disk]
        scsicontrollertype = pvscsi

[Network]
        public-network = "Network Name to be used"
```

Below is summary of supported parameters in the `vsphere.conf` file for Kubernetes version 1.9.x

* ```user``` is the vCenter username for vSphere Cloud Provider. Note that special characters such as `\` need to be escaped with `\`.
* ```password``` is the password for vCenter user specified with `user`. Note that special characters such as `\` need to be escaped with `\`.
* ```port``` is the vCenter Server Port. Default is 443 if not specified.
* ```insecure-flag``` is set to 1 if vCenter used a self-signed certificate.
* ```server``` is the vCenter Server IP or FQDN
* ```datacenters``` is the names of the datacenters on which Node VMs are deployed.
* ```default-datastore``` is the default datastore to use for provisioning volumes using storage classes/dynamic provisioning.
* ```resourcepool-path``` is the path to resource pool where dummy VMs for Storage Profile Based volume provisioning should be created. It is optional parameter.
* ```folder``` VM Folder path where Node VMs can be present.

##### For Kubernetes version 1.8.x or below

vSphere Cloud Provider config file needs to be placed in the shared directory which should be accessible from kubelet container, controller-manager pod, and API server pod.

**```vsphere.conf``` for Master Node:**

```
[Global]
        user = "vCenter username for cloud provider"
        password = "password"
        server = "IP/FQDN for vCenter"
        port = "443" #Optional
        insecure-flag = "1" #set to 1 if the vCenter uses a self-signed cert
        datacenter = "Datacenter name"
        datastore = "Datastore name" #Datastore to use for provisioning volumes using storage classes/dynamic provisioning
        working-dir = "vCenter VM folder path in which node VMs are located"
        vm-name = "VM name of the Master Node" #Optional
        vm-uuid = "UUID of the Node VM" # Optional
[Disk]
    scsicontrollertype = pvscsi
```

Note: **```vm-name``` parameter is introduced in 1.6.4 release.** Both ```vm-uuid``` and ```vm-name``` are optional parameters. If ```vm-name``` is specified then ```vm-uuid``` is not used. If both are not specified then kubelet will get vm-uuid from `/sys/class/dmi/id/product_serial` and query vCenter to find the Node VM's name.

**```vsphere.conf``` for Worker Nodes:** (Only Applicable to 1.6.4 release and above. For older releases this file should have all the parameters specified in Master node's ```vSphere.conf``` file)

```
[Global]
        vm-name = "VM name of the Worker Node"
```

Below is summary of supported parameters in the `vsphere.conf` file

* ```user``` is the vCenter username for vSphere Cloud Provider.
* ```password``` is the password for vCenter user specified with `user`.
* ```server``` is the vCenter Server IP or FQDN
* ```port``` is the vCenter Server Port. Default is 443 if not specified.
* ```insecure-flag``` is set to 1 if vCenter used a self-signed certificate.
* ```datacenter``` is the name of the datacenter on which Node VMs are deployed.
* ```datastore``` is the default datastore to use for provisioning volumes using storage classes/dynamic provisioning.
* ```vm-name``` is recently added configuration parameter. This is optional parameter. When this parameter is present, ```vsphere.conf``` file on the worker node does not need vCenter credentials.

  **Note:** ```vm-name``` is added in the release 1.6.4. Prior releases does not support this parameter.

* ```working-dir``` can be set to empty ( working-dir = ""), if Node VMs are located in the root VM folder.
* ```vm-uuid``` is the VM Instance UUID of virtual machine. ```vm-uuid``` can be set to empty (```vm-uuid = ""```). If set to empty, this will be retrieved from /sys/class/dmi/id/product_serial file on virtual machine (requires root access).

  * ```vm-uuid``` needs to be set in this format - ```423D7ADC-F7A9-F629-8454-CE9615C810F1```

  * ```vm-uuid``` can be retrieved from Node Virtual machines using following command. This will be different on each node VM.

        cat /sys/class/dmi/id/product_serial | sed -e 's/^VMware-//' -e 's/-/ /' | awk '{ print toupper($1$2$3$4 "-" $5$6 "-" $7$8 "-" $9$10 "-" $11$12$13$14$15$16) }'

* `datastore` is the default datastore used for provisioning volumes using storage classes. If datastore is located in storage folder or datastore is member of datastore cluster, make sure to specify full datastore path. Make sure vSphere Cloud Provider user has Read Privilege set on the datastore cluster or storage folder to be able to find datastore.
  * For datastore located in the datastore cluster, specify datastore as mentioned below

        datastore = "DatastoreCluster/datastore1"

  * For datastore located in the storage folder, specify datastore as mentioned below

        datastore = "DatastoreStorageFolder/datastore1"

#### Step 6: Add flags to controller-manager, API server and Kubelet

##### For Kubernetes version 1.9.x

* Add following flags to 
  - kubelet running on master node 
  - controller-manager manifest file
  - API server manifest file

        --cloud-provider=vsphere
        --cloud-config=<Path of the vsphere.conf file>

* Add following flags to kubelet running on each worker node.

        --cloud-provider=vsphere

##### For Kubernetes version 1.8.x or below

Add following flags to 
- kubelet running on all nodes
- controller-manager manifest file (typically running on master node)
- API server manifest file (typically running on master node)

        --cloud-provider=vsphere
        --cloud-config=<Path of the vsphere.conf file>

Manifest files for API server and controller-manager are generally located at `/etc/kubernetes`

#### Step 7: Restart Kubelet on all nodes

* Reload kubelet systemd unit file using ```systemctl daemon-reload```
* Restart kubelet service using ```systemctl restart kubelet.service```

**Note: For Kubernetes version 1.8.x or below, after enabling the vSphere Cloud Provider, Node names will be set to the VM names from the vCenter Inventory**
