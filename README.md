## Install MicroK8s Manually

```bash
# Install MicroK8s
sudo snap install microk8s --classic

# Wait until MicroK8s is ready
microk8s status --wait-ready

# Refresh group membership
newgrp microk8s

# Verify status
microk8s status

# Create kubectl alias
sudo snap alias microk8s.kubectl kubectl

# Check cluster nodes
kubectl get nodes

# Enable DNS, storage, and ingress
microk8s enable dns storage
microk8s enable ingress
```

### You can run script  to install microk8s from below repo as well 

## Clone Repo and Run Script

```bash
git clone https://github.com/rootpromptnext/ingress-blue-green.git
cd ingress-blue-green/
```

Run the provided install script:

```bash
bash microk8s-install/microk8s-install.sh
```

Script contents:

```bash
#!/bin/bash
set -e

echo "=== Installing MicroK8s ==="
sudo snap install microk8s --classic

echo "=== Waiting for MicroK8s to be ready ==="
microk8s status --wait-ready

echo "=== Adding current user to microk8s group ==="
sudo usermod -a -G microk8s $USER
sudo chown -f -R $USER ~/.kube

echo "=== Creating kubectl alias ==="
sudo snap alias microk8s.kubectl kubectl

echo "=== Enabling DNS and storage add-ons ==="
microk8s enable dns storage

echo "=== Enabling ingress controller ==="
microk8s enable ingress

echo "=== Setup complete! ==="
echo "IMPORTANT: Please log out and log back in (or run 'newgrp microk8s') to refresh your group membership before using kubectl."
```

## Expose Ingress via NodePort

By default, MicroK8s ingress doesn’t create a Service. Add one manually:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-ingress-microk8s-controller
  namespace: ingress
spec:
  type: NodePort
  selector:
    name: nginx-ingress-microk8s
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
    protocol: TCP
```

Apply it:

```bash
kubectl apply -f ingress-service.yaml
kubectl -n ingress get all
```
### Add Host Entries
For Canary:
```bash
echo "10.10.0.2 demo.local" | sudo tee -a /etc/hosts
```

For Blue‑Green testing:
```bash
echo "10.10.0.2 blue.local" | sudo tee -a /etc/hosts
echo "10.10.0.2 green.local" | sudo tee -a /etc/hosts
```
---

#### Blue‑Green
- `blue-deployment.yaml` → “Hello from BLUE”.  
- `green-deployment.yaml` → “Hello from GREEN”.  
- `blue-ingress.yaml` → routes `blue.local`.  
- `green-ingress.yaml` → routes `green.local`.

Test:
```bash
curl http://blue.local:30080
curl http://green.local:30080
```
---
### Switch Traffic (Blue‑Green)
When ready to promote green:
- Option A: Update service selector to `version: green`.  
- Option B: Change `/etc/hosts` so `demo.local` points to green ingress.  

Rollback is instant: switch back to blue.
