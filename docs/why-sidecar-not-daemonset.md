# Why sidecar, not DaemonSet?

a common Kubernetes logging pattern installs a log collector (Fluent Bit, Fluentd, Filebeat) as a **DaemonSet** — one agent pod on every node, collecting logs from all containers on that node.

now lets uses a **sidecar** instead: Fluent Bit runs inside **your** pod,

 next to **your** app,
 
 and only collects **your** app's logs.



## How 


### DaemonSet agent

```
┌─────────────────────────────────────────────┐
│ Node                                        │
│  ┌──────────────┐  ┌──────────────┐         │
│  │ Pod A        │  │ Pod B        │         │
│  │  [app]       │  │  [app]       │         │
│  └──────────────┘  └──────────────┘         │
│         │                  │                │
│         └────────┬─────────┘                │
│                  ▼                          │
│  /var/log/containers/*.log                  │
│                  │                          │
│                  ▼                          │
│  ┌──────────────────────────────┐           │
│  │ fluent-bit DaemonSet pod     │  ← cluster-wide
│  └──────────────────────────────┘           │
└─────────────────────────────────────────────┘
                    │
                    ▼
              Elasticsearch
```

- One agent per node, installed by a platform team or Helm chart.
- Collects logs from **every** pod on the node.
- Requires **ClusterRole permissions**.

### Sidecar agent (this repo)

```
┌──────────────────────────────┐
│ Pod (your Deployment)        │
│  ┌─────────┐  ┌────────────┐ │
│  │ your-app│  │ fluent-bit │ │  ← per-pod, opt-in
│  └────┬────┘  └─────┬──────┘ │
│       │ stdout       │ tail  │
└───────┼──────────────┼───────┘
        │              │
        ▼              ▼
  /var/log/containers/*_your-ns_your-app-*.log
                              │
                              ▼
                        Elasticsearch
```

- Fluent Bit is part of **your** Deployment manifest.
- Only tails log files matching **your** container name and namespace.
- No cluster-level install. No DaemonSet RBAC.



## Comparison

| Dimension | Sidecar | DaemonSet |
|-----------|---------|-----------|
| **Install scope** | Per Deployment — you opt in | Cluster-wide — platform installs once |
| **RBAC** | Namespace-level (Deployment, ConfigMap, Secret) | ClusterRole to read all pods/nodes |
| **Log isolation** | Only your app's log files (via Path glob) | All pods on the node |
| **Resource cost** | One Fluent Bit instance per **your** pod replica | One Fluent Bit per **node** (amortized) |




## When a sidecar is the better fit ?

Choose the sidecar pattern when:

- You **cannot** install a DaemonSet ---> managed Kubernetes, GKE Autopilot-style restrictions, corporate policy
- You want **team-owned** logging — each Deployment ships to its own ES index without waiting on a platform team
- You are adding logging to **one or a few** Deployments, not the entire cluster



