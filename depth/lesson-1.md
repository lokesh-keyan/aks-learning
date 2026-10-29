# AKS request journey: key terms and notes

**Lesson 1 | October 29, 2026**

**Simple picture:** The customer places an order -> Azure checks their identity and permission -> the AKS frontend checks the order -> background workers carry it out.

## Key terms

| Term | What to remember |
|---|---|
| **Client** | Portal, CLI, PowerShell, SDK, or REST client. Different ways to submit Azure management requests. |
| **Microsoft Entra ID** | Issues access tokens that establish the caller's identity. |
| **Authentication (AuthN)** | **Who are you?** |
| **Authorization (AuthZ)** | **Are you allowed to perform this action on this resource?** Successful sign-in does not guarantee permission. |
| **Azure RBAC** | Role-based permissions applied at scopes such as subscription, resource group, and resource. |
| **ARM (Azure Resource Manager)** | Azure's management entry point. Authenticates, authorizes, and routes requests to the appropriate service. |
| **RP (Resource Provider)** | Implements management operations for an Azure resource family. AKS uses **`Microsoft.ContainerService`**. |
| **`managedClusters`** | The Azure resource type representing AKS clusters. |
| **API version** | Selects the request/response contract, not the cluster's Kubernetes version. |
| **Frontend / RP sync** | Handles the initial request: validation, goal-state preparation, asynchronous handoff, and response. |
| **Goal state / desired state** | What the customer wants the cluster to become, not necessarily what already exists. |
| **RP async** | Background processing that works toward the requested state. |
| **LRO (Long-running operation)** | Work that continues after the initial HTTP response; the client polls for progress. |

## Two paths you must not confuse

```text
Manage an AKS Azure resource:
Portal / CLI / SDK / REST -> ARM -> AKS RP request path

Manage Kubernetes objects:
kubectl -> the customer cluster's Kubernetes API server
```

**Creating a cluster and creating a Deployment inside it are different operations against different APIs.** Azure resource permissions and Kubernetes permissions are also different authorization surfaces.

The ARM-to-RP path above is conceptual, not an exhaustive diagram of gateways and platform intermediaries.

## Where the frontend runs

```text
Azure VMs / VM scale sets
  -> infrastructure-team-operated Kubernetes service underlay
    -> containerservice namespace (base configuration)
      -> frontend pods running /containerservice
```

- **SVC underlay:** hosts service-management components such as RP and HCP.
- **CX underlay:** hosts customer control planes.
- The standard underlay architecture uses **self-managed Kubernetes**, bootstrapped with **kubeadm**, rather than a customer-style AKS managed cluster. "Self-managed" means operated by the AKS infrastructure team, not customers. It does not mean bare metal.
- Frontend releases use **AKS Deployer** and an **Argo Rollout**; an **HPA (Horizontal Pod Autoscaler)** adjusts replicas.
- The frontend is **not on the customer's worker nodes** and is **not their Kubernetes API server**.
- These statements describe the inspected standard architecture, not a live inventory of every environment or migration target.

## Storage versus telemetry

| Item | Meaning |
|---|---|
| **Operational storage** | State that services use to drive operations: the business's order book. |
| **`ManagedCluster` / `AgentPoolResource`** | Internal data models. Their names alone do **not** establish physical table names. |
| **`SaveManagedClusterAndAP`** | A method that saves state, not a table. |
| **`CreateOperationStatusV2`** | A method that creates operation tracking, not a table. |
| **Operation queue** | Delivers work for asynchronous processing, not the cluster's database record. |
| **`FrontEndQoSEvents`** | Kusto telemetry describing frontend outcomes, codes, and latency. |
| **`FrontEndContextActivity`** | Kusto telemetry containing detailed frontend log messages. |
| **Snapshot telemetry** | An observed copy of state, not the authoritative live database. |

## Investigation fields

The inspected frontend query files target the **`AKSprod`** database.

| Field | Use |
|---|---|
| **`operationID`** | Pivot from a frontend outcome to that operation's detailed logs. |
| **`correlationID`** | Correlate related activity; not a guaranteed universal join key. |
| **`RPTenant`** | Scope frontend telemetry to the RP region/tenant. |
| **`PreciseTimeStamp`** | Filter the relevant UTC time window. |
| **`resultType`** | **0:** success; **1:** client error; **2:** service failure; **3:** client cancellation/timeout. This is not the HTTP status code. |
| **`resultCode` / `resultSubCode`** | Outcome and more specific classification. `resultCode` is a string; convert it before numeric comparisons. |

**Investigation sequence:** Scope by time and authorized resource -> inspect frontend QoS -> copy operation ID and RP tenant -> read frontend context logs -> follow the dependency indicated by evidence.

## Most important rules

1. **Accepted is not completed.** The inspected PUT create path can return **201 Created** while the resource is still `Creating`.
2. **Operation status is not resource provisioning state.** They describe related but different things.
3. **Frontend success does not mean successful cluster provisioning.**
4. **Client timeout does not mean the background operation was canceled.** Check operation status before retrying.
5. **No log rows do not prove the request never arrived.** Check filters, retention, access, and telemetry coverage.
6. **A frontend error may originate in a dependency**, such as HCP, queueing, capacity, or another Azure service.
7. **Source configuration is not verified live deployment.** Manifests describe intended behavior; live evidence establishes what is running.

## Source pointers

These are notes from the source-backed lesson and deployment supplement, not results of a live production investigation.

- [Azure Resource Manager overview](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/overview)
- [Tracking asynchronous Azure operations](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/async-operations)
- [AKS Service Overview wiki](https://msazure.visualstudio.com/CloudNativeCompute/_wiki/wikis/CloudNativeCompute.wiki/16868/AKS-Service-Overview)
- [Underlay types, pinned infrastructure source](https://dev.azure.com/msazure/CloudNativeCompute/_git/aksiknife?path=%2Fdocs%2Funderlays%2Funderlay-types.md&version=GC7076f0859c73b55539078c9cd5a18c042d463864)
- [Underlay bootstrap code, lines 172-177](https://dev.azure.com/msazure/CloudNativeCompute/_git/aksiknife?path=%2Fpkg%2Fnodeagent%2Fcmd%2Fk8s%2Fbootstrap_control_plane.go&version=GC7076f0859c73b55539078c9cd5a18c042d463864&line=172&lineEnd=178&lineStartColumn=1&lineEndColumn=1)
- aks-rp: `resourceprovider\server\microsoft.com\containerservice\server\operations\managedcluster\putmanagedcluster\putmanagedcluster.go:785-818` for the PUT response; `resourceprovider\server\deployers\api-deployer\templates\base\rp-api-rollout.yaml` for workload deployment.
- aks-skills: `experimental\skills\aks-incident-troubleshoot\autonomous-incident-investigation\kql\resource-provider\frontend\frontend-qos-events-sampled.kql` and `frontend-context-activity.kql` for frontend telemetry.
