<img width="1308" height="693" alt="image" src="https://github.com/user-attachments/assets/294e6327-b3ec-4cba-96f0-f9e46badeefa" /># Title & Overview:

This guide documents the process of setting up an AWS EKS cluster, deploying the AWS Load Balancer Controller, and installing ArgoCD for GitOps deployments.

# Prerequisites:

- AWS CLI configured
- kubectl installed
- eksctl installed
- Helm installed
- IAM permissions to create EKS, IAM roles, ALB, and S3 access
- GitHub repository for ArgoCD manifests

# EKS Cluster Creation:

eksctl create cluster \
  --name my-eks-cluster \
  --version 1.31 \
  --region ap-south-1 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --managed

  # Verification:
  kubectl get nodes -o wide
  kubectl get pods -A
  
  <img width="1051" height="179" alt="image" src="https://github.com/user-attachments/assets/15674495-6522-488d-9458-619463b42047" />


---

## Install AWS Load Balancer Controller ##

# Create IAM OIDC provider for EKS
eksctl utils associate-iam-oidc-provider \
  --region ap-south-1 \
  --cluster my-eks-cluster \
  --approve

# Create IAM policy for ALB controller
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json

# Create service account and attach policy
eksctl create iamserviceaccount \
  --cluster my-eks-cluster \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --attach-policy-arn arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve

# Install Helm chart
helm repo add eks https://aws.github.io/eks-charts
helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-eks-cluster \
  --set serviceAccount.create=false \
  --set region=ap-south-1 \
  --set vpcId=<VPC_ID>

  #  Verification
 kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller

<img width="1049" height="143" alt="image" src="https://github.com/user-attachments/assets/6512bec7-1ddc-4da2-a1f2-c2b9fa2f2f79" />


---

#  Deploy ArgoCD

kubectl create namespace argocd

# Install ArgoCD using manifests
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Expose argocd-server using Ingress or LoadBalancer
kubectl apply -f argocd-ingress.yaml -n argocd

# Verification
<img width="1253" height="687" alt="image" src="https://github.com/user-attachments/assets/dc51b087-f4b9-4b29-8766-53a3888cb897" />
<img width="1308" height="693" alt="image" src="https://github.com/user-attachments/assets/3b3399dd-0f3b-48bc-95de-fd5870f44c8a" />

# After getting ArgoCD Admin Login Page, Command to Retrieve Password

kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

  After login Create Application with the respective github repo yaml
  <img width="1308" height="693" alt="image" src="https://github.com/user-attachments/assets/c223b943-e8af-45c1-8de5-c4c7c9329a90" />

# output #

<img width="1308" height="171" alt="image" src="https://github.com/user-attachments/assets/734f622b-ca2b-43e5-946e-4d1a3d932e82" />
<img width="1305" height="549" alt="image" src="https://github.com/user-attachments/assets/ad3144cd-b522-4635-a419-4489c533c50a" />


--
## Troubleshooting

- **ALB Error**: TargetGroup port is empty
  - Ensure your Service is of type `NodePort` or `LoadBalancer`
  - Ensure `alb.ingress.kubernetes.io/listen-ports` annotation matches service port

- **DNS / Ingress not resolving**
  - Use `nslookup` or `dig` to check DNS propagation
  - Check if LoadBalancer has `internet-facing` scheme

- **ArgoCD Application invalid**
  - Ensure app name follows RFC1123 lowercase rules (no spaces)
  - Ensure manifest path in repo is relative, not absolute



  
