## Step - 1 : Create EKS Management Host in AWS ##

1) Launch new Ubuntu VM using AWS Ec2 ( t2.micro )	  
2) Connect to machine and install kubectl using below commands  
```
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin
kubectl version --short --client
```
3) Install AWS CLI latest version using below commands 
```
sudo apt install unzip
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

4) Install eksctl using below commands
```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```
## Step - 2 : Create IAM role & attach to EKS Management Host ##

1) Create New Role using IAM service ( Select Usecase - ec2 ) 	
2) Add below permissions for the role <br/>
	- Administrator - acces <br/>
		
3) Enter Role Name (eksroleec2) 
4) Attach created role to EKS Management Host (Select EC2 => Click on Security => Modify IAM Role => attach IAM role we have created) 

## Step - 3 : Create EKS Cluster using eksctl ## 
**Syntax:** 

eksctl create cluster --name cluster-name  \
--region region-name \
--node-type instance-type \
--nodes-min 2 \
--nodes-max 2 \ 
--zones <AZ-1>,<AZ-2>

## N. Virgina: <br/>
```
eksctl create cluster --name chaitra-cluster --region us-east-1 --node-type t2.medium  --zones us-east-1a,us-east-1b
```
## Mumbai: <br/>
```
eksctl create cluster --name chaitra-cluster --region ap-south-1 --node-type t2.medium  --zones ap-south-1a,ap-south-1b
```

## Note: Cluster creation will take 5 to 10 mins of time (we have to wait). After cluster created we can check nodes using below command.

```
 kubectl get nodes  
```

### Note: We should be able to see EKS cluster nodes here. ##

### We are done with our Setup ###

============================================================================================

# aws-eks-secret-csi-demo


### Lets Understand what is Secrets Store CSI Driver

```
The Secrets Store CSI Driver secrets-store.csi.k8s.io allows Kubernetes to mount multiple secrets, keys,
and certs stored in enterprise-grade external secrets stores into their pods as a volume.
Once the Volume is attached, the data in it is mounted into the container's file system.
```


### Install CSI drivers


# Secrets Store CSI Secret driver.

```
helm repo add secrets-store-csi-driver \
  https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
```

```
helm install -n kube-system csi-secrets-store \
  --set syncSecret.enabled=true \
  --set enableSecretRotation=true \
  --set rotationPollInterval=15s \
  secrets-store-csi-driver/secrets-store-csi-driver
```

# AWS Secrets and Configuration Provider

```
kubectl apply -f https://raw.githubusercontent.com/aws/secrets-store-csi-driver-provider-aws/main/deployment/aws-provider-installer.yaml
```

# Verify the installations

```
kubectl get daemonsets -n kube-system -l app=csi-secrets-store-provider-aws
kubectl get daemonsets -n kube-system -l app.kubernetes.io/instance=csi-secrets-store
```


### Create an IAM OIDC identity provider for your cluster.

```
eksctl utils associate-iam-oidc-provider --cluster secret-csi-cluster  --approve --region us-east-2
```

### Creating a policy where s3 bucket mentioned which has to be used by the pod. (aws-eks-secret-csi-demo)

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "secretsmanager:GetSecretValue",
                "secretsmanager:DescribeSecret"
            ],
            "Resource": "<arn of your secret>"
        }
    ]
}

```

```
aws iam create-policy --policy-name aws-eks-secret-csi-demo --policy-document file://policy.json
```

### trust-relationship.json

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "<arn of your oidc>"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "oidc.eks.us-east-2.amazonaws.com/id/<change it with your ID>:sub": "system:serviceaccount:ns:sa"
                }
            }
        }
    ]
}

```


### Creating a role and appending the above trust policy.

```
aws iam create-role --role-name role-aws-eks-secret-csi-demo --assume-role-policy-document file://trust-relationship.json --description "secret-csi role description"
```

### Attaching the role with the policy we created in above steps

```
aws iam attach-role-policy --role-name role-aws-eks-secret-csi-demo --policy-arn=<policy arn>
```

### Appending Annotations in the ServiceAccount we have to use.

```
kubectl annotate serviceaccount my-sa eks.amazonaws.com/role-arn=<role arn>
```

	
## Step - 4 : After your practise, delete Cluster and other resources we have used in AWS Cloud to avoid billing ##

```
eksctl delete cluster --name chaitra-cluster4 --region ap-south-1
```
