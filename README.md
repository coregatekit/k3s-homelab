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
