
# Step 1: Create EKS Fargate Cluster
```
eksctl create cluster --name demo-cluster --region us-east-1 --fargate
aws eks update-kubeconfig --name demo-cluster --region us-east-1
```

# Step 2: Create Fargate Profile & Deploy 2048 App
```
eksctl create fargateprofile \
  --cluster demo-cluster \
  --region us-east-1 \
  --name alb-sample-app \
  --namespace game-2048
```

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml

```

# Step 3: Verify App Deployment
```
kubectl get pods -n game-2048 -w
kubectl get svc -n game-2048 -w
kubectl get ingress -n game-2048

```
 The service-2048 is of type NodePort, so no external access yet.

 Ingress will expose the app once the ALB controller is configured.

 Once successful, kubectl get ingress will show an ADDRESS (DNS name of ALB).

# Step 4: Associate IAM OIDC Provider
```
eksctl utils associate-iam-oidc-provider \
  --cluster demo-cluster \
  --approve

```
# Step 5: Create IAM Policy and Role for ALB Controller
# Download IAM Policy
```
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json
```
# Create IAM Policy
``` 
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json

```
# Create IAM Service Account
```
eksctl create iamserviceaccount \
  --cluster=demo-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name=AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::516611517801:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```
#  Step 6: Install ALB Ingress Controller
```
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks

```
# Install the Controller
``` helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=<cluster-name> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=<vpc-id>
```
#  Step 7: Verify Controller is Running
```
kubectl get deployment -n kube-system aws-load-balancer-controller
```
If needed, inspect logs or edit for troubleshooting:
```
kubectl edit deploy/aws-load-balancer-controller -n kube-system
```
Once the controller is active, check your Ingress again to see the ALB DNS:

```
kubectl get ingress -n game-2048
```

# Deletion / Cleanup
# Delete the App and Controller

```
kubectl delete -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
helm uninstall aws-load-balancer-controller -n kube-system

```
# Delete Service Account and IAM Policy
```
eksctl delete iamserviceaccount \
  --name aws-load-balancer-controller \
  --namespace kube-system \
  --cluster demo-cluster \
  --region us-east-1

aws iam delete-policy \
  --policy-arn arn:aws:iam::516611517801:policy/AWSLoadBalancerControllerIAMPolicy
```
# Delete Fargate Profile, Cluster, and Namespace

```
eksctl delete fargateprofile \
  --name alb-sample-app \
  --cluster demo-cluster \
  --region us-east-1

kubectl delete ingress ingress-2048 -n game-2048
kubectl delete ns game-2048

eksctl delete cluster --name demo-cluster --region us-east-1
```
# Notes
You must replace the vpcId, AWS account ID, and region as per your setup.

The ALB Ingress Controller is required to automatically manage the lifecycle of Application Load Balancers for Kubernetes Ingress resources.
