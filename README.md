# Introduction
The project aims to evaluate the performance, after any major events on the infrastructure like deployment, update or upgrade.  It measures packet throughput, bandwidth and latency of different NFV scenarios like guest VMs using SR-IOV VFs or OvS-DPDK vhostuser ports (primarily focussed on Fast Datapath).

## Architecture
### OvS-DPDK Setup


```+----------------------+
|  +----------------+  |
|  |  Testpmd VM    |  |
|  +-+------------+-+  |
|    |            |    |
| +--+------------+--+ |
| |    OVS DPDK      | |
| +--+------------+--+ |
|    |            |    |
|    | Compute 1  |    |
|  +-+-+        +-+-+  |
|  |   |        |   |  |
+--+-+-+--------+-+-+--+
     |            |
+----+------------+----+
|      TOR Switch      |
|                      |
+----+------------+----+
     |            |
+----+------------+----+
| || P ||      || P || |
| || F ||      || F || |
| +--+--+      +--+--+ |
|    | Compute 2  |    |
| +--+------------+--+ |
| |                  | |
| |       TReX VM    | |
| +------------------+ |
+----------------------+

"OSP Network Info:",
"  Bridge Mappings:      datacentre:br-ex,data1:br-link0,data2:br-link1",
"  Network VLANRanges:   datacentre:1205:1210,data1:201:220,data2:201:220,sriov1:201:220,sriov2:201:220",
"  Flat Networks:        datacentre,data1,data2",
"  Physical Dev Mapping: sriov1:em1,sriov2:em2",
```

### SR-IOV VF Setup
```+----------------------+
|  +----------------+  |
|  |   Testpmd VM   |  |
|  +-+------------+-+  |
|    |            |    |
|    | Compute 1  |    |
| +--+--+      +--+--+ |
| ||V  ||      || V || |
| ||F  ||      || F || |
+----+------------+----+
     |            |
+----+------------+----+
|         TOR Switch   |
|                      |
+----+------------+----+
     |            |
+----+------------+----+
| || P ||      || P || |
| || F ||      || F || |
| +--+--+      +--+--+ |
|    | Compute 2  |    |
| +--+------------+--+ |
| |                  | |
| |    TReX  VM      | |
| +------------------+ |
+----------------------+

```
## TReX Execution
TRex benchmark run by ansible-nfv, will by default determine the maximum throughput, using binary search, that could be attained with zero packet loss using frame size of 64 and number of flows set to 1.


* Install prerequisites to run the packet generator from the undercloud.
```
virtualenv venv
source venv/bin/activate
git clone --recurse-submodules https://github.com/yogananth-subramanian/post-deployment-validate.git
cd post-deployment-validate
patch -p1 --dry-run -d ansible-nfv/ < ../patch
pip install -r requirements.txt
```
* Initialize ansible inventory.
```
ansible-playbook initialize.yaml -e host=<UndercloudIP>  -e user=stack -e ssh_key=~/.ssh/id_rsa -e setup_type=baremetal
```
* Cluster Specific Configuration
 
  **user-specific/trex.yaml** file defines the minimal set of parameters required to run the performance validation. 

  * Only 3 mandatory variables need to be set:
    * **extern** to specify the external network configured in Openstack, 
    * **physical_network_pf** for specifying the SRIOV physical interface for TRex.
    * Interface  for testpmd ( **physical_network_dpdk** or  **physical_network_vf**).

* Running Tests
To start execution of trex benchmark, run:
```
ansible-playbook -i inventory  trex-run.yaml  --extra @user-specific/trex.yaml -e build_image=true -vvv
```

* To cleanup trex resources, run:
  ```
  ./trex_cleanup.sh
  ```

## Examples of Physical to Virtual back to Physical(PVP) senarios
## Senarios 1 - Device Under Test (DUT) backed by DPDK


```+----------------------+
|  +----------------+  |
|  |  Testpmd VM    |  |
|  +-+------------+-+  |
|    |            |    |
| +--+------------+--+ |
| |    OVS DPDK      | |
| +--+------------+--+ |
|    |            |    |
|    | Compute 1  |    |
|  +-+-+        +-+-+  |
|  |   |        |   |  |
+--+-+-+--------+-+-+--+
     |            |
+----+------------+----+
|      TOR Switch      |
|                      |
+----+------------+----+
     |            |
+----+------------+----+
| || P ||      || P || |
| || F ||      || F || |
| +--+--+      +--+--+ |
|    | Compute 2  |    |
| +--+------------+--+ |
| |                  | |
| |       TReX VM    | |
| +------------------+ |
+----------------------+

"OSP Network Info:",
"  Bridge Mappings:      datacentre:br-ex,data1:br-link0,data2:br-link1",
"  Network VLANRanges:   datacentre:1205:1210,data1:201:220,data2:201:220,sriov1:201:220,sriov2:201:220",
"  Flat Networks:        datacentre,data1,data2",
"  Physical Dev Mapping: sriov1:em1,sriov2:em2",
```
For the above network config, **user-specific/trex.yml** would be set as follows:

**extern: 'datacentre'** #[1]  
**mgmt: 'data1'**        #[2]  
**physical_network_dpdk: ['data1','data2']**  #[3]  
**physical_network_pf: ['sriov1', 'sriov2']** #[4]  

**[1]** The external floating network is mapped to **"datacenter"**, so **"extern"** is set to datacentre.    
**[2]** Since the DPDK Compute1 node does not have DPDK-VXLAN network, one of provider network **"data1"** is used as management network for the Testpmd VM, hence **"mgmt"** is set to data1. On compute2 a VXLAN network would be created for use as management.  
**[3]** Provider network based on **"data1"** and **"data2"**, within the VLANRanges, will be created on Compute1.  
**[4]** Provider network based on **"sriov1"** and **"sriov2"** will be created on Compute2.  

## Senarios 2 - Device Under Test (DUT) backed by SRIOV-VF

```+----------------------+
|  +----------------+  |
|  |   Testpmd VM   |  |
|  +-+------------+-+  |
|    |            |    |
|    | Compute 1  |    |
| +--+--+      +--+--+ |
| ||V  ||      || V || |
| ||F  ||      || F || |
+----+------------+----+
     |            |
+----+------------+----+
|         TOR Switch   |
|                      |
+----+------------+----+
     |            |
+----+------------+----+
| || P ||      || P || |
| || F ||      || F || |
| +--+--+      +--+--+ |
|    | Compute 2  |    |
| +--+------------+--+ |
| |                  | |
| |    TReX  VM      | |
| +------------------+ |
+----------------------+

```

## Scenarios supported at present
### Perf TRex and Testpmd on vanilla OSP install
| Parameter | Description |
| --- | --- |
| extern | (Mandatory) External network name for allocation of Floating IP |
| mgmt | (Optional) Management network to be used for VM access. This parameter can be ignored if the VM has the tenant network access for management. |
| physical_network_pf | (Mandatory) List of SR-IOV physical networks names to be used for PF allocation to TRex VM. |
| physical_network_vf | Required for SR-IOV VF tests. List of SR-IOV physical network names to be used for VF allocation to TestPMD VM |
| physical_network_dpdk | Required for OvS-DPDK vhostuser tests. List of OvS-DPDK physical network names to be used for vhostuser port allocation to TestPMD VM |
| testpmd_compute | (Mandatory) Compute on which Testpmd should be deployed.|

### Vanilla Perf with build image
| Parameter | Description |
| --- | --- |
| extern | (Mandatory) External network name for allocation of Floating IP |
| mgmt | (Optional) Management network to be used for VM access. This parameter can be ignored if the VM has the tenant network access for management. |
| physical_network_pf | (Mandatory) List of SR-IOV physical networks names to be used for PF allocation to TRex VM. |
| physical_network_vf | Required for SR-IOV VF tests. List of SR-IOV physical network names to be used for VF allocation to TestPMD VM |
| physical_network_dpdk | Required for OvS-DPDK vhostuser tests. List of OvS-DPDK physical network names to be used for vhostuser port allocation to TestPMD VM |
| build_image | (Mandatory) set to Ture |
| images | (Optional) Centos Cloud image would be used as default |
| testpmd_compute | (Mandatory) Compute on which Testpmd should be deployed |

### Perf with existing TRex and Testpmd VM in OSP
| Parameter | Description |
| --- | --- |
| trex_vm | (Mandatory) TReX VM name and access detail |
| testpmd_vm | (Mandatory) Testpmd VM name and access detail |
| dut_macs | (Mandatory) Testpmd VM mac address |
| trex_macs | (Mandatory) TReX VM mac address |
| trex_vlans | (Mandatory if VLAN is required)  VLANs to use for TReX traffic |
| testpmd_compute | (Mandatory) Compute on which Testpmd should be deployed |

### Perf with existing data path networks
| Parameter | Description |
| --- | --- |
| extern | (Mandatory) External network name for allocation of Floating IP |
| mgmt | (Optional) Management network to be used for VM access. This parameter can be ignored if the VM has the tenant network access for management. |
| physical_network_pf | (Mandatory) List of SR-IOV physical networks names to be used for PF allocation to TRex VM. |
| physical_network_vf | Required for SR-IOV VF tests. List of SR-IOV physical network names to be used for VF allocation to TestPMD VM |
| physical_network_dpdk | Required for OvS-DPDK vhostuser tests. List of OvS-DPDK physical network names to be used for vhostuser port allocation to TestPMD VM |
| testpmd_compute | (Mandatory) Compute on which Testpmd should be deployed |
| {network name} | Attributes to override for the network. <p>Example:<p>extern: 'datacentre'<br/>datacentre: {gateway_ip: 172.16.0.1, "physical_network": "datacentre","cidr": "172.16.0.0/24","allocation_pool_start": "172.16.0.226","allocation_pool_end": "172.16.0.236" | 

