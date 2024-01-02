### Deploying an Application on EKS

## Architecture
To overcome difficulties with the self-managed Kubernetes cluster, I plan to deploy my application inside an AWS managed control plane service, namely Elastic Kubernetes Service (EKS).And leverage Fargate as serverless compute for the worker nodes.With Fargate, there is no need to provision and scale EC2 instances to run pods. Fargate will handle this automatically and run my pods in a secure, multi-tenant serverless infrastructure.

In this setup, the worker nodes reside in a private subnet that is not directly accessible from the public network by default. To address this, I intend to use an Ingress Controller instead of relying on the LoadBalancer type service. This decision is driven by the cost considerations associated with the elastic IP provided by the cloud provider, especially when multiple pods are running in different clusters.

Here's the proposed approach: First, deploy a pod inside a worker node and on top of it, deploy a service with either the ClusterIP or NodePort type. Subsequently, deploy an Ingress Controller, which continuously monitors Ingress manifests. When an Ingress manifest is deployed on top of the service manifest, the Ingress Controller initiates the rollout of an Application Load Balancer in the public subnet of our VPC. This load balancer then efficiently routes traffic from the external world to the pod located within the worker node in the private subnet.

So in summary:
* EKS cluster for easy Kubernetes
* Worker nodes in private subnets for security and leverage the serverless architecture for easy maintenance
* Ingress Controller and ALB for external access to private nodes
* Services and Ingresses define internal routes
* VPC handles subnets inter-connectivity behind the scenes
This architecture provides scale, security and availability out of the box for my apps on Kubernetes

## Creating a Cluster 
so to create this cluster I am planning to use eksctl and run the following commands to  create one.
```
eksctl create cluster --name k8s-cluster --region us-east-1 --fargate

```
Next Connect your local Kubernetes command line tool (kubectl) to your Amazon EKS cluster by running the following command.
```
aws eks update-kubeconfig --name k8s-cluster --region us-east-1
```
### Updating Fargate Profile
I intend to use Appgame-2048 as my namespace instead of using the default name space, just to create some isolation.
```
eksctl create fargateprofile \
    --cluster k8s-cluster \
    --region us-east-1 \
    --name alb-sample-app \
    --namespace game-2048


```
## Deploy the deployment, service and Ingress
Now we will deploy the appdeploy.yaml file inside the cluster using namespace game-2048. So this appdeploy.yaml file contains manifests for deployment, service and Ingress.

```
kubectl apply -f appdeploy.yaml
```
Once the aws eks update-kubeconfig command is run, a deployment is created with a replicaSet of 5, so 5 pods are created for high availability. These pods have the label "app.kubernetes.io/name: app-2048" for routing traffic to all pods with the same label.

Next, on top of this deployment, a Service of type NodePort named "service-2048" is created. It is listening on port 80 with a targetPort of 80, since our containerPort is defined as 80. This means that whenever traffic enters the Service on port 80, it forwards it to targetPort 80 of all containers with the "app.kubernetes.io/name: app-2048" label.

And on top of this Service, we will deploy an Ingress resource. The Ingress configuration is saying: "For any HTTP traffic coming to the root path (/), forward it to a Service named service-2048 in the default namespace on port 80. Additionally, use annotations for configuring an AWS Application Load Balancer with an internet-facing scheme and IP address as the target type."

