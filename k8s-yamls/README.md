# k8s-yamls

Curated YAML files (and respective container images) for Multus CNI, OVN CNI, SR-IOV Device Plugin, SR-IOV CNI and other
k8s networking related projects.

# Getting Started

## Setting up Kubernetes nodes (master and workers)
1. Start with one master and two worker nodes -- all of them running CentOS 8 (other distributions like Ubuntu
   should work as well). Note: steps below will also work for multi-master k8s cluster setup.
   ```                                                                                                             
                                       ─────┬────────────────────────┬────Stream Network────┬─────────           
                                            │                        │                      │                    
                                         ┌──┴───┐                 ┌──┴───┐               ┌──┴───┐                
    ┌───────────────┐               ┌────┤ eno3 ├───┐        ┌────┤ eno3 ├───┐      ┌────┤ eno3 ├──────┐         
    │               │               │    └──────┘   │        │    └──────┘   │      │    └──────┘      │         
    │               │               │               │        │               │      │K8s Network Server│         
    │  K8s Master   │               │   K8s Node1   │        │   K8s Node2   │      │       Node       │         
    │    ┌──────┐   │               │   ┌──────┐    │        │    ┌──────┐   │      │ ┌──────┐ ┌──────┐│         
    └────┤ eno2 ├───┘               └───┤ eno2 ├────┘        └────┤ eno2 ├───┘      └─┤ eno2 ├─┤ eno4 ├┘         
         └──┬───┘                       └───┬──┘                  └──┬───┘            └───┬──┘ └───┬──┘          
            │                               │                        │                    │        │             
            │  1. Storage Network (VLAN Y)  │                        │                    │        │  Public     
    ────────┴──2. Admin Network ────────────┴────────────────────────┴────────────────────┴───   ──┴─Network──   
    ```
    
2. Install kubeadm, kubelet, and kubectl at version 1.16+ on all of the worker and master nodes 
   
   NOTE: If kubelet service is failing to start and complains about missing `tc` utility, then
   install iproute-tc package.
   
3. Install OVS 2.13 on each of the host
    ```text
   yum install -y epel-release
   curl -s https://38a0484b1764796f6ac26ab0eb03336ee168ab7fb3ac03a9:@packages.nvidia.com/install/repositories/sdn/nv-ovs-2-13-0-1/script.rpm.sh | sudo bash
   yum install -y openvswitch network-scripts-openvswitch
   systemctl enable openvswitch
   systemctl restart openvswitch
   # verify OVS is running
   ovs-vsctl --version && ovs-vsctl list Open_vSwitch
   ```
   **NOTE:** Above command will upgrade your current version of OVS to the latest version, 2.13.
   
4. Install and setup network-scripts on each of the host 
   ```text
   yum install network-scripts -y
   systemctl disable NetworkManager
   systemctl stop NetworkManager
   systemctl enable network
    ```
   
5. Determine the primary interface on the host. This is the interface through which default gateway is accessible.
   On my system, it is `eno2`. Create network-scripts configuration with name ifcfg-eno2, ifcfg-breno2 under
   /etc/sysconfig/network-scripts/.
   
    ```
    ┌────────────────┐          ┌───────────────┐
    │Node1           │          │Node1          │
    │                │          │               │
    │                │          │ ┌────┐        │
    │                │          │ │ IP │        │
    │                │          │ └─┬──┘        │
    │                │          │   │           │
    │ ┌────┐         │          │┌──┴───┐       │
    │ │ IP │         │          ││breno2│       │
    │ └──┬─┘         │          │└──┬───┘       │
    │    │           │          │   │           │
    │ ┌──┴─┐  ┌────┐ │          │ ┌─┴──┐ ┌────┐ │
    └─┤eno2├──┤eno3├─┘          └─┤eno2├─┤eno3├─┘
      └────┘  └────┘              └────┘ └────┘    
   ```

   ```text
   $ cat /etc/sysconfig/network-scripts/ifcfg-eno2 
    IPV6INIT=no
    BOOTPROTO=none
    DEVICE=eno2
    DEVICETYPE=ovs
    TYPE=OVSPort
    OVS_BRIDGE=breno2
    ONBOOT=yes
    NM_CONTROLLED="no"
    ```
    If DHCP configuration is needed, then we need this.

   ```text
   #Get the hardware mac address of the interface.
   $ cat /sys/class/net/eno2/address 
     d0:17:07:05:03:03
   $ cat /etc/sysconfig/network-scripts/ifcfg-breno2
     DEFROUTE="yes"
     IPV4_FAILURE_FATAL="no"
     IPV6INIT="no"
     NAME=breno2
     DEVICE=breno2
     ONBOOT=yes
     TYPE=OVSBridge
     OVSBOOTPROTO=dhcp
     OVSDHCPINTERFACES=eno2
     OVS_EXTRA="set bridge breno2 other-config:hwaddr=d0:17:07:05:03:03"
     HOTPLUG=no
    ```

    If static IP configuration is desired, then we need to use this:

   ```text
   #Get the hardware mac address of the interface.
   $ cat /sys/class/net/eno2/address 
     d0:17:07:05:03:03
   $ cat /etc/sysconfig/network-scripts/ifcfg-breno2
     TYPE="OVSBridge"
     BOOTPROTO="static"
     DEFROUTE="yes"
     IPV4_FAILURE_FATAL="no"
     IPV6INIT="no"
     NAME="breno2"
     DEVICE="breno2"
     DEVICETYPE="ovs"
     ONBOOT="yes"
     HOTPLUG=no
     IPADDR=10.0.1.7
     NETMASK=255.255.255.0
     GATEWAY=10.0.1.1
     DNS1="10.0.0.7"
     DNS2="8.8.8.8"
     NM_CONTROLLED="no"
     OVS_EXTRA="set bridge breno2 other-config:hwaddr=d0:17:07:05:03:03"
    ```

   NOTE: substitute your node's interface name for the file name  and file's contents

   After ifcfg-eno2 and ifcfg-breno2 is created, run the following command to create the breno2.
   ```text
   systemctl start network
    ```

## Firewall rules

This is a very important step and following ports must be opened up for OVN to work properly

### Firewall rules on OVN DB Nodes

On K8s nodes where OVN DB Pods run, you will need to open up following TCP Ports
1. TCP Port 6641, 6642, 6643, and 6644

**NOTE:** If you don't have a dedicated node for OVN DBs, then open up the above ports on the K8s master since
OVN DBs will run on that node.

### Firewall rules on ALL the K8s nodes (masters and workers)

On all of the K8s nodes, following UDP Ports must be opened up
1. UDP Port 6081
   
## Initializing K8s master
If you are running K8s version 1.16 and above, then use this to initialize a K8s cluster.

   ```text
    kubeadm init --apiserver-advertise-address=<API_SERVER_ADDRESS> \
       --skip-phases addon/kube-proxy  --.....
   ```

Essentially, we do not use `kube-proxy` while using OVN.

## Applying YAMLs

We are going to first install Multus CNI, then OVN CNI, then SR-IOV DP, and finally SR-IOV CNI.
1. First set K8SNET_VERSION to the right tag. For example:
    > export K8SNET_VERSION=v0.0.6
    
2. Setup quay.io credentials to access SDN container images referenced in the yaml file.

   Before we get started with applying yaml files, one will need to create a K8s
    secret resource that contains the necessary credentials to authenticate to
    quay.io for pulling the private images. We are going to use a `robot` account under quay.io's SDN team called `nvidia+sdn`.
    This user has `read` only permission to various repositories and can pull images from those repositories.
    To keep this simple and automated, the K8s secret resource snippet is already captured in various yaml files
    below.
    
    **Note:** if you want to pull images using your own account, then edit registry-credentials secret
        using `kubectl secret` command. Open the link below for more information.
        
        https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/


## Multus CNI
1. Download all the yaml files for Multus
    ```text
    curl -so multus.yaml https://gitlab-master.nvidia.com/sdn/k8s-yaml/raw/${K8SNET_VERSION}/multus/multus.yaml
    curl -so sriov-public-net-crd.yaml https://gitlab-master.nvidia.com/sdn/k8s-yaml/raw/${K8SNET_VERSION}/multus/sriov-public-net-crd.yaml
    curl -so sriov-storage-net-crd.yaml https://gitlab-master.nvidia.com/sdn/k8s-yaml/raw/${K8SNET_VERSION}/multus/sriov-storage-net-crd.yaml
    curl -so sriov-stream-net-crd.yaml https://gitlab-master.nvidia.com/sdn/k8s-yaml/raw/${K8SNET_VERSION}/multus/sriov-stream-net-crd.yaml
    ```
    
2. Apply the multus.yaml file which also creates network-attachment-definition CRD. It also creates secrets
necessary to pull the private images from quay.io/nvidia namespace.
    ```text
    kubectl apply -f multus.yaml
    ```
    
3. Apply the network CRDs for SR-IOV networks
    ```text
    kubectl apply -f sriov-public-net-crd.yaml -f sriov-storage-net-crd.yaml -f sriov-stream-net-crd.yaml
    ```
    **Note:** The IPAM related to SR-IOV networks will be covered in SR-IOV CNI section. Sine the
    IPAM information is going to be per-node, we don't have to specify it at the CRD level.
    
    **Note:** There is no need for OVN CRD because it is baked into Multus as the Cluster Network
    provider

4. Verify that Multus daemonsets are running
    ```text
    kubectl get pods -n kube-system |grep multus
    ``` 

## OVN CNI

1. Download all the yaml files for OVN CNI
    ```text
    curl -so ovn-setup.yaml https://gitlab-master.nvidia.com/sdn/k8s-yaml/raw/${K8SNET_VERSION}/ovn/centos/shared/ovn-setup.yaml
    curl -so ovnkube-db.yaml https://gitlab-master.nvidia.com/sdn/k8s-yaml/raw/${K8SNET_VERSION}/ovn/centos/shared/ovnkube-db.yaml
    curl -so ovnkube-db-raft.yaml https://gitlab-master.nvidia.com/sdn/k8s-yaml/raw/${K8SNET_VERSION}/ovn/centos/shared/ovnkube-db-raft.yaml   
    curl -so ovnkube-master.yaml https://gitlab-master.nvidia.com/sdn/k8s-yaml/raw/${K8SNET_VERSION}/ovn/centos/shared/ovnkube-master.yaml
    curl -so ovnkube-node.yaml https://gitlab-master.nvidia.com/sdn/k8s-yaml/raw/${K8SNET_VERSION}/ovn/centos/shared/ovnkube-node.yaml
    ```

2. Edit OVN configmap to update the information on K8s API server, Cluster IP CIDR, and Pod network CIDR.
   Open the ovn-setup.yaml and in ConfigMap resource section update the following fields:
   - k8s_apiserver
   - net_cidr (this is the network for the pods to be on. See the diagram below)
   - svc_cidr (this is service-cluster-ip-range used by the K8s apiserver. It's normally 10.96.0.0/12)
   
    ```
                                .───────.                           
    ┌─────────────────┐      ,─'         '─.      ┌────────────────┐
    │   K8s Cluster   │     ;  K8s Service  :     │POD Network CIDR│
    │Service IP Range ├─────: Load Balancer ;─────┤(192.168.0.0/16)│
    │ (10.96.0.0/12)  │      ╲             ╱      │                │
    └─────────────────┘       '─.       ,─'       └────────────────┘
                                 `─────'                            
   ```
   
3. Create OVN namespace, service accounts, ovnkube-db headless service, configmap, and RBAC
    ```text
    kubectl apply -f ovn-setup.yaml
    ```
    
4. There are two options here -- Standalone DB mode and Clustered DB mode
   - 4.1 Running OVN DB in standalone mode without HA.
   
        For this just apply the `ovnkube-db.yaml` file.
   The ovnkube-db pod will get scheduled on one of the K8s master nodes. This workload is a deployment
   with replica set to 1 (changing the replicas to higher than 1 will not provide HA and is not supported)

        ```text
        kubectl apply -f ovnkube-db.yaml
        ```

   - 4.2 Running OVN DB in clustered mode for HA (based on RAFT algorithm).
       **NOTE:**: If you want to upgrade from Standalone DB to Clustered DB read first the next section 4.3
   
        a. First label three nodes where you want the ovnkube-db pods to be scheduled
        ```text
        kubectl label nodes <NODE_SET_ASIDE_FOR_OVN_DB_ON_ADMIN_SERVER1> k8s.ovn.org/ovnkube-db=true
        kubectl label nodes <NODE_SET_ASIDE_FOR_OVN_DB_ON_ADMIN_SERVER2> k8s.ovn.org/ovnkube-db=true
        kubectl label nodes <NODE_SET_ASIDE_FOR_OVN_DB_ON_ADMIN_SERVER3> k8s.ovn.org/ovnkube-db=true
        ```
        **NOTE:** Folks who want to try HA but don't have dedicated nodes and instead have three K8s master nodes can
         label their master nodes and the OVN DB pods will end up on master.

        b. Next apply the `ovnkube-db-raft.yaml` file.
        
        This workload is defined as a statefulset with replicas set to 3. The three ovnkube-db-{0,1,2} pods will
        get scheduled on nodes with label k8s.ovn.org/ovnkube-db=true

        ```text
        kubectl apply -f ovnkube-db-raft.yaml
        ```
   - 4.3 Upgrading from standalone DB to Clustered DB

        a. On the node where standalone db pod (`ovnkube-db-blah`) is running, the OVN DB files are located at
        ```text
         /var/lib/openvswitch/ovnnb_db.db
         /var/lib/openvswitch/ovnsb_db.db
        ```

        b. Create a backup of those files

        c. Copy over both of the above files to the three OVN DB nodes under `/var/lib/openvswitch` folder

        d. Then apply the `ovnkube-db-raft.yaml`

5. Run ovnkube-master deployment
    ```text
    kubectl apply -f ovnkube-master.yaml
    ```

6. Run ovnkube daemonsets for nodes
    ```text
    kubectl apply -f ovnkube-node.yaml
    ```
    
7. Verify that OVN CNI Pods are running
    ```text
    kubectl get pods -n ovn-kubernetes
    ```

### Advanced configurations

1. Custom MAC for an OVN interface

   It is implemented per K8s Network Plumbers WG specification found here: 
https://docs.google.com/document/d/1Ny03h6IDVy_e_vmElOqR7UdTPAG_RNydhVE1Kx54kFQ/edit#

   First, apply the OVN CRD yaml as shown below

    ```text
   curl -so ovn-crd.yaml https://gitlab-master.nvidia.com/sdn/k8s-yaml/raw/${K8SNET_VERSION}/multus/ovn-crd.yaml
   kubectl apply -f ovn-crd.yaml
    ```

   Next on the pod specify the annotation like below to get the custom MAC of your choice.

    ```text
   annotations:
    v1.multus-cni.io/default-network: |
      [
        {
          "name":"ovn-kubernetes",
          "mac": "02:23:45:67:89:01",
          "namespace": "default"
        }
      ]
    ```

2. Default-route on a non-OVN interface

    It’s generally expected that the the cluster-wide default network attachment sets a default route.
    This default route may be overridden using the “default-route” key in the Network Attachment
    Selection Annotation (see 4.1.2.1.9 in the above document) on another attachments. In that case, we
    should skip adding default route on the OVN interface and instead just add cluster subnet routes for
    the OVN interface.
    
    Consider the set of annotations below on a pod:
    ```text
      annotations:
              v1.multus-cni.io/default-network: ovn-kubernetes
              k8s.v1.cni.cncf.io/networks: |
                [
                  {
                    "name":"sriov-storage-net",
                    "default-route": ["10.1.32.2"]
                  }
                ]
    ```
    The first interface to the pod is provided by ovn-kubernetes and the second interface is through
    SRIOV CNI. The default route for that pod should be through the second interface. So, the routes
    will be something like this:
    ```text

    $ ip ro
    default via 10.1.32.2 dev net1
    10.1.32.0/19 dev net1 proto kernel scope link src 10.1.32.142
    192.168.0.0/24 dev eth0 proto kernel scope link src 192.168.0.5
    192.168.0.0/16 via 192.168.0.1 dev eth0
    ```
   
    192.168.0.0/16 is the OVN Kubernetes Cluster Subnet for which the POD
    will have a route through OVN's interface (aka eth0)


## SR-IOV Device Plugin

1. Download all the yaml files for SR-IOV Device Plugin
    ```text
    curl -so sriov-dp.yaml https://gitlab-master.nvidia.com/sdn/k8s-yaml/raw/${K8SNET_VERSION}/sriov-dp/sriov-dp.yaml
    ```

2. We are going to have different classes of servers:
   -- BladeRunner servers with Intel (BR 1.0) or Mellanox (BR 1.5) NICs to run games
   -- Admin/Management/Network servers with Mellanox to run various NGN/GFN infrastructure services
   The SR-IOV supported NICs on these servers will get enumerated differently. On one server class it could be that the
   SR-IOV interface is eno2, but on an another server it could be eno3. So, we cannot use SR-IOV device plugin's
   ConfigMap to set the SR-IOV device selector globally. It has to be per-node.
   
   On the node with SR-IOV device, say eno2, perform the following steps for the device plugin to discover the
   SR-IOV devices.
   
   ```text
   # Create 4 VFs
   echo 4 > /sys/class/net/eno2/device/sriov_numvfs
 
   # make sure vfio-pci is loaded
   $ lsmod | grep vfio
     vfio_pci               53248  0
     vfio_virqfd            16384  1 vfio_pci
     vfio_iommu_type1       28672  0
     vfio                   36864  2 vfio_iommu_type1,vfio_pci
     irqbypass              16384  2 vfio_pci,kvm

   # else load it
   $ modprobe vfio-pci
    
   # Bind the VF devices to VFIO; Assume the 4 VF devices are: 0000:01:10.0, 0000:01:10.1. 0000:01:10.2 and
   # 0000:01:10.3. The PCI bus device function for these VFs can be obtained from the link
   # /sys/class/net/eno2/device/virtfnXX. For each of these VFs.
   
   $ echo 0000:01:10.0 > /sys/class/net/eno2/device/virtfn0/driver/unbind

   # get the dev info using:
 
   $ devinfo=`lspci -n -s 0000:01:10.0  | sed 's/:/ /g' | awk -e '{print $4 " " $5}'`
   $ echo $devinfo > /sys/bus/pci/drivers/vfio-pci/new_id
 
   # Repeat the above for 0000:01:10.1 (virtfn1) 0000:01:10.2 (virtfn2)and 0000:01:10.3 (virtfn3). 
   
   # 8086 is the VendorID and 37cd is the DeviceID (from $devinfo above).
   # create the following file /etc/pcidp/config.json and populate it with
    {
        "resourceList": [{
                "resourceName": "sriov_storage",
                "selectors": {
                    "vendors": ["8086"],
                    "devices": ["37cd"],
                    "drivers": ["vfio-pci"],
                    "pfNames": ["eno2"]
                }
            }
        ]
    }
   
   # similarly do it for other SR-IOV NIC and append the information to the aforementioned file
   # In my case, the 2nd interface was eno3 and my final file looks like
   {
       "resourceList": [{
               "resourceName": "sriov_stream",
               "selectors": {
                   "vendors": ["8086"],
                   "devices": ["37cd"],
                   "drivers": ["vfio-pci"],
                   "pfNames": ["eno3"]
               }
           },
           {
               "resourceName": "sriov_storage",
               "selectors": {
                   "vendors": ["8086"],
                   "devices": ["37cd"],
                   "drivers": ["vfio-pci"],
                   "pfNames": ["eno2"]
               }
           }
       ]
   }
    ````  
 
3. Finally apply the sriov-dp.yaml
    ```text
    kubectl apply -f sriov-dp.yaml
    ```

4. Verify that SR-IOV Device plugin daemonset is running
    ```text
    kubectl get pods -n kube-system | grep sriov-device
    ```

5. Verify that the SR-IOV device plugin has successfully registered the SR-IOV VFs with the kubelet
    ```text
    kubectl get nodes -o yaml |grep 'intel.com/sriov_'
    ```

## SR-IOV CNI

1. Download all the yaml files for SR-IOV CNI
    ```text
    curl -so sriov-cni.yaml https://gitlab-master.nvidia.com/sdn/k8s-yaml/raw/${K8SNET_VERSION}/sriov-cni/sriov-cni.yaml
    ```

2. On each of the worker nodes where SR-IOV networks will be used, we need to drop SRIOV CNI configuration
file in '/etc/cni/net.d' that contains the IPAM information for those networks.

    For example:
    ```
    root@node1:/etc/cni/net.d# cat sriov-storage-net.conf
    {
      "cniVersion": "0.3.1",
      "type": "sriov",
      "name": "sriov-storage-net",
      "vlan": 42,
      "ipam": {
        "type": "host-local",
        "subnet": "10.1.32.0/19",
        "rangeStart": "10.1.32.121",
        "rangeEnd": "10.1.32.140",
        "routes": [{
          "dst": "0.0.0.0/0"
        }],
        "gateway": "10.1.32.1"
      }
    }
    ```
    
    What it says is that on `node1` the VFs belonging to SRIOV-STORAGE-NET will get IPs in the range of
    10.1.32.121 - 10.1.32.140. Those VFs will still be in the larger 10.1.32.0/19 CIDR, but on this
    node we have artificially controlled the range. We will need to do this on all of the nodes that is
    going to serve SR-IOV networks. We will work with the `Foreman` team to create this file on such nodes
    ahead of time (or we might explore a K8s way of distributing the IPAM files (TBD)). Continuing with
    this example, on node2 everything will be same except that rangeStart will now be 10.1.32.141 and
    rangeEnd will be 10.1.32.160.
    
    Similarly, for the sriov-stream-net SR-IOV network we will have
   
    ```text
    {
      "cniVersion": "0.3.1",
      "type": "sriov",
      "name": "sriov-stream-net",
      "vlan": 41,
      "ipam": {
        "type": "host-local",
        "subnet": "10.1.0.0/19",
        "rangeStart": "10.1.0.121",
        "rangeEnd": "10.1.0.140",
        "routes": [{
          "dst": "0.0.0.0/0"
        }],
        "gateway": "10.1.0.1"
      }
    }
    ```
   
3. Support for changing MTU on the SR-IOV Networks. So, say you want to change the MTU of the storage network
to 8900, then you would just add `"mtu": 8900` field to the SR-IOV network configuration file.

    For example:
   
    ```text
    {
      "cniVersion": "0.3.1",
      "type": "sriov",
      "name": "sriov-storage-net",
      "vlan": 42,
      "mtu": 8900
    <snipped>
    }
    ```
 
