The diagram illustrates all major components of the **OpenShift architecture** and their relationships.
---



```mermaid
graph TD
    A[User/API Request] --> B[Ingress Controller]

    subgraph OpenShift Cluster
        B --> C[Control Plane Node]
        C --> D[kube-apiserver]
        C --> E[kube-controller-manager]
        C --> F[kube-scheduler]
        C --> G[etcd (Key-Value Store)]

        D --> H[Admission Controllers]
        H --> I[RBAC, Quotas, etc.]

        D --> J[Operators (CRDs)]

        D --> K[Compute Nodes]
        K --> L[Pods]
        L --> M[kubelet]
        L --> N[CRI-O (Container Runtime)]
        L --> O[CNI Plugins (SDN/OVN-K)]

        K --> P[Persistent Volume Claim]
        P --> Q[Persistent Volume]

        subgraph Infrastructure Services
            R[Monitoring (Prometheus, Alertmanager)]
            S[Logging (Fluentd, Loki, Elasticsearch)]
            T[Image Registry]
            U[GitOps (ArgoCD)]
        end

        C --> R
        C --> S
        C --> T
        C --> U
    end

    T --> V[Container Images]
    U --> J

    style G fill:#f9f,stroke:#333,stroke-width:1px
    style C fill:#ccf,stroke:#333,stroke-width:1px
    style K fill:#cfc,stroke:#333,stroke-width:1px
    style InfrastructureServices fill:#ffe,stroke:#333,stroke-width:1px
```




---

###  Highlights of the Architecture

* **Control Plane** (API server, scheduler, controller manager, etcd)
* **Compute Nodes** (Pods, kubelet, container runtime, CNI)
* **Infrastructure Services** (Monitoring, Logging, Registry, GitOps)
* **User interaction** via `Ingress Controller` → `API Server`
* **Operators** manage CRDs and automate applications
* **Persistent storage** mapped via PVCs/PVs

---


