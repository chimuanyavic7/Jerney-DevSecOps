## **Setup Instruction:**

## **Prerequisites**
- AWS Account
- use Terraform or Bastion


### **Step 1 Lunch Bastion Host:**
- What is Bastion Host ?
  - The Server through we will control our Cluster, This EC2 will not be the part of our cluster
- Lunch `T2.micro`
- Update EC2 `sudo apt update`

### **Step 2 Install AWS CLI and Configure:**

- Install AWS CLI command

```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install
```

- Configure AWS
  - Go to AWS IAM and create User
  - create `Access Key` and `Secret Access Key`
  - Then use this command `AWS configure`
  - it will ask for your accesskey and sceret accesskey, copy past them
  - now your Bastion Host is configured with your AWS

### **Step 3 Install kubectl ,esksctl and Helm:**

- kubectl install command:

```
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin
kubectl version --short --client
```

- eksctl install command

```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

- Helm install command

```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

### **Step 4 Create Cluster:**

- Create EKS Cluster

```bash
eksctl create cluster --name=my-cluster \
                      --region=ap-south-1 \
                      --version=1.34 \
                      --without-nodegroup
```

- Associate IAM OIDC Provider

```bash
eksctl utils associate-iam-oidc-provider \
    --region ap-south-1 \
    --cluster my-cluster \
    --approve
```

#

- Create Nodegroup

```bash
eksctl create nodegroup --cluster=my-cluster \
                       --region=ap-south-1 \
                       --name=my-cluster \
                       --node-type=t2.medium \
                       --nodes=2 \
                       --nodes-min=2 \
                       --nodes-max=2 \
                       --node-volume-size=25 \
                       --ssh-access \
                       --ssh-public-key=ec2_keypair
```

#### Note: Make sure the ssh-public-key "ec2_keypair is available in your aws account"

#

- Update Kubectl Context

```bash
aws eks update-kubeconfig --region ap-south-1 --name my-cluster
```

#

- Delete EKS Cluster (Once you are done)

```bash
eksctl delete cluster --name=my-cluster --region=ap-south-1
```

### **Step 5 Install AWS EBS CSI Driver:**

- We are Installing AWS EBS CSI Driver because we are storing data in EBS, so that data is persisted across cluster
- Add the AWS EBS CSI Driver Helm chart repository:

```
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm repo update
```

- Identify Node Group IAM Role:

```
aws eks describe-nodegroup \
  --cluster-name <your-cluster-name> \
  --nodegroup-name <your-nodegroup-name> \
  --query "nodegroup.nodeRole" --output text

```

- this will written something like this
  `arn:aws:iam::123456789012:role/eksctl-my-cluster-nodegroup-NodeInstanceRole
`

- Attach IAM Policy for EBS CSI

```
aws iam attach-role-policy \
  --role-name <your-node-role-name> \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy
```

**Note:** Replace `your-node-role-name` with the actual IAM role name from previous step

- Install the EBS CSI Driver via Helm

```
helm install aws-ebs-csi-driver/aws-ebs-csi-driver \
  --name-template aws-ebs-csi-driver \
  --namespace kube-system

```

- verify Installation
  `kubectl get pods -n kube-system | grep ebs
`

---
 ## **Using Hashicorp vault for managing our secrets in Kubernetes**
  - Install hashicorp vault on the cluster
  
  ```
    helm repo add hashicorp https://helm.releases.hashicorp.com/
    helm repo update
    helm install vault hashicorp/vault --set "server.dev.enabled=true"
       
  ```
  
  ```
    helm repo add external-secrets https://charts.external-secrets.io
    helm repo update
    helm install external-secrets external-secrets/external-secrets --namespace external-secrets --create-namespace --set installCRDs=true   
  ```

  - Install EBS CSI driver
  
  ```
    aws iam attach-role-policy \
        --role-name <NodeInstanceRoleName> \
        --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy
       
  ```

  ```
    kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.44"
       
  ```

  - Create a secret for vault-token
  ```
    kubectl create secret generic vault-token \
        --namespace default \
        --from-literal=token=root
       
  ```
- change vault to Loadbalaner: password is root 
```bash
kubectl patch svc vault -n default -p '{"spec": {"type": "LoadBalancer"}}'
````


---
## **Install ArgoCD**


- we are adding this because so that routing is done to apppropitate service
```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
````

- change AgroCD to Loadbalaner
```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
````

- To AgroCD secret password  
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo'
````


---
 ## **Monitoring with Prometheus and Grafana**
  
  - Install kube-prometheus-stack
  ```
    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    helm repo update
       
  ```
  
  - Install kube-prometheus-stack
  ```
    kubectl create ns monitoring
    helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring 
       
  ```

  - Prometheus UI:
  ```
     kubectl port-forward service/prometheus-operated -n monitoring 9090:9090
       
  ```

  - Grafana UI: 
  ```
     kubectl port-forward service/monitoring-grafana -n monitoring 8080:80
       
  ```
  - Fetch the Grafana password Grafana: 
  ```
     kubectl get secret

     kubectl get secret --namespace default monitoring-grafana -o jsonpath="{.data.admin-password}" | base64 --decode; echo
    
  ```
  **NOTE:** If you are using an EC2 Instance or Cloud VM, you need to pass --address 0.0.0.0 to the above command. Then you can access the UI on instance-ip:port

    - Open ports of Bastion Host port: 9090and 8080
    - Access Prometheus <BastionHostIP>:9090 
    - Access Grafana <BastionHostIP>:8080
    - Or change to LoadBalancer

 ### **Dashboard for Kubernetes monitoring:**
    - 15760
    - 1860
    

### **Install AWS Load Balancer Controller:**
- Edit Cluster-name and region
```
    export CLUSTER_NAME="CLUSTER-NAME"
    export AWS_REGION="REGION-NAME"
    export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
    
```

```
# 1. Ensure the cluster has an IAM OIDC provider
eksctl utils associate-iam-oidc-provider \
  --cluster "$CLUSTER_NAME" \
  --region "$AWS_REGION" \
  --approve

# 2. Download the official IAM policy
curl -Lo iam_policy.json \
  https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.14.1/docs/install/iam_policy.json

# 3. Create the IAM policy
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json

# 4. Create the dedicated IAM role + Kubernetes ServiceAccount
eksctl create iamserviceaccount \
  --cluster "$CLUSTER_NAME" \
  --region "$AWS_REGION" \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn \
    "arn:aws:iam::${AWS_ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy" \
  --approve \
  --override-existing-serviceaccounts

# 5. Add the official AWS EKS Helm repository
helm repo add eks https://aws.github.io/eks-charts
helm repo update

# 6. Get the VPC ID
export VPC_ID="$(aws eks describe-cluster \
  --name "$CLUSTER_NAME" \
  --region "$AWS_REGION" \
  --query 'cluster.resourcesVpcConfig.vpcId' \
  --output text)"

# 7. Install a pinned controller version
helm upgrade --install aws-load-balancer-controller \
  eks/aws-load-balancer-controller \
  --namespace kube-system \
  --version 1.14.0 \
  --set clusterName="$CLUSTER_NAME" \
  --set region="$AWS_REGION" \
  --set vpcId="$VPC_ID" \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --wait \
  --timeout 10m
       
  ```


- **Another meethod for Install AWS Load Balancer Controller but i prefer the first method**

- Create IAM Policy:
```
    curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.13.3/docs/install/iam_policy.json
    
```
 
```
  aws iam create-policy \
     --policy-name AWSLoadBalancerControllerIAMPolicy \
     --policy-document file://iam_policy.json
    
```

- Create an IAM-Backed Kubernetes Service Account
```
  eksctl create iamserviceaccount \
    --cluster=wanderblog-eks-cluster \
    --namespace=kube-system \
    --name=aws-load-balancer-controller \
    --attach-policy-arn=arn:aws:iam::<Account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
    --override-existing-serviceaccounts \
    --region ap-south-1 \
    --approve
    
```

- **Get External IP of NGINX Ingress**

  ```bash
  kubectl get svc -n ingress-nginx
  nslookup <external-ip>
  ```

- **Update `/etc/hosts`**
  Add the following line to your `/etc/hosts` file:

```
<external-ip> your-app-domain.com
```

- **Access the Application**
  Open your browser and navigate to:

```
http://your-app-domain.com
```

---