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
