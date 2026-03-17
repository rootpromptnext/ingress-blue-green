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
### Add Host Entries
Map your node IP to hostnames for testing:
```bash
echo "10.10.0.2 blue.local" | sudo tee -a /etc/hosts
echo "10.10.0.2 green.local" | sudo tee -a /etc/hosts
```
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
  
### Option A: Update `demo-service` Selector

### Initial `demo-service.yaml` (pointing to Blue)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-service
spec:
  selector:
    app: demo
    version: blue   # currently pointing to BLUE
  ports:
  - port: 80
    targetPort: 80
```

### Change to Green
```yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-service
spec:
  selector:
    app: demo
    version: green   # switched to GREEN
  ports:
  - port: 80
    targetPort: 80
```

Apply the change:
```bash
kubectl apply -f demo-service.yaml
```

Test:
```bash
curl http://demo.local:30080
Hello from GREEN
```

Rollback (switch back to Blue):
```yaml
selector:
  app: demo
  version: blue
```
```bash
kubectl apply -f demo-service.yaml
curl http://demo.local:30080
Hello from BLUE
```

---

### Option B: Change `/etc/hosts` Mapping

### Initial `/etc/hosts`
```text
10.10.0.2 blue.local
10.10.0.2 green.local
10.10.0.2 demo.local   # pointing to BLUE ingress
```

### Switch to Green
Edit `/etc/hosts`:
```bash
sudo sed -i 's/blue.local/demo.local/' /etc/hosts
```
Or explicitly:
```bash
echo "10.10.0.2 green.local demo.local" | sudo tee -a /etc/hosts
```

Test:
```bash
curl http://demo.local:30080
Hello from GREEN
```

Rollback:
```bash
echo "10.10.0.2 blue.local demo.local" | sudo tee -a /etc/hosts
curl http://demo.local:30080
Hello from BLUE
```

---

### Rollback Demo Section

### Promote Green
```bash
kubectl apply -f demo-service.yaml   # Option A
# or update /etc/hosts for demo.local -> green.local   # Option B
curl http://demo.local:30080
Hello from GREEN
```

### Rollback to Blue
```bash
kubectl apply -f demo-service.yaml   # reset selector to blue
# or update /etc/hosts for demo.local -> blue.local
curl http://demo.local:30080
Hello from BLUE
```
