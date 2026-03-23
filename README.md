# Deploying AWS Load Balancer Controller on Amazon EKS with Fargate - A Kubernetes Multi-Tenancy Ingress Setup

Deploying applications on Amazon EKS Fargate provides a truly serverless Kubernetes experience — eliminating the need to manage EC2 worker nodes, perform OS patching, or handle infrastructure scaling.

This guide demonstrates how to set up this architecture in a cost-efficient and production-ready manner by integrating ingress capabilities for controlled external access and traffic management.

You will walk through the complete process of provisioning an EKS Fargate cluster, configuring the AWS Load Balancer Controller using IAM Roles for Service Accounts (IRSA) for secure, least-privilege access, and deploying two real-world applications behind a shared Application Load Balancer (ALB) using path-based routing.

By the end, you will have a scalable, resilient, and operationally simplified Kubernetes deployment pattern suitable for modern cloud-native workloads.

# Tools:

-	Container Orchestration - Kubernetes
-	Ingress Controller - AWS Load Balancer Controller
-	Serverless Archtechture - AWS Fargate

---

## Architecture Overview

Before diving in, here is what we are building:

-	An EKS cluster (ingress-demo-cluster) in us-east-1, running entirely on Fargate
-	Three Fargate profiles: fp-default (default + kube-system), focus-app-profile, world-clock-profile
-	Two applications in isolated namespaces: world-clock and focus-app
-	A single shared ALB routing traffic by URL path: /clock/ and /focus/
-	The AWS Load Balancer Controller running in kube-system, managing ALB lifecycle

A key architectural decision in this design is implementing path-based routing across multiple namespaces using a single Application Load Balancer (ALB).

Provisioning a dedicated ALB for every service or namespace can quickly become inefficient and expensive. Instead, by leveraging shared ingress patterns such as an IngressGroup or coordinated Ingress resources, applications running in different namespaces can utilize the same load balancer while maintaining clear and isolated routing rules.

With this approach, each application is exposed through distinct URL paths, allowing the ALB to forward traffic to the correct backend service based on the request path. This not only reduces infrastructure cost but also simplifies load balancer management and promotes a scalable, multi-tenant cluster design.

Such shared ingress architectures are widely adopted in production Kubernetes environments because they strike the right balance between cost optimization, operational simplicity, and traffic isolation.

---

## Step 1: Create the EKS Fargate Cluster

We will be provisioning this cluster through the aws cli allowing eksctl to handle a significant amount of infrastructure creation under the hood: it creates a CloudFormation stack, provisions a VPC with public and private subnets across two availability zones, configures the EKS control plane, and sets up the default Fargate profile. 

```
eksctl create cluster \
  --name ingress-demo-cluster \
  --region us-east-1 \
  --fargate
```

![Kubernetes EKS Fargate Cluster is Ready](images/create_cluster.png)

It took about 15 minutes for the cluster to be ready so be ready to give it some time.

---

![AWS EKS Dashboard](images/AWS_EKS_dashboard.png)

---

## Verify the Cluster
Once creation completes, verify the cluster is ready and that Fargate nodes have registered. Note that the Fargate nodes are AWS managed nodes which are different from EC2 instances. You do not have access to manage node configuration and updates on Fargate since they are managed by AWS.

```
kubectl get nodes
```
![Kubernetes Cluster is Ready](images/cluster_ready.png)

---

Step 2: Create an IAM OIDC Provider
The controller needs permission to create an ALB. First, associate your cluster with an IAM OIDC provider.

________________________________________________
1. The Cluster Role vs. Pod Permissions
The IAM Role ARN you provided (...ServiceRole...) is the EKS Cluster Role. This role gives the AWS Control Plane permission to manage resources like VPCs and Security Groups on your behalf.

However, the Ingress Controller runs as a Pod inside your cluster.

By default, a Pod has no identity in AWS IAM.

The Ingress Controller needs to perform specific actions (like elasticloadbalancing:CreateLoadBalancer) that the Cluster Role doesn't necessarily cover.

We use IRSA (IAM Roles for Service Accounts) so that the Pod itself can "assume" a role, rather than giving the entire cluster wide-open permissions.

2. Why "Associate IAM OIDC Provider"?
Even though your cluster has an OIDC URL, AWS IAM doesn't "trust" it by default.
Running eksctl utils associate-iam-oidc-provider creates an IAM Identity Provider entry in your AWS Console. This essentially tells AWS IAM: "If you see a request from a Pod in this specific EKS cluster, you are allowed to validate its identity using this OIDC URL."

3. Why Create a New Role?
The Ingress Controller needs a very specific set of permissions (defined in that iam_policy.json we downloaded earlier).

The Least Privilege Principle: You don't want to attach "Create Load Balancer" permissions to your main EKS Cluster Role.

Scoped Access: By creating a dedicated role for the controller, you ensure that if that specific Pod is compromised, the attacker only has access to Load Balancer settings, not your entire cluster or VPC.

Summary of the "Trust Chain"
OIDC Provider: Establishes trust between your Cluster and AWS IAM.

IAM Role: Defines what the controller is allowed to do (LB management).

Service Account: Links the Pod inside Kubernetes to that IAM Role using the OIDC bridge.
_______________________________________________________

```
eksctl utils associate-iam-oidc-provider \
    --region us-east-1 \
    --cluster ingress-demo-cluster \
    --approve
```

### Step 2: Create the IAM Policy
Download and create the IAM policy that allows the controller to manage AWS resources.

**Create the IAM policy using the iam_policy.json file in the repository (see alternative below):**
    ```
    aws iam create-policy \
        --policy-name AWSLoadBalancerControllerIAMPolicy \
        --policy-document file://iam_policy.json
    ```
    *Remember to copy the ARN returned by this command (e.g., `arn:aws:iam::111122223333:policy/AWSLoadBalancerControllerIAMPolicy`).*


# Alternatively, if preferred

Download the official IAM policy that allows the controller to manage AWS resources. 
    
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json

then - aws iam create-policy \
        --policy-name AWSLoadBalancerControllerIAMPolicy \
        --policy-document file://iam_policy.json

You might need to review permissions and ensure Allow is enabled on "elasticloadbalancing:SetRulePriorities"

### Step 3: Create a Service Account
Use `eksctl` to create a Kubernetes Service Account and map it to the IAM policy created above.

```bash
eksctl create iamserviceaccount \
  --cluster=ingress-demo-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \ #add your ACCOUNT_ID
  --approve
```

### Step 4: Install the Controller using Helm
Add the EKS chart repository and install the controller. On Fargate, we specifically tell Helm to use the Service Account we just created.

1.  **Add the Repo:**
    ```bash
    helm repo add eks https://aws.github.io/eks-charts
    helm repo update
    ```
2.  **Install the Chart:**
    ```bash
    helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
      -n kube-system \
      --set clusterName=<CLUSTER_NAME> \
      --set serviceAccount.create=false \
      --set serviceAccount.name=aws-load-balancer-controller \
      --set region=<REGION> \
      --set vpcId=<VPC_ID>
    ```

---

### Step 5: Verify the Installation
Check that the deployment is running. Because you are on Fargate, it may take a minute for the pod to transition from `Pending` to `Running`.

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```
eksctl get fargateprofile --cluster ingress-demo-cluster

```

#for fargate, you need to create the profile and namespace to be able to create namespace in your ccluster

eksctl create fargateprofile \
    --cluster ingress-demo-cluster \
    --name world-clock-profile \
    --namespace world-clock
   

eksctl create fargateprofile \
    --cluster ingress-demo-cluster \
    --name focus-app-profile \
    --namespace focus-app
```

### Important Fargate Considerations
* **Target Type:** When you create an Ingress resource, you **must** use the annotation `alb.ingress.kubernetes.io/target-type: ip`. Fargate does not support `instance` mode because there are no EC2 instances to route to.
* **Subnet Tagging:** Ensure your private subnets have the tag `kubernetes.io/role/internal-elb: 1` and your public subnets have `kubernetes.io/role/elb: 1` so the controller can auto-discover them.

# create namespaces

# create deployment
# create services

# create ingress x2

kubectl get ingress <ingress-name> -n <namespace>


------------------------------------------
## Delete the cluster

```
eksctl delete cluster --name demo-cluster --region us-east-1
```