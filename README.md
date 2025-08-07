
# 🛠️ Lugx gaming solution runbook

## Prerequisites
- OS: Linux/macOS/Windows
- IAM user with sufficient AWS permissions (EKS, ECR, IAM, VPC, EC2)
- AWS credentials (access key and secret)

---

## 1. ✅ Install AWS CLI

### Linux/macOS
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

### Windows
Download from: [https://aws.amazon.com/cli/](https://aws.amazon.com/cli/)

### Verify
```bash
aws --version
```

---

## 2. ✅ Install `kubectl`

### Linux/macOS
```bash
curl -LO "https://dl.k8s.io/release/$(curl -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

### Windows
Download from: [https://kubernetes.io/docs/tasks/tools/](https://kubernetes.io/docs/tasks/tools/)

### Verify
```bash
kubectl version --client
```

---

## 3. 🔐 Authenticate to AWS using CLI

```bash
aws configure
```

Provide:
- AWS Access Key ID
- AWS Secret Access Key
- Default region (e.g., `us-west-2`)
- Default output format (e.g., `json`)

---

## 4. 🔐 Authenticate to Kubernetes (EKS)

### Install `eksctl` (optional, but helpful)
```bash
curl --silent --location "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
```

### Update kubeconfig
```bash
aws eks update-kubeconfig --region <region> --name <cluster_name>
```

### Verify cluster access
```bash
kubectl get svc
```

---

## 5. ⚙️ Create an EKS Cluster

### Option A: Using `eksctl` (recommended)
```bash
eksctl create cluster   --name my-cluster   --region us-west-2   --nodegroup-name linux-nodes   --node-type t3.medium   --nodes 2   --nodes-min 1   --nodes-max 3   --managed
```

### Option B: Using AWS CLI
1. Create EKS control plane:
```bash
aws eks create-cluster   --name my-cluster   --region us-west-2   --role-arn arn:aws:iam::<account-id>:role/<eks-service-role>   --resources-vpc-config subnetIds=<subnet-ids>,securityGroupIds=<sg-ids>
```

2. Wait for the cluster status to become `ACTIVE`.

3. Check openID provider exist for the cluster
```bash
aws eks describe-cluster --name <cluster_name> --query "cluster.identity.oidc.issuer" --output text
```

4. Confirm OpenID provider is configured. Following command should output an ARN.
```bash
aws iam list-open-id-connect-providers | grep <openid_provider_id>
```

5. If OICD provider is not configured, associate IAM OICD provider to the cluster
```bash
eksctl utils associate-iam-oidc-provider --cluster <cluster_name> --approve
```

6. Confirm OpenID provider is configure. This is important for Volume provisioning.

7. Create IAM trust_policy file
```bash
cat <<EOF > trust-policy.json 
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<aws_account_id>:oidc-provider/oidc.eks.us-east-2.amazonaws.com/id/<openid_provider_id>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.us-east-2.amazonaws.com/id/<openid_provider_id>:aud": "sts.amazonaws.com",
          "oidc.eks.us-east-2.amazonaws.com/id/<openid_provider_id>:sub": "system:serviceaccount:kube-system:ebs-csi-controller-sa"
        }
      }
    }
  ]
}
EOF
```

8. Use the created trust-policy.json to create a role

```bash
aws iam create-role \
> --role-name AmazonEKS_EBS_CSI_DriverRole \
> --assume-role-policy-document file://"trust-policy.json"
```

9. Attach EBS CSID policy to the role
```bash
aws iam attach-role-policy \
>  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
>  --role-name AmazonEKS_EBS_CSI_DriverRole
```

9. Create EBS CSID addon
```bash
aws eks create-addon \
> --addon-name aws-ebs-csi-driver \
> --service-account-role-arn arn:aws:iam::<aws_account_id>:role/AmazonEKS_EBS_CSI_DriverRole \
> --cluster-name <cluster_name>
```

10. Update kubeconfig
```bash
aws eks update-kubeconfig --region <region-code> --name <cluster-name>
```

11. Create nodegroup if it doesn't exist
```bash


---

## 6. 🐳 Create an ECR Repository

```bash
aws ecr create-repository --repository-name my-app --region us-west-2
```

### Authenticate Docker to push images
```bash
aws ecr get-login-password --region us-west-2 | docker login --username AWS --password-stdin <aws_account_id>.dkr.ecr.us-west-2.amazonaws.com
```

---


## 🚀 Post-Setup Deployment Steps

### 7. 📦 Apply Storage Class for EBS gp3

```bash
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: lugx-gp3
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp3
  fsType: ext4
  encrypted: "true"
EOF
```

---

### 8. 🧬 Clone All Application Repositories

```bash
git clone --recurse-submodules https://github.com/ChoxmiIIT
cd ChoxmiIIT
```

---

### 9. 🔁 Update Configuration

- Replace placeholder values in `deployment.yaml`, `*.sh`, GitHub Actions, and Helm charts:

```text
<OLD_ACCOUNT_ID> → <NEW_ACCOUNT_ID>
<OLD_CLUSTER_NAME> → <YOUR_CLUSTER_NAME>
<OLD_ECR_PATH> → <NEW_ECR_PATH>
```

Use a global search/replace tool or script to do this efficiently.

---

### 10. 🔐 Set AWS Credentials in GitHub

- Go to your GitHub repo → Settings → Secrets → Actions
- Add or update:
  - `AWS_ACCESS_KEY_ID`
  - `AWS_SECRET_ACCESS_KEY`

---

### 11. 🧪 Trigger GitHub Actions

Option 1: Run the latest GitHub Action manually  
Option 2: Push a dummy commit:

```bash
git commit --allow-empty -m "Trigger deployment"
git push
```

This will trigger CI/CD and deploy to EKS.

---

### 12. 🌐 Apply Ingress for ALB

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]'
    alb.ingress.kubernetes.io/backend-protocol: HTTP
    alb.ingress.kubernetes.io/group.name: public-apps
spec:
  ingressClassName: alb
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
      - path: /game-data
        pathType: Prefix
        backend:
          service:
            name: node-game-data-service
            port:
              number: 5000
      - path: /order
        pathType: Prefix
        backend:
          service:
            name: node-order-service
            port:
              number: 5000
      - path: /analytics
        pathType: Prefix
        backend:
          service:
            name: analytics-service
            port:
              number: 5000
EOF
```

---

### 13. 🔎 Get ALB Hostname

```bash
kubectl get ingress app-ingress
```

Use the `ADDRESS` or `HOSTS` value as your application endpoint.

---

### 14. 🔐 Allow ClickHouse Access

After applying `clickhouse-service.yaml`:

1. Identify the internal Load Balancer service:
   ```bash
   kubectl get svc
   ```
2. Copy the hostname under `EXTERNAL-IP` for `clickhouse-service`.

3. Go to AWS EC2 → Security Groups → Edit inbound rules:
   - Add rules for ports:
     - `8123` (HTTP interface)
     - `9004` (TCP native interface)
   - Source: Your IP or CIDR used by QuickSight

---

### 15. 📊 Connect QuickSight to ClickHouse

- Host: `internal-load-balancer-hostname`
- Port: `9004`
- JDBC: `clickhouse://<hostname>:9004/default`
- Auth: As configured
- Test and Save the data source in QuickSight
