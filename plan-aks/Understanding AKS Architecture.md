# Azure Kubernetes Service (AKS) Overview

Azure Kubernetes Service (AKS) is a **managed Kubernetes service** that simplifies deploying, managing, and scaling containerized applications.

---

## Key Components of AKS
### **1. Control Plane (Managed by Microsoft)**
- **API Server**: Manages workloads.
- **etcd Database**: Stores Kubernetes configuration.
- **Scheduler**: Assigns new pods to nodes.
- **Controller Managers**:  
  - **Kube-Controller-Manager**: Maintains cluster state.  
  - **Cloud Controller Manager**: Manages Azure resources (nodes, routes, services).

### **2. Node Pools (Managed by You)**
- Made up of one or more **Azure VMs** in **VM Scale Sets**.
- Includes:
  - **Kubelet**: Ensures containers run in pods.
  - **Kube-Proxy**: Maintains network rules.
  - **Container Runtime**: Manages container lifecycles.
  - **Containers**:
    - **Workload Containers**: Your applications.
    - **System Containers**: Kubernetes management (e.g., DNS, monitoring).

---

## Resource Groups in AKS
- When creating an AKS cluster:
  - **Primary Resource Group**: For the cluster.
  - **Node/Infrastructure Resource Group**: Automatically created (modifiable name).

### Node/Infrastructure Resource Group Details:
- Contains resources like **VMs**, **networking**, and **load balancers**.
- Default name format: `mc_<resource group>_<cluster name>_<location>`.
- **Important**:
  - Do not modify Azure-configured resources or tags in this group (e.g., AKS and K8s tags).
  - Custom tags can be added for governance or cost management.

### Lifecycle:
- The node/infrastructure resource group is deleted when the AKS cluster is deleted.