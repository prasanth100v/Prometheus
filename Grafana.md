### STEP 1: Install Grafana using Helm
Add Grafana Helm Repo
```
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```
STEP 2: Create Grafana values.yaml (Persistent + Secure)
```
vi grafana-values.yaml
```
```
persistence:
  enabled: true
  storageClassName: gp3
  size: 5Gi
  accessModes:
    - ReadWriteOnce

adminUser: admin
adminPassword: admin123   # change in production

service:
  type: LoadBalancer

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 128Mi
```
📌 Why persistence?

Dashboards
Data sources
Users
Alerts
            will survive pod restarts.

STEP 3: Install Grafana
```
helm upgrade --install grafana grafana/grafana \
  -n monitoring \
  --create-namespace \
  -f grafana-values.yaml
```
STEP 4: Verify Grafana
```
kubectl get pods -n monitoring | grep grafana
kubectl get pvc -n monitoring | grep grafana
kubectl get svc grafana -n monitoring
```
Expected:

Pod → Running
PVC → Bound
Service → LoadBalancer

### Login:
```
Username: admin
Password: admin123
```


