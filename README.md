Here is the reformatted content in a clean, structured Markdown format optimized for a GitHub repository or documentation page.

---

# KarpenterConfiguration

This repository contains the configuration and definitions for **Karpenter**, a high-performance Kubernetes node autoscaling solution. It defines how the cluster provisions nodes in response to unschedulable pods.

## 📖 Overview

KarpenterConfiguration manages the two primary Custom Resource Definitions (CRDs) required to scale compute:

* **NodePool**: Defines constraints on the nodes that can be created (e.g., instance types, zones, architecture) and pod scheduling requirements.
* **EC2NodeClass (AWS specific)**: Defines the infrastructure configuration for the nodes (e.g., AMIs, Security Groups, Subnets, User Data).

## 🚀 Prerequisites

* Kubernetes Cluster (v1.25+)
* Karpenter Controller installed in the cluster.
* `kubectl` configured to communicate with your cluster.
* Cloud Provider permissions (e.g., AWS IAM Roles for Service Accounts).

## 📂 Repository Structure

```text
.
├── nodepools/              # Definitions for scheduling constraints
│   ├── default.yaml        # General purpose compute (Spot/On-Demand)
│   ├── gpu-workload.yaml   # GPU specific configuration
│   └── system.yaml         # Critical system components
├── nodeclasses/            # Cloud provider infrastructure specs
│   ├── default-al2.yaml    # Amazon Linux 2 configuration
│   └── bottlerocket.yaml   # Bottlerocket OS configuration
└── values.yaml             # Helm values (if applicable)

```

## 🛠 Usage

### 1. Apply a NodeClass

First, define the infrastructure (Security Groups, Subnets, IAM Roles).

```bash
kubectl apply -f nodeclasses/default-al2.yaml

```

### 2. Apply a NodePool

Next, define the scheduling logic. This references the NodeClass applied above.

```bash
kubectl apply -f nodepools/default.yaml

```

---

## ⚙️ Configuration Reference

### NodePool Configuration (`nodepools/`)

The NodePool controls which instances Karpenter selects.

| Parameter | Description | Recommended Value |
| --- | --- | --- |
| **requirements** | Key-value pairs for instance selection (Arch, OS, Type). | `karpenter.sh/capacity-type`: `["spot", "on-demand"]` |
| **limits** | Hard cap on total resources provisioned by this pool. | `CPU: 1000`, `Memory: 1000Gi` |
| **disruption** | Controls how nodes are de-provisioned (consolidation). | `consolidationPolicy: WhenUnderutilized` |
| **weight** | Priority of the NodePool (higher = higher priority). | `10` for specialized, `1` for general. |

### EC2NodeClass Configuration (`nodeclasses/`)

The EC2NodeClass controls how instances are configured in AWS.

| Parameter | Description |
| --- | --- |
| **amiFamily** | The OS family (AL2, AL2023, Bottlerocket, Ubuntu). |
| **subnetSelectorTerms** | Tags used to discover subnets for node placement. |
| **securityGroupSelectorTerms** | Tags used to discover security groups. |
| **role** | The IAM instance profile role name for the nodes. |

---

## 📝 Examples

### General Purpose Spot Instances

Use this for stateless web applications and background workers.

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot"]
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["c", "m", "r"]
        - key: karpenter.k8s.aws/instance-generation
          operator: Gt
          values: ["2"]
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
  limits:
    cpu: 1000
  disruption:
    consolidationPolicy: WhenUnderutilized
    expireAfter: 720h

```

### GPU Workload (On-Demand)

Use this for AI/ML training or inference.

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: gpu-workload
spec:
  template:
    spec:
      taints:
        - key: nvidia.com/gpu
          value: "true"
          effect: NoSchedule
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand"]
        - key: karpenter.k8s.aws/instance-type
          operator: In
          values: ["g4dn.xlarge", "g5.xlarge"]
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: gpu-optimized

```

---

## 🔍 Troubleshooting

* **Nodes are not scaling up:**
* Check Karpenter logs: `kubectl logs -l app.kubernetes.io/name=karpenter -n karpenter`
* Ensure the NodePool requirements match your Pod's `nodeSelector` or `affinity`.
* Verify EC2NodeClass subnet and security group tags match actual AWS resources.


* **Nodes are not consolidating (scaling down):**
* Check if pods have `karpenter.sh/do-not-disrupt: "true"` annotation.
* Ensure `disruption.consolidationPolicy` is set to `WhenUnderutilized`.
