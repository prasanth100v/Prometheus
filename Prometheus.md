✅ Below is a complete, end-to-end step-by-step guide to install Prometheus on AWS EKS (production-ready, simple and clear).

🔹 STEP 1: Prerequisites
```
aws --version
kubectl version --client
helm version
```
🔹 STEP 2: Enable IAM OIDC Provider (MANDATORY)
```
eksctl utils associate-iam-oidc-provider \
  --cluster <your-cluster-name> \
  --region ap-south-1 \
  --approve
```
🔐 Step 3: Create IAM Role for EBS CSI Driver (IRSA – Recommended)
Create IAM Service Account
```
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster eks-cluster \
  --region ap-south-1 \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve
```
✅ This creates:

IAM role
Trust relationship with OIDC
Kubernetes ServiceAccount

✅✅🚀 Step 4: Install EBS CSI Driver (Managed Add-on – Best)
```
aws eks create-addon \
  --cluster-name <cluster-name> \
  --addon-name aws-ebs-csi-driver \
  --service-account-role-arn <ROLE_ARN_CREATED_ABOVE> \
  --region <region>
```
🔍 Step 5: Verify Installation
```
kubectl get pods -n kube-system | grep ebs
kubectl get sc
```

💾 Step 6: Make gp3 Default (Recommended)
```
kubectl patch storageclass gp2 \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'

kubectl patch storageclass gp3 \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

STEP 7: Create Namespace
```
kubectl create namespace monitoring
```
🔹 STEP 8: Add Helm Repo
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```
✅ Recommended values.yaml for Prometheus on EKS (EBS CSI)
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

🚀 4️⃣ Install / Upgrade Prometheus
```
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






4️⃣ create EBS CSI IRSA (clean & correct) This fixes 99% of CrashLoopBackOff cases.
```
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster <your-cluster-name> \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --override-existing-serviceaccounts \
  --approve
```
✅ This:

Creates IAM role

Attaches policy

Annotates service account correctly
