# K3S Setup

## Step 1: Prepare the Environment

```bash
sudo apt update
sudo apt install -y curl open-iscsi nfs-common
sudo systemctl enable open-iscsi
```

Open kernel modules:

```bash
sudo modprobe br_netfilter
sudo modprobe overlay
```

Set up sysctl:

```bash
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system
```

## Step 2: Install Tailscale

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

Login to Tailscale:

```bash
sudo tailscale up
```

Check Tailscale IP:

```bash
tailscale ip -4
```

You should see an IP address in the range `100.x.y.z`. This will be used as the TLS SAN for K3S.

Install Tailscale on your laptop as well and ensure both the laptop and the K3S node are connected to the same Tailscale network. You can verify connectivity by pinging the Tailscale IPs from each device.

## Step 3: Install K3S

```bash
#!/bin/bash

NODE_IP="10.0.0.2"
TAILSCALE_IP="100.101.102.103"

export INSTALL_K3S_EXEC="
server
--node-ip=${NODE_IP}
--advertise-address=${NODE_IP}
--tls-san=${TAILSCALE_IP}
--write-kubeconfig-mode=644
"

curl -sfL https://get.k3s.io | sh -
```

Copy kubeconfig to your laptop:

```bash
scp user@k3s-node:/etc/rancher/k3s/k3s.yaml ~/.kube/config
```

Update the kubeconfig to use the Tailscale IP:

```yaml
server: https://<TAILSCALE_IP>:6443
```

Verify the cluster is up:

```bash
kubectl get nodes
```
