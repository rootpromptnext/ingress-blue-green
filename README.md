### Install MicroK8s

Either run the commands manually:
```bash
sudo snap install microk8s --classic
microk8s status --wait-ready
sudo usermod -a -G microk8s $USER
sudo chown -f -R $USER ~/.kube
newgrp microk8s
microk8s status
sudo snap alias microk8s.kubectl kubectl
kubectl get nodes
microk8s enable dns storage
microk8s enable ingress
```

Or clone your repo and run the script:
```bash
git clone https://github.com/rootpromptnext/ingress-blue-green.git
cd ingress-blue-green/
bash microk8s-install/microk8s-install.sh
```

### Expose Ingress via NodePort
Create and apply `ingress-service.yaml`:
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

```bash
kubectl apply -f ingress-service.yaml
kubectl -n ingress get all
```
---
### Add Host Entries
Map your node IP to hostnames for testing:
```bash
echo "10.10.0.2 blue.local" | sudo tee -a /etc/hosts
echo "10.10.0.2 green.local" | sudo tee -a /etc/hosts
```
---
### Deploy Blue and Green
Apply manifests:
- `blue-deployment.yaml` → “Hello from BLUE”  
- `green-deployment.yaml` → “Hello from GREEN”  
- `blue-ingress.yaml` → routes `blue.local`  
- `green-ingress.yaml` → routes `green.local`

```bash
kubectl apply -f blue-deployment.yaml
kubectl apply -f green-deployment.yaml
kubectl apply -f blue-ingress.yaml
kubectl apply -f green-ingress.yaml
```

### Test Both Environments
```bash
curl http://blue.local:30080
curl http://green.local:30080
```
You’ll see responses from each environment separately.

### Switch Traffic
When ready to promote **Green**:
- **Option A:** Update `demo-service` selector to `version: green`.  
- **Option B:** Change `/etc/hosts` so `demo.local` points to green ingress.  

Rollback is instant: switch back to **Blue**.
  

Do you want me to also add a **rollback demo section** (like we did for Canary) with actual `kubectl` commands and expected curl outputs, so your README shows the full lifecycle?
