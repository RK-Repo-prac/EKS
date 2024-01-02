# ALB Ingress Contoller

ALB Ingress controller is a Kubernetes controller which works with AWS Application Load Balancer to route traffic from the external world to
inside our cluster where our application is deployed inside a pod.
when using an ALB Ingress Controller with Amazon EKS (Elastic Kubernetes Service), you will need the appropriate AWS Identity and Access Management (IAM) permissions. The IAM permissions are necessary for the ALB Ingress Controller to interact with AWS resources, such as creating and managing Application Load Balancers. TO do thiis we need to configure IAM OIDC Provider

## IAM OIDC Provider
 IAM OIDC is used to connect IAM with Kuberbnetes. SO for this IAM OIDC you will assign IAM roles. So when a kubernetes entity tries to access AWS resource then it will forward to IAM to check whether the kubernetes cluster has permisiion to access that particular resource or not. If these  permissions are met then the kubernetes resource here ALB Ingress controller can access the AWS resource
  
```
eksctl utils associate-iam-oidc-provider --cluster k8s-cluster --approve
```
Next create a IAM role policy using the in our IAM policy

```
AWSLoadBalancerControllerIAMPolicy.json
```

Now once the policy is created , create a role and attach that policy
```
eksctl create iamserviceaccount \
  --cluster=k8s-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve

```
In simple terms, this single command:
* Creates a Kubernetes service account named aws-load-balancer-controller
* Creates an IAM role named AmazonEKSLoadBalancerControllerRole
* Attaches the AWSLoadBalancerControllerIAMPolicy managed policy to this IAM role.
* Binds the Kubernetes service account to the IAM role
This allows the AWS Load Balancer controller to access AWS resources and APIs by assuming the IAM role attached to its service account. The policy defines the permissions it needs.So in summary, it sets up access permissions for the controller to provision/manage AWS load balancers.

## Deploying ALB Controller
Add helm repo
```
helm repo add eks https://aws.github.io/eks-charts
```
Update the repo
```
helm repo update eks
```
Install the ALB

```
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \            
  -n kube-system \
  --set clusterName=k8s-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=vpc-0aa5e2e13e98401aa

```
Verify that the deployments are running.
```
kubectl get deployment -n kube-system aws-load-balancer-controller

```
Once the deployment is created, our Ingress controller using the service account aws-load-balancer-controller will create a Load Balancer based on our Ingress resource. And one your Load Balancer is active using the DNS of our Load Balancer we can access our application from any where.


