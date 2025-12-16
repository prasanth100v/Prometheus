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
