
flowchart TD
    %% Global Styling & Classes
    classDef infra fill:#E0F2FE,stroke:#0284C7,stroke-width:2px,color:#0B132B;
    classDef cyber fill:#FEE2E2,stroke:#DC2626,stroke-width:2px,color:#7F1D1D;
    classDef devops fill:#DCFCE7,stroke:#16A34A,stroke-width:2px,color:#14532D;
    classDef shared fill:#F3E8FF,stroke:#9333EA,stroke-width:2px,color:#581C87;
    classDef external fill:#F1F5F9,stroke:#64748B,stroke-width:2px,stroke-dasharray: 4 4,color:#334155;

    %% ZONE 0: FLEET & PLATFORM SERVICES
    subgraph Zone0 ["ZONE 0: PLATFORM SERVICES & GOVERNANCE (Shared Management Plane)"]
        TMC["Tanzu Mission Control (TMC)<br/><i>Fleet Policies & Cluster LCM</i>"]:::shared
        TBS["Tanzu Build Service (TBS)<br/><i>Kpack / OCI Rebase</i>"]:::devops
        Harbor["Harbor Registry + Trivy<br/><i>Vulnerability Scanning & CVE Gates</i>"]:::cyber
        IdP["Enterprise Identity (IdP)<br/><i>OIDC / Dex / Active Directory</i>"]:::cyber
        GitOps["GitOps Engine<br/><i>Flux CD / ArgoCD Repos</i>"]:::devops
        Vault["HashiCorp Vault<br/><i>Secrets Engine</i>"]:::cyber
    end

    %% ZONE 1: vSPHERE SUPERVISOR CLUSTER
    subgraph Zone1 ["ZONE 1: vSPHERE SUPERVISOR INFRASTRUCTURE LAYER (ESXi / vCenter)"]
        Supervisor["Supervisor Control Plane VMs<br/><i>Spherelet Engine</i>"]:::infra
        CAPI["Cluster API (CAPI) & vSphere CPI<br/><i>VM Lifecycle & Node Scaling</i>"]:::infra
        CNS_Ctrl["vSphere CSI Controller<br/><i>Storage Control Plane</i>"]:::infra
        Avi_Ctrl["NSX ALB (Avi) Controller<br/><i>VIP Allocation, IPAM & WAF Engine</i>"]:::infra
    end

    %% ZONE 2: TKG WORKLOAD CLUSTER CONTROL PLANE
    subgraph Zone2 ["ZONE 2: TKG WORKLOAD CLUSTER CONTROL PLANE (Master VMs)"]
        KubeAPI["kube-apiserver<br/><i>REST API Gateway</i>"]:::infra
        etcd["etcd Cluster<br/><i>Distributed State Store</i>"]:::infra
        Pinniped["Pinniped Concierge<br/><i>Auth Webhook & Token Exchange</i>"]:::cyber
        OPA["OPA Gatekeeper / Admission Webhook<br/><i>Policy Enforcement & Compliance</i>"]:::cyber
    end

    %% ZONE 3: TKG WORKLOAD CLUSTER WORKER NODES
    subgraph Zone3 ["ZONE 3: TKG WORKLOAD CLUSTER WORKER NODES (Execution Data Plane)"]
        subgraph SubZone3A ["3A. Host OS & Node DaemonSets (Root Level)"]
            Kubelet["kubelet + containerd"]:::infra
            Antrea["Antrea CNI Agent<br/><i>Open vSwitch (OVS) Engine</i>"]:::infra
            CSI_Node["vSphere CSI Node Driver<br/><i>Mounts Block/File Volumes</i>"]:::infra
            eBPF["eBPF Security Sensor<br/><i>Falco / Tanzu Guard Sensor</i>"]:::cyber
        end

        subgraph SubZone3B ["3B. Ingress & Cluster Edge (tanzu-system-ingress)"]
            Contour["Contour Ingress Controller + Envoy<br/><i>L7 Routing, TLS Termination & HTTPProxy</i>"]:::shared
        end

        subgraph SubZone3C ["3C. Tenant / Project Namespaces (Application Sandbox)"]
            AppPods["Application Pods<br/><i>Business Logic Containers</i>"]:::devops
            ServiceMesh["Tanzu Service Mesh<br/><i>Envoy Sidecar Proxies</i>"]:::devops
            Services["ClusterIP Services & CoreDNS"]:::devops
            PVC["PersistentVolumeClaims (PVC)"]:::devops
            NetPol["Antrea NetworkPolicies & PSA"]:::cyber
        end
    end

    %% ZONE 4: STORAGE & NETWORK PERIMETER
    subgraph Zone4 ["ZONE 4: PERSISTENT STORAGE & EXTERNAL EDGE (Physical Fabric)"]
        vSAN["VMware vSAN / VMFS<br/><i>First Class Disks (FCD Block Storage)</i>"]:::infra
        vSAN_File["vSAN File Services / MinIO<br/><i>NFS (RWX) & S3 Object Storage</i>"]:::infra
        Avi_SE["NSX ALB Service Engines (SE)<br/><i>L4-L7 VIPs, SSL Offload & WAF</i>"]:::infra
        Client["External Users & API Consumers"]:::external
    end

    %% CONNECTIONS & FLOWS
    %% 1. Fleet & Management Lifecycle
    TMC -->|Fleet Manifests| GitOps
    GitOps -->|Provisions Cluster CRD| CAPI
    CAPI -->|Instantiates & Scales| Zone2
    CAPI -->|Instantiates VMs| Kubelet

    %% 2. Auth & Policy Enforcement
    IdP -.->|OIDC Tokens| Pinniped
    Pinniped -->|Validates User| KubeAPI
    OPA -->|Admission Gate Validation| KubeAPI

    %% 3. Ingress & Traffic Flow (North-South)
    Client ==>|External HTTPS Request| Avi_SE
    Avi_SE ==>|Routed Inbound Traffic| Contour
    Contour -->|Routes via HTTPProxy| Services
    Services --> AppPods

    %% 4. Storage Provisioning Flow
    PVC -.->|Claims StorageClass| CSI_Node
    CSI_Node ==>|Delegates Attachment| CNS_Ctrl
    CNS_Ctrl ==>|Creates & Attaches FCD| vSAN
    vSAN -.->|Mounted to Worker VM| AppPods
    vSAN_File -.->|NFS/S3 Mount| AppPods

    %% 5. Security & Network Policies (East-West)
    NetPol -.->|Enforces Firewall Rules| Antrea
    Antrea ---|Micro-segmentation| AppPods
    eBPF -.->|Runtime Threat Telemetry| Harbor