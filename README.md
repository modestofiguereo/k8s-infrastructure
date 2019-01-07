# Deploying a Highly Available/Production Ready Kubernetes Cluster On AWS

#### Requirements
- [AWS CLI](https://aws.amazon.com/cli/)
- [Kops](https://github.com/kubernetes/kops)
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
- [OpenSSL](https://www.openssl.org/)

#### Useful
- [kubectx & kubens](https://github.com/ahmetb/kubectx)
- [Kompose](https://github.com/kubernetes/kompose)

### Content
- [Installing AWS CLI](#installing-aws-cli)
- [Installing kops](#installing-kops)
- [Installing kubectl](#installing-kubectl)
- [Create kops user](#create-kops-user)
- [Create k8s cluster](#create-k8s-cluster)
- [Deploy kube ingress aws controller](#deploy-kube-ingress-aws-controller)
- [Deploy k8s dashboard and heapster](#deploy-k8s-dashboard-and-heapster)
- [Deploy jenkins](#deploy-jenkins)
- [Deploy Gitlab](#deploy-gitlab)
- [Grant access to your teammates](#grant-access-to-the-cluster-to-your-teammates)

## Installing AWS CLI

If you already have AWS CLI you can skip this section.

#### First let's get PIP
**NOTE:** Depending on your env maybe you should replace `.profile` with `.bash_profile`, or `.bash_login`.
```
curl -O https://bootstrap.pypa.io/get-pip.py
python get-pip.py --user
echo "export PATH=~/.local/bin:$PATH" | tee -a ~/.profile
source ~/.profile
pip --version
```

#### Now we can proceed
```
pip install awscli --upgrade --user
aws --version
```

## Installing Kops

Kops is a tool that automates the creation and maintenance of the kubernentes cluster.
```
curl -Lo kops https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64
chmod +x ./kops
sudo mv ./kops /usr/local/bin/
```

## Installing kubectl
We need kubectl in order to interect with the kubernetes cluster:

```
curl -Lo kubectl https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
```

## Create kops user

```
export AWS_ACCESS_KEY_ID=$(aws configure get aws_access_key_id)
export AWS_SECRET_ACCESS_KEY=$(aws configure get aws_secret_access_key)

aws iam create-group --group-name kops                                                                     # Create group for kops

aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess --group-name kops     # Give permission for EC2
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonRoute53FullAccess --group-name kops # Give permission for Route53
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --group-name kops      # Give permission for S3
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/IAMFullAccess --group-name kops           # Give permission for IAM
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonVPCFullAccess --group-name kops     # Give permission for the VPC

aws iam create-user --user-name kops                                                                       # Create user for kops
aws iam add-user-to-group --user-name kops --group-name kops                                               # Add user kops to the group kops

aws iam create-access-key --user-name kops                                                                 # Generate access key
```

You should record the SecretAccessKey and AccessKeyID in the returned JSON output, and then use them below:

```
# configure the aws client to use your new IAM user
aws configure           # Use your new access and secret key here
aws iam list-users      # you should see a list of all your IAM users here

# Because "aws configure" doesn't export these vars for kops to use, we export them now

echo "export AWS_ACCESS_KEY_ID=$(aws configure get aws_access_key_id)" | tee -a ~/.profile
echo "export AWS_SECRET_ACCESS_KEY=$(aws configure get aws_secret_access_key)" | tee -a ~/.profile

source ~/.profile
```

## Create k8s cluster

#### Create S3 state store bucket

Put the name of your cluster and the state store in these variables:
```
export NAME=<YOUR_CLUSTER_NAME>
export KOPS_STATE_STORE_NAME=prefix-example-com-state-store
export KOPS_STATE_STORE=s3://${KOPS_STATE_STORE_NAME}
```

Create the state store bucket:
```
aws s3api create-bucket \
    --bucket ${KOPS_STATE_STORE_NAME} \
    --region us-east-1
    # --create-bucket-configuration LocationConstraint=<region> # for regions other than us-east-1.

# kops recomends using versioning
aws s3api put-bucket-versioning --bucket ${KOPS_STATE_STORE_NAME}  --versioning-configuration Status=Enabled
```

#### Create cluster

Create a HA cluster (you can find more information on these flags on kops documenetation):
```
kops create cluster \
   --master-count=3 \
   --master-size t2.small \
   --node-count 5 \                                                       
   --node-size t2.micro \
   --zones us-east-2a,us-east-2b,us-east-2c \
   --master-zones us-east-2a,us-east-2b,us-east-2c \
   --cloud-labels "kubernetes.io/cluster/${NAME}: owned" \
   ${NAME}
```

**NOTE: kops automatically creates a kubeconfig file (``~/.kube/config`) for you so you can interact with your cluster**

#### Add policy types to the cluster

Execute: `kops edit cluster --name ${NAME}`

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

Apply the changes with `kops update cluster --name ${NAME} --yes`. After that the AWS is going to take sometime to set up the cluster. We can use the command `kops validate cluster` to know when the cluster is ready.

**NOTE: you can tweak the details of you instance group by executing (for example, min and max nodes for auto scaling) `kops edit ig <instance group name>`**

## Deploy kube ingress aws controller

Kube Ingress AWS Controller together with Skipper and External-DNS make life easier when dealing routing traffic to your services through ingress resources. As soon as we create an ingress resource they create the DNS record in AWS for us and route the traffic in the ports and paths that are specified in the ingress resource.

#### Service Accounts

Before deploying any of the above mentioned resources we need to create the service account the will give them permission to perform. We can do that with the following commands:
```
kubectl apply -f ./kube-ingress-aws-controller/skipper-aws-ingress-service-account.yaml
kubectl apply -f ./kube-ingress-aws-controller/external-dns/external-dns-service-account.yaml
```

#### kube-ingress-aws-controller

###### Security Group
Before deploying the kube-ingress-aws-controller we need to create an special security group for it:

```
KOPS_CLUSTER_NAME=${NAME}

VPC_ID=$(aws ec2 describe-security-groups --filters Name=group-name,Values=nodes.${KOPS_CLUSTER_NAME} | jq '.["SecurityGroups"][0].VpcId' -r)

aws ec2 create-security-group --description ingress.${KOPS_CLUSTER_NAME} --group-name ingress.${KOPS_CLUSTER_NAME} --vpc-id $VPC_ID
aws ec2 describe-security-groups --filter Name=vpc-id,Values=$VPC_ID  Name=group-name,Values=ingress.${KOPS_CLUSTER_NAME}

SG_ID_INGRESS=$(aws ec2 describe-security-groups --filter Name=vpc-id,Values=$VPC_ID  Name=group-name,Values=ingress.${KOPS_CLUSTER_NAME} | jq '.["SecurityGroups"][0]["GroupId"]' -r)
SG_ID_NODE=$(aws ec2 describe-security-groups --filter Name=vpc-id,Values=$VPC_ID  Name=group-name,Values=nodes.${KOPS_CLUSTER_NAME} | jq '.["SecurityGroups"][0]["GroupId"]' -r)

aws ec2 authorize-security-group-ingress --group-id $SG_ID_INGRESS --protocol tcp --port 443 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $SG_ID_INGRESS --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-egress --group-id $SG_ID_INGRESS --protocol all --port -1 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $SG_ID_NODE --protocol all --port -1 --source-group $SG_ID_INGRESS
aws ec2 create-tags --resources $SG_ID_INGRESS --tags Key="kubernetes.io/cluster/${KOPS_CLUSTER_NAME}",Value="owned" Key="kubernetes:application",Value="kube-ingress-aws-controller"
```

###### AWS Certificate Manager (ACM)

To have TLS termination you can use AWS managed certificates. If you are unsure if you have at least one certificate provisioned use the following command to list ACM certificates:

`aws acm list-certificates`

If you have one, you can move on to the next section.

To create an ACM certificate, you have to request a CSR with a domain name that you own in route53, for example.org. We will here request one wildcard certificate for example.org:

`aws acm request-certificate --domain-name *.example.org`

You will have to successfully do a challenge to show ownership of the given domain. In most cases you have to click on a link from an e-mail sent by certificates.amazon.com. E-Mail subject will be Certificate approval for <example.org>.

If you did the challenge successfully, you can now check the status of your certificate. Find the ARN of the new certificate:

`aws acm list-certificates`

Describe the certificate and check the Status value:

`aws acm describe-certificate --certificate-arn arn:aws:acm:<snip> | jq '.["Certificate"]["Status"]'`

If this is no "ISSUED", your certificate is not valid and you have to fix it. To resend the CSR validation e-mail, you can use:

`aws acm resend-validation-email`

###### Deploy the ingress controller

Now we can deploy kube-ingress-aws-controller with the following commands:

```
AWS_REGION=[AWS REGION TO USE]   # Assign the name of the region you want to use this variable
DOMAIN=[YOUR DOMAIN]             # Assign your domain name to this variable

sed -i "s/<AWS_REGION>/$AWS_REGION/" ./kube-ingress-aws-controller/kube-ingress-aws-controller-deployment.yaml
sed -i "s/<DOMAIN>/$DOMAIN/" ./kube-ingress-aws-controller/external-dns/external-dns-deployment.yaml
sed -i "s/<AWS_REGION>/$AWS_REGION/" ./kube-ingress-aws-controller/external-dns/external-dns-deployment.yaml

kubectl apply -f ./kube-ingress-aws-controller/kube-ingress-aws-controller-deployment.yaml
```

#### Skipper

Let's deploy skipper:
```
kubectl apply -f ./kube-ingress-aws-controller/skipper/skipper-deployment.yaml
```

#### External-dns

Finally Let's deploy external-dns:
```
kubectl apply -f ./kube-ingress-aws-controller/external-dns/external-dns-deployment.yaml
```

## Deploy k8s dashboard and heapster

#### Kubernetes Dashboard

Kubernetes Dashboard is a handy tool for visualizing and performing basic tasks in your cluster.

To deploy Kubernetes Dashboard is as easy as:
```
kubectl apply -f ./kubernetes-dashboard/kubernetes-dashboard.yaml
```

To enter the dashboard execute: `kubectl proxy`. <br />
Now you can access the dashboard in this url: [`http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/`](`http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/`).

#### Create an admin user for your dashboard

For accessing the dashboard we need an user with the right permissions, here I will create an admin user but you can create users with more restrictions if you want to give access to your teammates.

Using the admin user yaml inside the `users/` directory we can create the user and grant them admin privileges as follows:
```
kubectl apply -f ./kubernetes-dashboard/admin-user-service-account.yaml
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

Heapster is a tool that allow you to visualize the cpu and memory consumption of your pods and nodes.
You can set it up by executing:

```
DOMAIN=<DOMAIN>         # Assign your domain.

sed -i "s/<DOMAIN>/$DOMAIN/" ./kubernetes-dashboard/heapster/grafana.yaml

kubectl apply -f ./kubernetes-dashboard/heapster/influxdb.yaml
kubectl apply -f ./kubernetes-dashboard/heapster/grafana.yaml
kubectl apply -f ./kubernetes-dashboard/heapster/heapster.yaml
```

After Heapster is deployed it gets something in order to get metrics to display, after that you should get some charts in your dashboard displaying cpu and memory metrics.

## Deploy Jenkins

To deploy jenkins we need to build the image from the `jenkins/docker/jenkins.Dockerfile`, then push it to docker hub or our private registry.

After that we can perform the deployment:

```
IMAGE=<IMAGE_NAME>      # Assign the image name to this variable. (If you choosed to use a private repo, remember to prepend the name of the registry before the image name, i.e: registry/image-name:tag)
DOMAIN=<DOMAIN>         # Assign your domain.

sed -i "s/<IMAGE_NAME>/$IMAGE/" ./jenkins/jenkins-deployment.yaml
sed -i "s/<DOMAIN>/$DOMAIN/" ./jenkins/jenkins-ingress.yaml

kubectl apply -f ./jenkins/jenkins-namespace.yaml
kubectl apply -f ./jenkins/jenkins-service-account.yaml
kubectl apply -f ./jenkins/jenkins-persistent-volumen-claim.yaml
kubectl apply -f ./jenkins/jenkins-deployment.yaml
kubectl apply -f ./jenkins/jenkins-service.yaml
kubectl apply -f ./jenkins/jenkins-ingress.yaml
```

After jenkins is deployed we need to do some manual configuration:
- First of all enable security `Manage Jenkins -> Configure Global Security -> Enable Security`, then choose the options that better fits you and hit save.
- Add kubernetes cloud `Manage Jenkins -> Configure System -> Add Cloud`, fill the options.

@TODO: Insert here an screenshot as example on how to fill those options.

## Deploy Gitlab
Manifests to deploy GitLab on Kubernetes

##### Bump versions

* gitlab-runner:alpine-v11.6.0
* gitlab:11.6.2-ce.0
* postgresql:9.5.15
* redis:5.0.3 (official redis container)

#### Deploying GitLab itself
```
DOMAIN=<DOMAIN>                      # Assign your domain.

sed -i "s/<DOMAIN>/$DOMAIN/" ./gitlab/gitlab/gitlab-ingress.yaml

# create gitlab namespace
kubectl create -f ./gitlab/gitlab-namespace.yaml

# deploy redis
kubectl create -f ./gitlab/redis/redis-persistent-volume-claim.yaml
kubectl create -f ./gitlab/redis/redis-deployment.yaml
kubectl create -f ./gitlab/redis/redis-service.yaml

# deploy postgres
kubectl create -f ./gitlab/postgres/postgres-persistent-volume-claim.yaml
kubectl create -f ./gitlab/postgres/postgres-secret.yaml
kubectl create -f ./gitlab/postgres/postgres-config-map.yaml
kubectl create -f ./gitlab/postgres/postgres-deployment.yaml
kubectl create -f ./gitlab/postgres/postgres-service.yaml

# deploy gitlab itself
kubectl create -f ./gitlab/gitlab/gitlab-persistent-volume-claim.yaml
kubectl create -f ./gitlab/gitlab/gitlab-deployment.yaml
kubectl create -f ./gitlab/gitlab/gitlab-service.yaml
kubectl create -f ./gitlab/gitlab/gitlab-ingress.yaml
```

#### Deploying GitLab Runner

```
# deploy Minio
kubectl create -f ./gitlab/minio/gitlab-persistent-volume-claim.yaml
kubectl create -f ./gitlab/minio/minio-deployment.yaml
kubectl create -f ./gitlab/minio/minio-service.yaml

# check that it's running
kubectl get pods --namespace=gitlab

    gitlab   minio-2719559383-y9hl2    1/1    Running   0   1m

# check logs for AccessKey and SecretKey
> $ kubectl logs -f minio-2719559383-y9hl2 --namespace=gitlab
+ exec app server /export

Endpoint:  http://10.2.23.7:9000  http://127.0.0.1:9000
AccessKey: 9HRGG3EK2DD0GP2HBB53
SecretKey: zr+tKa6fS4/3PhYKWekJs65tRh8pbVl07cQlJfkW
Region:    us-east-1

Browser Access:
   http://10.2.23.7:9000  http://127.0.0.1:9000

Command-line Access: https://docs.minio.io/docs/minio-client-quickstart-guide
   $ mc config host add myminio http://10.2.23.7:9000 9HRGG3EK2DD0GP2HBB53 zr+tKa6fS4/3PhYKWekJs65tRh8pbVl07cQlJfkW

Object API (Amazon S3 compatible):
   Go:         https://docs.minio.io/docs/golang-client-quickstart-guide
   Java:       https://docs.minio.io/docs/java-client-quickstart-guide
   Python:     https://docs.minio.io/docs/python-client-quickstart-guide
   JavaScript: https://docs.minio.io/docs/javascript-client-quickstart-guide


# create bucket named `runner`
> $ kubectl exec -it minio-2719559383-y9hl2 --namespace=gitlab -- bash -c 'mkdir /export/runner'

# register runner with token found on http://yourrunning-gitlab/admin/runners
> $ kubectl run -it runner-registrator --image=gitlab/gitlab-runner:v1.5.2 --restart=Never -- register
Waiting for pod default/runner-registrator-1573168835-tmlom to be running, status is Pending, pod ready: false

Hit enter for command prompt

Please enter the gitlab-ci coordinator URL (e.g. https://gitlab.com/ci):
http://gitlab.gitlab/ci
Please enter the gitlab-ci token for this runner:
_TBBy-zRLk7ac1jZfnPu
Please enter the gitlab-ci description for this runner:

Please enter the gitlab-ci tags for this runner (comma separated):
shared,specific
Registering runner... succeeded                     runner=_TBBy-zR
Please enter the executor: virtualbox, docker+machine, docker-ssh+machine, docker, docker-ssh, parallels, shell, ssh:
docker
Please enter the default Docker image (eg. ruby:2.1):
python:3.5.1
Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded!
Session ended, resume using 'kubectl attach runner-registrator-1573168835-tmlom -c runner-registrator -i -t' command when the pod is running

# delete temporary registrator
> $ kubectl delete deployment/runner-registrator


```

Edit `gitlab/gitlab-runner/gitlab-runner-docker-configmap.yaml` and fill in your runner token and minio credentials.

```
# deploy configmap and runnner itself
> $ kubectl create -f ./gitlab-runner/gitlab-runner-docker-configmap.yaml
> $ kubectl create -f ./gitlab-runner/gitlab-runner-docker-deployment.yaml
```

## Grant access to the cluster to your teammates
#### Generate the certificates

Copy the key and certificates from the kops configuration:
```
export KEY=<KEY_FILENAME>.key
export CRT=<CRT_FILENAME>.crt
export MASTER_CRT=<CRT_FILENAME>.crt
export USERNAME=<USERNAME>

aws s3 cp s3://${KOPS_STATE_STORE}/${NAME}/pki/private/ca/${KEY} ca.keY
aws s3 cp s3://${KOPS_STATE_STORE}/${NAME}/pki/issued/ca/${CRT} ca.crt
aws s3 cp s3://${KOPS_STATE_STORE}/${NAME}/pki/issued/master/${MASTER_CRT} ca_master.crt
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
kubectl config set-cluster ${NAME} \
  --server=https://api.${NAME} \
  --certificate-authority=./ca_master.crt \
  --embed-certs=true \
  --kubeconfig=~/path/to/config

kubectl config set-credentials ${NAME} \
  --client-certificate=./${USERNAME}.crt \
  --client-key=./${USERNAME}.key \
  --embed-certs=true \
  --kubeconfig=~/path/to/config

kubectl config set-context ${NAME} \
  --cluster=${NAME} \
  --user=${NAME} \
  --kubeconfig=~/path/to/config     
```
