# Analysis: poor-mans-drs vs. Right-Sizing for OpenShift Virtualization

These two projects solve **fundamentally different problems** in the OpenShift Virtualization resource management space, despite both using ACM and dealing with VM resources.

---

## What poor-mans-drs Does

[`ch-stark/poor-mans-drs`](https://github.com/ch-stark/poor-mans-drs) is an **ACM policy that emulates VMware DRS (Distributed Resource Scheduler)**. It focuses on **workload placement** -- moving VMs away from overloaded nodes in real time.

### Workflow

1. Iterates worker nodes, collects allocatable CPU/memory and current usage
2. Flags nodes exceeding a threshold (default: 50%)
3. Stores metrics in a ConfigMap (`drs-worker-node-name-info` in `openshift-cnv`)
4. Identifies VMs on overloaded nodes
5. Checks for existing active migrations (avoids duplicates)
6. Creates `VirtualMachineInstanceMigration` (VMIM) objects to live-migrate VMs off hot nodes

### Key Characteristics

- **Reactive / operational** -- responds to current node pressure
- **Takes action** -- automatically live-migrates VMs
- Implemented as ACM policy with shell scripting
- Configurable thresholds and evaluation intervals
- VM opt-out via `acm-drs/exclude` label
- Tested on OCP 4.15 / ACM 2.11

---

## What Right-Sizing for Virt Does

Right-Sizing for OpenShift Virtualization (GA in ACM 2.16) is an **analytical/advisory tool** that tells you whether your VMs are requesting the right amount of CPU and memory.

### Workflow

1. Collects KubeVirt metrics via Prometheus
2. Applies Prometheus recording rules over long-term historical data (15/30 day windows)
3. Calculates utilization percentage relative to requested resources
4. Surfaces recommendations through dashboards at cluster, namespace, and per-VM levels
5. Highlights overestimation (too many resources requested) and underestimation (too few)

### Key Characteristics

- **Proactive / analytical** -- identifies chronic misconfigurations
- **Advisory only** -- generates recommendations, does not take action
- Uses Thanos-based aggregation for multi-cluster fleet-wide views
- Complements VPA (VPA is the "execution layer"; right-sizing is the "analytical layer")
- GA in ACM 2.16 (March 2026)

---

## Key Differences

| Dimension | poor-mans-drs | Right-Sizing for Virt |
|---|---|---|
| **Problem solved** | Node imbalance / hotspots | VM resource mis-allocation |
| **Question answered** | "Which node should this VM run on?" | "How much CPU/memory should this VM request?" |
| **Action** | Automated live migration (VMIM) | Recommendations only (dashboards) |
| **Trigger** | Real-time node utilization threshold | Long-term historical usage analysis |
| **Scope** | Single cluster | Multi-cluster fleet (via Thanos) |
| **VMware analogy** | **DRS** (Distributed Resource Scheduler) | **vROps right-sizing recommendations** |
| **Metrics focus** | Node-level (host CPU/mem usage) | VM-level (guest CPU/mem vs. requests) |
| **Maturity** | Community / PoC (ACM policy) | GA product feature (ACM 2.16) |
| **Time horizon** | Instantaneous (current load) | Historical trends (15-30 days) |

---

## How They Complement Each Other

1. **Right-Sizing** ensures each VM is configured with appropriate resource requests.
2. **poor-mans-drs** ensures VMs are distributed across nodes to avoid hotspots.

A well-optimized environment needs both. Right-sizing reduces the *need* for DRS-style rebalancing, but doesn't eliminate it.

The **Kubernetes Descheduler** with `LongLifecycle` / `DevKubeVirtRelieveAndMigrate` profiles is the more official path toward DRS-like behavior.

---

## Recommended Next Steps for poor-mans-drs

1. **Integrate Right-Sizing data** -- consume Prometheus recording rules for smarter migration target selection
2. **Evaluate Descheduler parity** -- benchmark against `DevKubeVirtRelieveAndMigrate` profile
3. **Multi-cluster awareness** -- leverage ACM for cross-cluster migration decisions (differentiator vs descheduler)
4. **Modernize** -- move from shell-in-policy to controller/Ansible, update tested versions to OCP 4.17+ / ACM 2.16
5. **Clarify positioning** -- document relationship to Right-Sizing (GA) and Descheduler

## References

- [poor-mans-drs repository](https://github.com/ch-stark/poor-mans-drs)
- [Announcing right-sizing for OpenShift Virtualization](https://developers.redhat.com/articles/2025/04/28/announcing-right-sizing-openshift-virtualization)
- [Right-sizing recommendations for OpenShift Virtualization](https://developers.redhat.com/articles/2025/12/05/right-sizing-recommendations-openshift-virtualization)
- [ACM 2.16 right-sizing recommendation GA](https://developers.redhat.com/articles/2026/03/17/advanced-cluster-management-216-right-sizing-recommendation-ga)
- [Simulating VMware DRS for OpenShift Virtualization](https://linuxelite.com.br/blog/drs-for-openshift-virtualization/)
- [OpenShift Virtualization and the Kubernetes Descheduler](https://xphyr.net/post/openshift_virt_and_k8s_descheduler/)
