# Deploying a Highly Available/Production Ready Kubernetes Cluster in AWS

#### Requirements
- [AWS CLI](https://aws.amazon.com/cli/)
- [Kops](https://github.com/kubernetes/kops)
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
- [OpenSSL](https://www.openssl.org/)
- [Kubernetes Dashboard](https://github.com/kubernetes/dashboard)

#### Useful
- [kubectx & kubens](https://github.com/ahmetb/kubectx)

### Content
- [Installing AWS CLI](#installing-aws-cli)
- [Installing kops](#installing-kops)
- [Installing kubectl](#installing-kubectl)
- [Create kops user](#create-kops-user)
- [Create k8s cluster](#create-k8s-cluster)
- [Deploy k8s dashboard and heapster](#deploy-k8s-dashboard-and-heapster)
- [Grant access to your teammates](#grant-access-to-your-teammates)

## Installing AWS CLI

If you already have AWS CLI you can skip this section.

#### First let's get PIP
**NOTE:** Depending on your env maybe you should replace `.profile` with `.bash_profile`, or `.bash_login`.
```
curl -O https://bootstrap.pypa.io/get-pip.py;
python get-pip.py --user;
echo "export PATH=~/.local/bin:$PATH" | tee -a ~/.profile;
source ~/.profile;
pip --version;
```

#### Now we can proceed
```
pip install awscli --upgrade --user;
aws --version;
```

## Installing Kops

Kops is a tool that automates the creation and maintenance of the kubernentes cluster.
```
curl -Lo kops https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64;
chmod +x ./kops;
sudo mv ./kops /usr/local/bin/;
```

## Installing kubectl
We need kubectl in order to interect with the kubernetes cluster:

```
curl -Lo kubectl https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl;
chmod +x ./kubectl;
sudo mv ./kubectl /usr/local/bin/kubectl;
```

## Create kops user

export AWS_ACCESS_KEY_ID=$(aws configure get aws_access_key_id)
export AWS_SECRET_ACCESS_KEY=$(aws configure get aws_secret_access_key)
```
aws iam create-group --group-name kops;                                                                     # Create group for kops

aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess --group-name kops;     # Give permission for EC2
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonRoute53FullAccess --group-name kops; # Give permission for Route53
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --group-name kops;      # Give permission for S3
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/IAMFullAccess --group-name kops;           # Give permission for IAM
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonVPCFullAccess --group-name kops;     # Give permission for the VPC

aws iam create-user --user-name kops;                                                                       # Create user for kops
aws iam add-user-to-group --user-name kops --group-name kops;                                               # Add user kops to the group kops

aws iam create-access-key --user-name kops;                                                                 # Generate access key
```

You should record the SecretAccessKey and AccessKeyID in the returned JSON output, and then use them below:

```
# configure the aws client to use your new IAM user
aws configure           # Use your new access and secret key here
aws iam list-users      # you should see a list of all your IAM users here

# Because "aws configure" doesn't export these vars for kops to use, we export them now

echo "export AWS_ACCESS_KEY_ID=$(aws configure get aws_access_key_id)" | tee -a ~/.profile;
echo "export AWS_SECRET_ACCESS_KEY=$(aws configure get aws_secret_access_key)" | tee -a ~/.profile;

source ~/.profile;
```

## Create k8s cluster

#### Create S3 state store bucket

Put the name of your cluster and the state store in these variables:
```
export NAME=<YOUR_CLUSTER_NAME>
export KOPS_STATE_STORE=<YOUR_S3_STORE_BUCKET_NAME>
```

Create the state store bucket:
```
aws s3api create-bucket \
    --bucket prefix-example-com-state-store \
    --region us-east-1
    # --create-bucket-configuration LocationConstraint=<region> # for regions other than us-east-1.

# kops recomends using versioning
aws s3api put-bucket-versioning --bucket prefix-example-com-state-store  --versioning-configuration Status=Enabled
```

#### Create cluster

Create a HA cluster (you can find more information on these flags on kops documenetation):
```
kops create cluster \
   --master-count=3 \                                                          # number of master nodes
   --master-size t2.small \                                                    # size of the master instances
   --node-count 5 \                                                            # number of nodes
   --node-size t2.micro \                                                      # size of node instances
   --zones us-east-2a,us-east-2b,us-east-2c \                                  # availability zone to use for nodes
   --master-zones us-east-2a,us-east-2b,us-east-2c \                           # availability zone to use for master
   --cloud-labels "Client=GBH,PartOf=xhacluster,CreatedBy=Modesto Figuereo" \  # Some labels/tags
   ${NAME}
```

**NOTE: kops automatically creates a kubeconfig file (``~/.kube/config`) for you so you can interact with your cluster**

#### Add policy types to the cluster

Execute: `kops edit cluster --name ${NAME}``

Add this under the `spec` section:
```
  additionalPolicies:
    node: |
      [
        {
          "Effect": "Allow",
          "Action": [
             "acm:ListCertificates",
             "acm:GetCertificate",
             "acm:DescribeCertificate",
             "autoscaling:DescribeAutoScalingGroups",
             "autoscaling:DescribeLoadBalancerTargetGroups",
             "autoscaling:AttachLoadBalancers",
             "autoscaling:DetachLoadBalancers",
             "autoscaling:DetachLoadBalancerTargetGroups",
             "autoscaling:AttachLoadBalancerTargetGroups",
             "cloudformation:*",
             "elasticloadbalancing:*",
             "elasticloadbalancingv2:*",
             "ec2:DescribeInstances",
             "ec2:DescribeSubnets",
             "ec2:DescribeSecurityGroups",
             "ec2:DescribeRouteTables",
             "ec2:DescribeVpcs",
             "ec2:DescribeAccountAttributes",
             "ec2:DescribeInternetGateways",
             "iam:GetServerCertificate",
             "iam:ListServerCertificates",
             "iam:CreateServiceLinkedRole",
             "route53:GetHostedZone",
             "route53:ListHostedZonesByName",
             "route53:CreateHostedZone",
             "route53:DeleteHostedZone",
             "route53:ChangeResourceRecordSets",
             "route53:CreateHealthCheck",
             "route53:GetHealthCheck",
             "route53:DeleteHealthCheck",
             "route53:UpdateHealthCheck",
             "route53:ListHostedZones",
             "route53:ListResourceRecordSets",
             "route53:ChangeResourceRecordSets",
             "ec2:DescribeVpcs",
             "ec2:DescribeRegions",
             "servicediscovery:*"
          ],
          "Resource": ["*"]
        }
      ]
```

Apply the changes with `kops update cluster --name ${NAME} --yes`.

**NOTE: you can tweak the details of you instance group by executing (for example, min and max nodes for auto scaling) `kops edit ig <instance group name>`**


## Deploy k8s dashboard and heapster

#### Kubernetes Dashboard

Kubernetes Dashboard is a handy tool for visualizing and performing basic tasks in your cluster.

To deploy Kubernetes Dashboard is as easy as:
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
```

To enter the dashboard execute: `kubectl proxy`. <br />
Now access the url: `http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/`.

#### Create an admin user for your dashboard

For accessing the dashboard we need an user with the right permissions, here I will create an admin user but you can create users with more restrictions if you want to give access to your teammates.

Using the admin user yaml inside the `users/` directory we can create the user and grant them admin privileges as follows:
```
kubectl apply -f ./user/admin-user-service-account.yaml
```

Now we need the token for that user in other to login to the dashboard, we can get it with this command:
```
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```

This will print something like this:
```
Name:         admin-user-token-6gl6l
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name=admin-user
              kubernetes.io/service-account.uid=b16afba9-dfec-11e7-bbb9-901b0e532516

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLTZnbDZsIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJiMTZhZmJhOS1kZmVjLTExZTctYmJiOS05MDFiMGU1MzI1MTYiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.M70CU3lbu3PP4OjhFms8PVL5pQKj-jj4RNSLA4YmQfTXpPUuxqXjiTf094_Rzr0fgN_IVX6gC4fiNUL5ynx9KU-lkPfk0HnX8scxfJNzypL039mpGt0bbe1IXKSIRaq_9VW59Xz-yBUhycYcKPO9RM2Qa1Ax29nqNVko4vLn1_1wPqJ6XSq3GYI8anTzV8Fku4jasUwjrws6Cn6_sPEGmL54sq5R4Z5afUtv-mItTmqZZdxnkRqcJLlg2Y8WbCPogErbsaCDJoABQ7ppaqHetwfM_0yMun6ABOQbIwwl8pspJhpplKwyo700OSpvTT9zlBsu-b35lzXGBRHzv5g_RA
```

We can copy the value for token and pasted in the login screen in the Dashboard. <br />
That's it! Now you have visualization of all your resources in the cluster.

#### Deploy Heapster

TBD.

## Grant access to your teammates

#### Generate the certificates

Copy the key and certificates from the kops configuration:
```
export KEY=<KEY_FILENAME>.key;
export CRT=<CRT_FILENAME>.crt;
export MASTER_CRT=<CRT_FILENAME>.crt;
export USERNAME=<USERNAME>;

aws s3 cp s3://${KOPS_STATE_STORE}/xhacluster.gbhplayground.com/pki/private/ca/${KEY} ca.keY;
aws s3 cp s3://${KOPS_STATE_STORE}/xhacluster.gbhplayground.com/pki/issued/ca/${CRT} ca.crt;
aws s3 cp s3://${KOPS_STATE_STORE}/xhacluster.gbhplayground.com/pki/issued/master/${MASTER_CRT} ca_master.crt
```

Generate certificates for the user:
```
openssl genrsa -out ${USERNAME}.key 4096
openssl req -new -key ${USERNAME}.key -out ${USERNAME}.csr -subj '/CN=${USERNAME}/O=devops'
openssl x509 -req -in ${USERNAME}.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out ${USERNAME}.crt -days 365
```

Bind the view role to the user:
```
kubectl create clusterrolebinding viewer-cluster-admin-binding --clusterrole=view --user=${USERNAME} -n <NAMESPACE>
```

#### Create a kubeconfig for the user

**NOTE: After generating the config file send it through a secure channel the your teammates.** <br />
**NOTE: You can generate different kubeconfig files for each of your teammates with different roles and for different namespaces.**
```
kubectl config set-cluster xhacluster.gbhplayground.com \
  --server=https://api.xhacluster.gbhplayground.com \
  --certificate-authority=./ca_master.crt \
  --embed-certs=true \
  --kubeconfig=~/path/to/config

kubectl config set-credentials xhacluster.gbhplayground.com \
  --client-certificate=./${USERNAME}.crt \
  --client-key=./${USERNAME}.key \
  --embed-certs=true \
  --kubeconfig=~/path/to/config

kubectl config set-context xhacluster.gbhplayground.com \
  --cluster=xhacluster.gbhplayground.com \
  --user=xhacluster.gbhplayground.com \
  --kubeconfig=~/path/to/config     
```

- @TODO: Add section for kube-ingress-aws-controller deployment
- @TODO: Add section for skipper deployment
- @TODO: Add section for external-dns deployment
- @TODO: Add section for Jenkins deployment
