# 🚀 OpenID Connect (OIDC) in AWS EKS

## 🔹 **What is OIDC?**
OpenID Connect (**OIDC**) is an authentication protocol built on **OAuth 2.0** that allows applications to verify user identity using an **identity provider (IdP)** like AWS IAM, Google, Okta, or GitHub.

In **AWS EKS**, OIDC is used to allow Kubernetes **Pods** to securely access **AWS services** without requiring long-term AWS credentials.

---

## 🔹 **Why is OIDC Used in AWS EKS?**
EKS uses OIDC for authentication when integrating IAM roles with Kubernetes **ServiceAccounts**. This enables **IAM Roles for Service Accounts (IRSA)**, allowing Pods to assume IAM roles dynamically and access AWS services **securely**.

### ✅ **Key Benefits:**
- 🔐 **No Need for Long-Term Credentials** - Avoids storing AWS access keys inside Pods.
- 🛡 **Secure IAM Role Access** - Follows **least privilege** best practices.
- 🎯 **Fine-Grained Access Control** - Each Pod can have a different IAM role based on its ServiceAccount.

---

## 🔹 **How OIDC Works in AWS EKS**
1️⃣ **EKS Cluster has an OIDC Provider** – AWS automatically creates an OIDC identity provider when an EKS cluster is set up.
2️⃣ **IAM Role is Associated with the OIDC Provider** – The IAM role is created with an **OIDC trust policy**.
3️⃣ **IAM Role is Mapped to a Kubernetes ServiceAccount** – The Kubernetes **ServiceAccount** is linked to the IAM role.
4️⃣ **Pods Assume IAM Role via ServiceAccount** – Any Pod using the ServiceAccount can securely access AWS services with temporary credentials.

---

## 🔹 **How to Check If OIDC is Enabled in Your EKS Cluster**
Run the following command to check if OIDC is enabled for your EKS cluster:
```bash
aws eks describe-cluster --name <your-cluster-name> --query "cluster.identity.oidc.issuer" --output text
```
✅ **If OIDC is enabled**, you will see an output like:
```
https://oidc.eks.us-east-2.amazonaws.com/id/EKS_CLUSTER_ID
```

---

## 🔹 **Setting Up OIDC with IRSA in AWS EKS**
### 1️⃣ **Create an OIDC Provider for Your EKS Cluster**
```bash
eksctl utils associate-iam-oidc-provider --region <AWS_REGION> --cluster <EKS_CLUSTER_NAME> --approve
```

### 2️⃣ **Create an IAM Role with OIDC Trust Policy**
Create a JSON file (`trust-policy.json`) for the IAM role:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::<AWS_ACCOUNT_ID>:oidc-provider/oidc.eks.<AWS_REGION>.amazonaws.com/id/<EKS_CLUSTER_ID>"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "oidc.eks.<AWS_REGION>.amazonaws.com/id/<EKS_CLUSTER_ID>:sub": "system:serviceaccount:default:my-service-account"
                }
            }
        }
    ]
}
```
Create the IAM Role:
```bash
aws iam create-role --role-name EKS_OIDC_ROLE --assume-role-policy-document file://trust-policy.json
```

### 3️⃣ **Attach Required Policies to the IAM Role**
Example: Attach Amazon S3 policy
```bash
aws iam attach-role-policy --role-name EKS_OIDC_ROLE --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

### 4️⃣ **Create a Kubernetes ServiceAccount and Associate It with the IAM Role**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-service-account
  namespace: default
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<AWS_ACCOUNT_ID>:role/EKS_OIDC_ROLE
```
Apply it:
```bash
kubectl apply -f service-account.yaml
```

### 5️⃣ **Deploy a Pod Using the ServiceAccount**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
  namespace: default
spec:
  serviceAccountName: my-service-account
  containers:
  - name: test-container
    image: amazonlinux
    command: ["sleep", "3600"]
```
Apply the Pod:
```bash
kubectl apply -f test-pod.yaml
```

### 6️⃣ **Verify the Pod Can Assume the IAM Role**
Enter the Pod and check AWS credentials:
```bash
kubectl exec -it test-pod -- aws sts get-caller-identity
```
✅ **If successful, the output will show the IAM role assumed by the Pod.**

---

## 🔹 **Conclusion**
- 🔹 **OIDC enables AWS EKS to securely authenticate Kubernetes Pods** to AWS services **without hardcoded credentials**.
- 🔹 **IRSA (IAM Roles for Service Accounts) allows Pods to assume IAM roles dynamically**.
- 🔹 **Follow best practices by using least privilege policies** to grant only the necessary AWS permissions.

---
