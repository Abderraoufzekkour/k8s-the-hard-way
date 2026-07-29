# ☸️ Kubernetes The Hard Way — OpenStack Bare-Metal

> Manual, installer-free Kubernetes cluster deployment on real OpenStack infrastructure — built to understand every layer of the stack.

![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.x-326CE5?style=flat-square&logo=kubernetes&logoColor=white)
![OpenStack](https://img.shields.io/badge/OpenStack-Bare--Metal-ED1944?style=flat-square&logo=openstack&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04_LTS-E95420?style=flat-square&logo=ubuntu&logoColor=white)
![containerd](https://img.shields.io/badge/Runtime-containerd-gray?style=flat-square)
![Flannel](https://img.shields.io/badge/CNI-Flannel_VXLAN-purple?style=flat-square)

---

## 🎯 Project Overview

This project follows Kelsey Hightower's **"Kubernetes The Hard Way"** methodology, adapted for a real **OpenStack bare-metal** environment with a **single control plane architecture**. Every component is manually provisioned — no kubeadm, no automation, no shortcuts.

The goal is full exposure to the internals of a Kubernetes cluster:

| What | Why it matters |
|---|---|
| TLS bootstrapping & mTLS | Understand how components authenticate each other |
| Manual `etcd` setup | See the consensus layer that backs the entire control plane |
| `containerd` + `runc` config | Know what actually runs your containers |
| Flannel VXLAN overlay | Understand pod-to-pod networking across nodes |
| CoreDNS deployment | See how service discovery works from the ground up |

> This is not a tutorial clone — it's a **real deployment on OpenStack infrastructure** with documented failures, root-cause analysis, and production-relevant fixes.

---

## 🏗️ Architecture

```
                    ┌──────────────────┐
                    │   p1-bastion     │
                    │  (admin access,  │
                    │  TLS gen, kubectl)│
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────┐
                    │ kthw-controller-1│
                    │                  │
                    │  • etcd          │
                    │  • kube-apiserver│
                    │  • kube-ctrl-mgr │
                    │  • kube-scheduler│
                    └────────┬─────────┘
                             │
               ┌─────────────┴─────────────┐
               │                           │
      ┌────────▼────────┐       ┌──────────▼──────┐
      │  kthw-worker-1  │       │  kthw-worker-2  │
      │                 │       │                 │
      │  • kubelet      │       │  • kubelet      │
      │  • kube-proxy   │       │  • kube-proxy   │
      │  • containerd   │       │  • containerd   │
      └─────────────────┘       └─────────────────┘
```

### Component Stack

| Layer | Component |
|---|---|
| Cloud Infrastructure | OpenStack Bare-Metal — Ubuntu 22.04 LTS |
| Container Runtime | `containerd` + `runc` |
| Networking (CNI) | Flannel (VXLAN backend) |
| DNS | CoreDNS |
| Control Plane | `etcd`, `kube-apiserver`, `kube-controller-manager`, `kube-scheduler` |
| Worker Nodes | `kubelet`, `kube-proxy` |

---

## 📂 Repository Structure

```text
kthw/
├── 📁 configs/                        # Systemd unit files & component configs
│   ├── containerd.toml                # containerd runtime config (systemd cgroup driver)
│   ├── kube-apiserver.service         # API server systemd unit
│   ├── kube-controller-manager.service
│   ├── kube-proxy-config.yaml
│   ├── kube-scheduler.yaml
│   ├── kubelet-config.yaml
│   └── kubelet.service
├── 📁 manifests/                      # Kubernetes YAML manifests
│   ├── coredns.yaml                   # CoreDNS deployment + ClusterIP service
│   ├── flannel.yaml                   # Flannel CNI DaemonSet
│   └── metrics-server.yaml
├── 📁 tls/                            # Certificate generation configs (cfssl)
│   ├── ca-config.json                 # CA signing profile
│   ├── ca-csr.json                    # CA Certificate Signing Request
│   ├── etcd-csr.json                  # etcd peer/server cert config
│   └── kube-csr.json                  # Component CSRs (apiserver, kubelet, proxy, admin)
└── 📁 screenshots/                    # Live cluster verification output
    ├── cluster-nodes.png
    ├── cluster-pods-all.png
    └── dns-resolution-test.png
```

---

## 🚀 Deployment Guide

### Step 1 — Environment Setup

Provision OpenStack bare-metal instances running **Ubuntu 22.04 LTS**:

| Host | Role |
|---|---|
| `p1-bastion` | Admin access, TLS generation, `kubectl` |
| `kthw-controller-1` | Single control plane node |
| `kthw-worker-1` | Worker node |
| `kthw-worker-2` | Worker node |

Install required tooling on the bastion:

```bash
sudo apt-get update && sudo apt-get install -y wget curl vim jq
# Also required: cfssl, cfssljson, kubectl
```

---

### Step 2 — TLS Certificate Authority & Certificates

Generate a CA on the bastion and sign certificates for all cluster components:

| Certificate | Used by |
|---|---|
| CA | Root of trust for all components |
| `etcd` | Peer and server TLS |
| `kube-apiserver` | Server TLS (SAN includes controller IP) |
| `kubelet` (per node) | Client auth to API server |
| `kube-proxy` | Client auth |
| `admin` | kubeconfig for cluster admin |

All configs in [`tls/`](./tls/).

---

### Step 3 — Control Plane Bootstrap (`kthw-controller-1`)

**3.1 — etcd**

Install and configure `etcd` for secure key-value storage backing the API server.

**3.2 — Kubernetes Control Plane Binaries**

Download `kube-apiserver`, `kube-controller-manager`, and `kube-scheduler` into `/usr/local/bin/`.

**3.3 — Systemd Services**

Place unit files from [`configs/`](./configs/) into `/etc/systemd/system/` and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
```

**3.4 — RBAC**

Apply RBAC rules to authorize `kubelet` API calls from the API server.

---

### Step 4 — Worker Node Bootstrap (`worker-1` & `worker-2`)

**4.1 — Container Runtime**

Install `containerd` and configure the systemd cgroup driver:

```toml
# configs/containerd.toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
  SystemdCgroup = true
```

**4.2 — Kubernetes Worker Binaries**

Install `kubelet` and `kube-proxy` into `/usr/local/bin/`.

**4.3 — Start Services**

```bash
sudo systemctl enable containerd kubelet kube-proxy
sudo systemctl start containerd kubelet kube-proxy
```

---

### Step 5 — Cluster Add-ons

Apply from the bastion:

```bash
kubectl apply -f manifests/flannel.yaml        # Pod overlay network
kubectl apply -f manifests/coredns.yaml        # Cluster DNS
kubectl apply -f manifests/metrics-server.yaml # Resource metrics
```

---

## ✅ Cluster Verification

| Check | Command | Expected |
|---|---|---|
| Node status | `kubectl get nodes -o wide` | All nodes `Ready` |
| All pods | `kubectl get pods -A` | All pods `Running` |
| DNS resolution | `kubectl exec -it <pod> -- nslookup kubernetes` | Resolves to ClusterIP |

Screenshots of a live, verified cluster are in [`screenshots/`](./screenshots/).

---

## 🔧 Troubleshooting & Lessons Learned

> Bare-metal Kubernetes exposes failure domains that automated installers silently paper over. Below are real issues encountered during this deployment.

---

### 🐛 Container Runtime Sandbox Failures

**Symptom:** Pods stuck in `ContainerCreating`, `runc` processes hung in uninterruptible sleep.

**Root Cause:** Stale process namespaces from a previous failed `containerd` run blocking new sandbox creation.

**Fix:**
```bash
systemctl stop containerd
rm -rf /run/containerd/*
systemctl start containerd
```

---

### 🌐 Cascading CNI / Flannel Failures

**Symptom:** `kube-flannel` pod in `CrashLoopBackOff`, all pods unable to establish networking.

**Root Cause:** Corrupted CNI state directories from a partial previous initialization.

**Fix:**
```bash
rm -rf /run/flannel /etc/cni/net.d
systemctl restart kubelet
```

---

### 🕐 NTP Time Drift → `401 Unauthorized`

**Symptom:** CoreDNS returning `401 Unauthorized` from `kube-apiserver` despite correct RBAC config.

**Root Cause:** Clock skew between the controller and worker nodes caused JWT ServiceAccount token timestamp validation to fail at the API server.

**Fix:**
```bash
systemctl restart systemd-timesyncd
timedatectl status   # verify sync
```

> ⚠️ **Lesson:** Always verify NTP sync across **all nodes** before bootstrapping. Clock skew produces misleading auth errors that are easy to misdiagnose as RBAC misconfigurations.

---

## 📎 References

- [Kubernetes The Hard Way — Kelsey Hightower](https://github.com/kelseyhightower/kubernetes-the-hard-way)
- [OpenStack Documentation](https://docs.openstack.org/)
- [containerd](https://containerd.io/) / [runc](https://github.com/opencontainers/runc)
- [Flannel CNI](https://github.com/flannel-io/flannel)
- [cfssl — CloudFlare PKI Toolkit](https://github.com/cloudflare/cfssl)
- [CoreDNS](https://coredns.io/)
