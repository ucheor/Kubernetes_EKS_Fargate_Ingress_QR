## System Requirements & Setup

To manage your Amazon EKS environment, ensure the following tools are installed and configured:

- kubectl: The standard CLI for interacting with Kubernetes. Refer to the [Official Installation Guide]("https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html") for setup instructions.

- AWS CLI: Required for managing AWS resources. Follow the [AWS CLI Installation Guide]("https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html"), and ensure you run "aws configure" to set up your credentials.

```aws configure```

- eksctl: A dedicated CLI tool that simplifies EKS cluster management and automation. See the [eksctl setup guide]("https://docs.aws.amazon.com/eks/latest/eksctl/installation.html").

- Helm installed. See instructions [here]("https://helm.sh/docs/intro/install")

# Cluster Authentication

While creating a cluster usually updates your kubeconfig automatically, you can manually point kubectl to your new cluster using the AWS CLI:


```aws eks update-kubeconfig --name <cluster-name> --region <region>```


Validation: Confirm the connection by listing your active nodes:


```kubectl get nodes```

