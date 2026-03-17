```text
                +-------------------+
                |   Client Request  |
                +-------------------+
                          |
                          v
                +-------------------+
                |   Ingress (NGINX) |
                +-------------------+
                          |
                          v
                +-------------------+
                |   demo.local      |
                |   (Demo Service)  |
                +-------------------+
                          |
          ---------------------------------
          |                               |
          v                               v
+-------------------+           +-------------------+
|  Blue Service     |           |  Green Service    |
| (Production Pods) |           | (New Version Pods)|
+-------------------+           +-------------------+
          |                               |
          v                               v
+-------------------+           +-------------------+
|  Blue Deployment  |           |  Green Deployment |
| "Hello from BLUE" |           | "Hello from GREEN"|
+-------------------+           +-------------------+
```

### What is a Blue‑Green Deployment?

- **Blue (Current Production):** The stable version of your application that all traffic initially flows to.  
- **Green (New Version):** A parallel environment running the updated application, isolated from production traffic until promotion.  
- **Demo Service/Ingress:** Provides a single entry point (`demo.local`) that can be switched between Blue and Green by editing the service selector.  
- **Traffic Switching:** Unlike Canary (which splits traffic), Blue‑Green sends 100% of traffic to either Blue or Green. Switching is binary and immediate.  
- **Purpose:** Allows you to test the new version (Green) side‑by‑side with Blue, then flip production traffic when ready.  
- **Rollback Safety:** If issues occur, you can instantly revert traffic back to Blue by changing the service selector.  

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

## Testing Blue‑Green Deployment with Demo Service

Once Blue and Green environments are deployed and tested individually (`blue.local`, `green.local`), you can introduce a **demo service and ingress** to act as the single production entry point (`demo.local`). This allows you to flip traffic between Blue and Green easily.

---

## Demo Service (initially pointing to Blue)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-service
spec:
  selector:
    app: demo
    version: blue   # initially BLUE
  ports:
  - port: 80
    targetPort: 80
```

### Demo Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress
  namespace: default
spec:
  rules:
  - host: demo.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: demo-service
            port:
              number: 80
```

---

### Apply and Configure Hosts

```bash
kubectl apply -f demo-service.yaml
kubectl apply -f demo-ingress.yaml
```

Add host entry:
```bash
echo "10.10.0.2 demo.local" | sudo tee -a /etc/hosts
```

---

### Test Initial Traffic

```bash
curl http://demo.local:30080
Hello from BLUE
```

---

### Promote Green

Edit `demo-service.yaml` selector to point to Green:

```yaml
spec:
  selector:
    app: demo
    version: green   # switched to GREEN
```

Re‑apply:
```bash
kubectl apply -f demo-service.yaml
```

Test:
```bash
curl http://demo.local:30080
Hello from GREEN
```

---

### Rollback to Blue

If needed, change the selector back:

```yaml
spec:
  selector:
    app: demo
    version: blue
```

Re‑apply:
```bash
kubectl apply -f demo-service.yaml
```

Test:
```bash
curl http://demo.local:30080
Hello from BLUE
```




