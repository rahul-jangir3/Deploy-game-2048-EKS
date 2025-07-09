
## Install and configure kubectl

```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
kubectl version --client --output=yaml
```


 ## Install eksctl

```
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"
```
```
curl -sL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_checksums.txt" | grep $PLATFORM | sha256sum --check
tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz
sudo install -m 0755 /tmp/eksctl /usr/local/bin && rm /tmp/eksctl
eksctl --version
```


 ## Install AWS CLI

```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
aws configure
```


 ## Install AWS CLI (alternative method)

```
curl "https://s3.amazonaws.com/aws-cli/awscli-bundle.zip" -o "awscli-bundle.zip"
sudo apt install unzip
unzip awscli-bundle.zip
sudo ./awscli-bundle/install -i /usr/local/aws -b /usr/local/bin/aws
```


 ## Create EKS cluster

```eksctl create cluster --name demo-cluster --region ap-south-1 --fargate
```


 ## Configure kubectl for EKS

```
aws eks update-kubeconfig --name demo-cluster --region ap-south-1
```
## Check cluster resources

```
kubectl get ns
kubectl get pods -n kube-system
```
## Create Fargate profile for game-2048 application

```
eksctl create fargateprofile --cluster demo-cluster --region ap-south-1 --name alb-sample-app --namespace game-2048
```

## Deploy 2048 game application

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
kubectl get pods -n game-2048
kubectl get svc -n game-2048
kubectl get ingress -n game-2048
```

## Set up AWS Load Balancer Controller

```
export cluster_name=demo-cluster
eksctl utils associate-iam-oidc-provider --cluster demo-cluster --approve
oidc_id=$(aws eks describe-cluster --name $cluster_name --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json
```

## Create IAM service account for Load Balancer Controller

```
eksctl create iamserviceaccount --cluster= --namespace=kube-system --name=aws-load-balancer-controller --role-name AmazonEKSLoadBalancerControllerRole --attach-policy-arn=arn:aws:iam:::policy/AWSLoadBalancerControllerIAMPolicy --approve
```

## Create IAM service account for AWS Load Balancer Controller

```
eksctl create iamserviceaccount \
  --cluster=demo-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::886436931941:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

## Add and update Helm repository

```
helm repo add eks https://aws.github.io/eks-charts
./get_helm.sh
helm repo update eks
```

## Install AWS Load Balancer Controller

```
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=demo-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=ap-south-1 \
  --set vpcId=vpc-0172d39afb61d0b45
```

## Verify deployment and resources

```
kubectl get deployment -n kube-system aws-load-balancer-controller
kubectl get deployment -n kube-system aws-load-balancer-controller -w
kubectl get pods -n kube-system -w
kubectl edit deploy/aws-load-balancer-controller -n kube-system
kubectl get deploy -n kube-system
kubectl get ingress -n game-2048
```