# K3S Home lab

This repository contains the configuration files and documentation for setting up a K3S home lab environment.

## Prerequisites

- A machine or virtual machine with at least 4GB of RAM and 2 CPU cores

## Architecture

```text
Your Laptop
     ↓ (Tailscale private network)
VM (k3s control-plane)
     ↓
Kubernetes API :6443
```

We will allow only Tailscale IPs to access the VM. So we will access to cluster via Tailscale private network.

## Installation

1. Follow the instructions in the [K3S Setup](k3s-setup.md) to install K3S on your VM.
2. Follow the instructions in the [Cloudflare Setup](ingress-setup.md) to set up Tailscale on your VM and laptop.
3. Follow the instructions in the [Monitoring Setup](monitoring-setup.md) to set up monitoring on your cluster.
4. Follow the instructions in the [ArgoCD Setup](gitops-setup.md) to set up ArgoCD on your cluster.
5. Follow the instructions in the [Vault Setup](vault-setup.md) to set up Vault on your cluster.
6. Follow the instructions in the [CNPG Setup](cnpg-setup.md) to set up CNPG on your cluster.
