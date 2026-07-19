# Kubernetes The Hard Way on OpenStack

A production-grade implementation of Kubernetes The Hard Way (KTHW) deployed on bare-metal OpenStack infrastructure.

## Repository Structure
* **`pki/`**: Certificate Authority, TLS certificates, and CSR configurations. *(Note: Private keys are excluded via .gitignore for security).*
* **`kubeconfigs/`**: Client authentication files for the Kubernetes components and workers.
* **`configs/`**: Data encryption configurations for etcd secrets at rest.
