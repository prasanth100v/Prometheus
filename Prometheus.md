✅ Below is a complete, end-to-end step-by-step guide to install Prometheus on AWS EKS (production-ready, simple and clear).

🔹 STEP 1: Prerequisites
```
aws --version
kubectl version --client
eksctl version
```
✅ (BEST PRACTICE): Attach AmazonEBSCSIDriverPolicy using IRSA 
  This attaches the policy only to the EBS CSI controller pod, not to all worker nodes.
🔹 STEP 2: Enable IAM OIDC Provider (MANDATORY)
```
eksctl utils associate-iam-oidc-provider \
  --cluster eks-cluster \
  --region ap-south-1 \
  --approve
```
🔹 List all ServiceAccounts in a namespace
```
kubectl get serviceaccounts -n kube-system
```
Clean up the WRONG ServiceAccount
```
kubectl delete sa ebs-csi-controller-saa -n kube-system
```

🔐 Step 3:Create ONLY IAM ROLE (NO ServiceAccount)
⚠️ Do NOT create Kubernetes ServiceAccount manually
```
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster eks-cluster \
  --region ap-south-1 \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --role-only \
  --approve
```
✅ This creates:
IAM role
OIDC trust policy
❌ Does NOT create ServiceAccount

✅ 👉 Copy the IAM ROLE ARN from here
```
eksctl get iamserviceaccount \
  --cluster eks-cluster \
  --region ap-south-1
```
🔹 STEP 4: Install EBS CSI Addon (Addon creates ServiceAccount)
```
aws eks create-addon \
  --cluster-name eks-cluster \
  --addon-name aws-ebs-csi-driver \
  --service-account-role-arn <ROLE_ARN_FROM_STEP_3> \
  --region ap-south-1 \
  --resolve-conflicts OVERWRITE
```
✔ Addon will:

Create ebs-csi-controller-sa
Attach IAM role
Set correct labels

🔹 STEP 5: Verify Addon Status
```
aws eks describe-addon \
  --cluster-name eks-cluster \
  --addon-name aws-ebs-csi-driver
```

Step 6: Create CSI-Based StorageClass (gp3) = Create a new StorageClass using CSI driver.
```
vi gp3-sc.yaml
```
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  fsType: ext4
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```
✅Apply
```
kubectl apply -f gp3-sc.yaml
```
🔍 Step 7: Verify Installation
```
kubectl get pods -n kube-system | grep ebs
kubectl get sc
```

💾 Step 8: Make gp3 Default (Recommended) this step is OPTIONAL: Only required if gp2 already exists and is default
```
kubectl patch storageclass gp2 \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'

kubectl patch storageclass gp3 \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

STEP 9: Create Namespace
```
kubectl create namespace monitoring
```
✅ Recommended values.yaml for Prometheus on EKS (EBS CSI)
```
vi values.yaml
```
```
server:
  persistentVolume:
    enabled: true
    storageClass: gp3
    accessModes:
      - ReadWriteOnce
    size: 8Gi
  retention: "15d"

alertmanager:
  enabled: true
  persistentVolume:
    enabled: true
    storageClass: gp3
    accessModes:
      - ReadWriteOnce
    size: 2Gi
    mountPath: /data
```
🔹 STEP 10: Add Helm Repo
```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
chmod 700 get_helm.sh
./get_helm.sh
helm version
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm upgrade --install prometheus prometheus-community/prometheus \
  -n monitoring \
  --create-namespace \
  -f values.yaml
```

🔍 5️⃣ Verify (Very Important)
```
kubectl get pvc -n monitoring
kubectl get pods -n monitoring
```
🔹Access Prometheus UI (via LoadBalancer)
  ✏️ Edit Prometheus Service
  ```
kubectl edit svc prometheus-server -n monitoring
```
🔄 Change this:
```
spec:
  type: ClusterIP  -->  type: LoadBalancer
```
🔹Access Alertmanager UI (via LoadBalancer)
  ✏️ Edit Alertmanager Service
```
kubectl edit svc prometheus-alertmanager -n monitoring
```
🔄 Change this:
```
spec:
  type: ClusterIP  -->  type: LoadBalancer
```
Save and exit.

🌐 Get External URL
```
kubectl get svc prometheus-alertmanager -n monitoring
```
