

# **Private LLM Deployment on AKS: Ollama + Open Web UI**

## Why This Matters Now
**73% of enterprises** cite data privacy as their top AI concern (Gartner 2024). This guide shows how to run LLMs entirely on your hardware without cloud dependencies.
## Key Benefits of Private Deployment
:lock: **Full Data Control** - No data leaves your infrastructure. 

:money_with_wings: **Zero Ongoing Costs** - No per-token fees or API charges. 

:zap: **Low-Latency** - Local processing eliminates network delays. 

🕧 **Customizable** - Modify/train models to your needs. 

:globe_with_meridians: **Offline Capable** - Air-gapped environment support.

## **1. Prerequisites**
- Azure account with active subscription
- Azure CLI installed (`az`)
- `kubectl` configured
- Docker (for local testing)

---

## **2. Create AKS Cluster**
```bash
# Create resource group
az group create --name ollama-aks --location eastus

# Create AKS cluster (GPU optional)
az aks create \
  --resource-group ollama-aks \
  --name ollama-cluster \
  --node-count 3 \
  --node-vm-size Standard_D4s_v3 \
  --enable-addons monitoring \
  --generate-ssh-keys

# Get credentials
az aks get-credentials --resource-group ollama-aks --name ollama-cluster
```

---

## **3. Prepare Kubernetes Manifests**

### **3.1 Namespace Setup**
`namespace.yaml`:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ollama
```

### **3.2 Ollama StatefulSet**
`ollama-statefulset.yaml`:
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: ollama
  namespace: ollama
spec:
  serviceName: ollama
  replicas: 1
  selector:
    matchLabels:
      app: ollama
  template:
    metadata:
      labels:
        app: ollama
    spec:
      containers:
      - name: ollama
        image: ghcr.io/ollama/ollama:latest
        ports:
        - containerPort: 11434
        volumeMounts:
        - name: ollama-data
          mountPath: /root/.ollama
        resources:
          requests:
            cpu: "2"
            memory: "8Gi"
  volumeClaimTemplates:
  - metadata:
      name: ollama-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 50Gi
      storageClassName: default # Use Azure Disk
```

### **3.3 Open Web UI Deployment**
`openwebui-deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: openwebui
  namespace: ollama
spec:
  replicas: 1
  selector:
    matchLabels:
      app: openwebui
  template:
    metadata:
      labels:
        app: openwebui
    spec:
      containers:
      - name: openwebui
        image: ghcr.io/openwebui/openwebui:latest
        env:
        - name: OLLAMA_BASE_URL
          value: "http://ollama.ollama.svc.cluster.local:11434"
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: webui-data
          mountPath: /app/backend/data
      volumes:
      - name: webui-data
        persistentVolumeClaim:
          claimName: webui-pvc

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: webui-pvc
  namespace: ollama
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi

---
apiVersion: v1
kind: Service
metadata:
  name: openwebui
  namespace: ollama
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: openwebui
```

---

## **4. Deploy to AKS**
```bash
# Apply all manifests
kubectl apply -f namespace.yaml
kubectl apply -f ollama-statefulset.yaml
kubectl apply -f openwebui-deployment.yaml

# Verify deployment
kubectl get pods -n ollama -w

# Check persistent volumes
kubectl get pvc -n ollama
```

---

## **5. Initialize Ollama Model**
```bash
# Connect to Ollama pod
kubectl exec -n ollama -it ollama-0 -- bash

# Inside container:
ollama pull phi3
ollama run phi3
```

---

## **6. Access Open Web UI**
```bash
# Get public IP
kubectl get svc -n ollama openwebui -o jsonpath='{.status.loadBalancer.ingress[0].ip}'

# Access via browser:
http://<EXTERNAL-IP>
```

---

## **7. Security Hardening (Optional)**

### **7.1 Enable Authentication**
```bash
# Update Open Web UI deployment
env:
- name: OPENWEBUI_AUTH
  value: "true"
- name: OPENWEBUI_API_KEY
  value: "your-secret-key-here"
```

### **7.2 Network Policies**
`network-policy.yaml`:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ollama-isolation
  namespace: ollama
spec:
  podSelector:
    matchLabels:
      app: ollama
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: openwebui
```

---

## **8. Verification Tests**

### **8.1 API Test**
```bash
curl http://ollama.ollama.svc.cluster.local:11434/api/tags
```

### **8.2 Load Test**
```bash
kubectl run -n ollama -it --rm load-test --image=alpine:latest -- sh
apk add curl
for i in {1..100}; do
  curl -s -o /dev/null -w "%{http_code}" ollama:11434/api/generate -d '{"model":"phi3","prompt":"test"}'
done
```

---

## **9. Cleanup**
```bash
az group delete --name ollama-aks --yes
kubectl delete ns ollama
```

---

## **Troubleshooting Guide**

| **Issue**                     | **Solution**                                  |
|-------------------------------|-----------------------------------------------|
| Model not downloading         | Check pod logs: `kubectl logs -n ollama ollama-0` |
| Open Web UI connection failed | Verify service: `kubectl describe svc -n ollama openwebui` |
| Persistent volume claims pending | Check storage class: `kubectl get storageclass` |

---

This provides a complete production-ready deployment with:
- Persistent storage for models
- Isolated network policies
- Load-balanced web interface
- Scalable architecture
