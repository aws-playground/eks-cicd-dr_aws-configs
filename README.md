# eks-cicd-dr_aws-configs

## ECR

* Create private repos

```
$ aws ecr create-repository --region ap-northeast-2 --repository-name eks-cicd-dr_service-peccy-web
$ aws ecr create-repository --region ap-northeast-1 --repository-name eks-cicd-dr_service-peccy-web
$ aws ecr create-repository --region ap-northeast-2 --repository-name eks-cicd-dr_service-peccy-app
$ aws ecr create-repository --region ap-northeast-1 --repository-name eks-cicd-dr_service-peccy-app
```


## CodeBuild

* Create role for CodeBuild
  * Ref : https://blog.chrismitchellonline.com/posts/codebuild-iam-role

```
$ aws iam create-role --role-name CodeBuildRole --assume-role-policy-document file://codebuild-iam-role-trust.json
$ aws iam attach-role-policy --role-name CodeBuildRole --policy-arn arn:aws:iam::aws:policy/AWSCodeBuildAdminAccess
$ aws iam attach-role-policy --role-name CodeBuildRole --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryFullAccess
$ aws iam attach-role-policy --role-name CodeBuildRole --policy-arn arn:aws:iam::aws:policy/CloudWatchLogsFullAccess
```

* Create CodeBuild project
  * Ref : https://docs.aws.amazon.com/codebuild/latest/userguide/create-project-cli.html
  * TODO


## Route 53

* Create hosted zone
  * TODO


## Certificate Manager

* Create letsencrypt IAM User, Access Key

```
# Create letsencrypt IAM user
$ aws iam create-user --user-name letsencrypt

# Create & attach policy for letsencrypt IAM user
$ cd aws_certificate_maanger
$ aws iam create-policy --policy-name letsencrypt-policy --policy-document file://iam-policy.json
$ aws iam attach-user-policy --user-name letsencrypt --policy-arn arn:aws:iam::727618787612:policy/letsencrypt-policy

# Create access key for letsencryt user
$ aws iam create-access-key --user-name letsencrypt
{
    "AccessKey": {
        "UserName": "letsencrypt",
        "AccessKeyId": "{AWS_Access_ID"},
        "Status": "Active",
        "SecretAccessKey": "{AWS_Secret_Key}",
        "CreateDate": "2022-12-05T03:38:49+00:00"
    }
}
```

* Create crdential file for certbot

```
$ cat <<EOF > ~/.aws/credentials
[letsencrypt]
aws_access_key_id={AWS_Access_ID}
aws_secret_access_key={AWS_Secret_Key}
EOF
```

* Create certificate by using certbot

```
$ mkdir certbot && cd certbot
$ certbot certonly --dns-route53 -n -d "aws-playground.dev" -d "*.aws-playground.dev" --email supsup5642@gmail.com --agree-tos --config-dir . --work-dir . --logs-dir .
Saving debug log to /Users/ssup2/certbot/letsencrypt.log
Account registered.
Requesting a certificate for aws-playground.dev and *.aws-playground.dev

Successfully received certificate.
Certificate is saved at: /Users/ssupp/certbot/live/aws-playground.dev/fullchain.pem
Key is saved at:         /Users/ssupp/certbot/live/aws-playground.dev/privkey.pem
This certificate expires on 2023-03-05.
These files will be updated when the certificate renews.

NEXT STEPS:
- The certificate will need to be renewed before it expires. Certbot can automatically renew the certificate in the background, but you may need to take steps to enable that functionality. See https://certbot.org/renewal-setup for instructions.

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
If you like Certbot, please consider supporting our work by:
 * Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
 * Donating to EFF:                    https://eff.org/donate-le
```

* Import created certificate

```
$ cd live/aws-playground.dev
$ aws acm import-certificate --region ap-northeast-2 --certificate fileb://cert.pem --private-key fileb://privkey.pem --certificate-chain fileb://chain.pem
{
    "CertificateArn": "arn:aws:acm:ap-northeast-2:727618787612:certificate/08554d50-4579-4ea9-bc0d-42b7591df7a2"
}
$ aws acm import-certificate --region ap-northeast-1 --certificate fileb://cert.pem --private-key fileb://privkey.pem --certificate-chain fileb://chain.pem
{
    "CertificateArn": "arn:aws:acm:ap-northeast-1:727618787612:certificate/043ffa30-3d8a-42c4-ae51-566594fe6787"
}
```


## EFS

* TODO


## EKS

* Create primary-seoul and secondary-tyoko EKS clusters

```
$ cd eks_eksctl
$ eksctl create cluster -f primary-seoul.yaml
$ eksctl create cluster -f secondary-tokyo.yaml
```

* Install AWS load balancer controller
  * Ref : https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html

```
# Create IAM policy
$ cd eks_loadbalancer-controller
$ aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json

# Create EKS cluster's oidc-provider
$ eksctl utils associate-iam-oidc-provider --cluster eks-cicd-dr-primary-seoul --approve --region ap-northeast-2
$ eksctl utils associate-iam-oidc-provider --cluster eks-cicd-dr-secondary-tyoko --approve --region ap-northeast-1

# Create aws-load-balancer-controller service account in EKS clusters and assign role AmazonEKSLoadBalancerControllerRole to aws-load-balancer-controller service account
$ eksctl create iamserviceaccount --cluster=eks-cicd-dr-primary-seoul --region ap-northeast-2 --namespace=kube-system --name=aws-load-balancer-controller --role-name "AmazonEKSLoadBalancerControllerRoleSeoul" --attach-policy-arn=arn:aws:iam::727618787612:policy/AWSLoadBalancerControllerIAMPolicy --approve
$ eksctl create iamserviceaccount --cluster=eks-cicd-dr-secondary-tyoko --region ap-northeast-1 --namespace=kube-system --name=aws-load-balancer-controller --role-name "AmazonEKSLoadBalancerControllerRoleTyoko" --attach-policy-arn=arn:aws:iam::727618787612:policy/AWSLoadBalancerControllerIAMPolicy --approve

# Install AWS Load Balancer Controller   
$ helm repo add eks https://aws.github.io/eks-charts
$ helm repo update
$ eksctl utils write-kubeconfig --region ap-northeast-2 --cluster eks-cicd-dr-primary-seoul
$ helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=eks-cicd-dr-primary-seoul --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller --set nodeSelector."eks.\amazonaws\.com/nodegroup"=manage
$ eksctl utils write-kubeconfig --region ap-northeast-1 --cluster eks-cicd-dr-secondary-tyoko
$ helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=eks-cicd-dr-secondary-tyoko --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller --set nodeSelector."eks\.amazonaws\.com/nodegroup"=manage
```

* Install EFS CSI driver
  * Ref : https://docs.aws.amazon.com/eks/latest/userguide/efs-csi.html

```
# Create IAM policy for EFS CSI driver
$ eks-efs-csi 
$ aws iam create-policy --policy-name AmazonEKS_EFS_CSI_Driver_Policy --policy-document file://iam-policy.json 

# Create efs-csi-controller-sa service account in EKS clusters and assign role AmazonEKS_EFS_CSI_Driver_Policy to efs-csi-controller-sa service account
$ eksctl create iamserviceaccount --cluster eks-cicd-dr-primary-seoul --namespace kube-system --name efs-csi-controller-sa --attach-policy-arn arn:aws:iam::727618787612:policy/AmazonEKS_EFS_CSI_Driver_Policy --approve --region ap-northeast-2
$ eksctl create iamserviceaccount --cluster eks-cicd-dr-secondary-tyoko --namespace kube-system --name efs-csi-controller-sa --attach-policy-arn arn:aws:iam::727618787612:policy/AmazonEKS_EFS_CSI_Driver_Policy --approve --region ap-northeast-1

# Install EFS CSI driver
$ helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver/
$ helm repo update
$ eksctl utils write-kubeconfig --region ap-northeast-2 --cluster eks-cicd-dr-primary-seoul
$ helm upgrade -i aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver --namespace kube-system --set controller.serviceAccount.create=false --set controller.serviceAccount.name=efs-csi-controller-sa --set controller.nodeSelector."eks\.amazonaws\.com/nodegroup"=manage --set node.nodeSelector."eks\.amazonaws\.com/nodegroup"=app-server
$ eksctl utils write-kubeconfig --region ap-northeast-1 --cluster eks-cicd-dr-secondary-tyoko
$ helm upgrade -i aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver --namespace kube-system --set controller.serviceAccount.create=false --set controller.serviceAccount.name=efs-csi-controller-sa --set controller.nodeSelector."eks\.amazonaws\.com/nodegroup"=manage --set node.nodeSelector."eks\.amazonaws\.com/nodegroup"=app-server

# Create EFS PV
$ eksctl utils write-kubeconfig --region ap-northeast-2 --cluster eks-cicd-dr-primary-seoul
$ kubectl apply -f storageclass.yaml -f pv-seoul.yaml -f pvc.yaml
$ eksctl utils write-kubeconfig --region ap-northeast-1 --cluster eks-cicd-dr-secondary-tyoko
$ kubectl apply -f storageclass.yaml -f pv-tyoko.yaml -f pvc.yaml
```

* Install ExternalDNS
  * Ref : https://aws.amazon.com/ko/premiumsupport/knowledge-center/eks-set-up-externaldns/

```
# Create IAM policy
$ cd eks_externaldns
$ aws iam create-policy --policy-name AmazonEKSExternalDNSControllerIAMPolicy --policy-document file://iam-policy.json
{
    "Policy": {
        "PolicyName": "AmazonEKSExternalDNSControllerIAMPolicy",
        "PolicyId": "ANPA2S2LVTEOAJP3HTM5T",
        "Arn": "arn:aws:iam::727618787612:policy/AmazonEKSExternalDNSControllerIAMPolicy",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "CreateDate": "2022-12-13T23:35:31+00:00",
        "UpdateDate": "2022-12-13T23:35:31+00:00"
    }
}

# Create external-dns service account in EKS clusters and assign role AmazonEKSLoadBalancerControllerRole to aws-load-balancer-controller service account
$ eksctl create iamserviceaccount --cluster=eks-cicd-dr-primary-seoul --region ap-northeast-2 --namespace=kube-system --name=external-dns --role-name "AmazonEKSExternalDNSControllerRoleSeoul" --attach-policy-arn=arn:aws:iam::727618787612:policy/AmazonEKSExternalDNSControllerIAMPolicy --approve
$ eksctl create iamserviceaccount --cluster=eks-cicd-dr-secondary-tyoko --region ap-northeast-1 --namespace=kube-system --name=external-dns --role-name "AmazonEKSExternalDNSControllerRoleTyoko" --attach-policy-arn=arn:aws:iam::727618787612:policy/AmazonEKSExternalDNSControllerIAMPolicy --approve

# Install ExternalDNS
$ eksctl utils write-kubeconfig --region ap-northeast-2 --cluster eks-cicd-dr-primary-seoul
$ kubectl apply -f externaldns-with-rbac-seoul.yaml
$ eksctl utils write-kubeconfig --region ap-northeast-1 --cluster eks-cicd-dr-secondary-tyoko
$ kubectl apply -f externaldns-with-rbac-tyoko.yaml
```

* Install CloudWatch Agent
  * Ref : https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Container-Insights-prerequisites.html
  * Ref : https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Container-Insights-setup-metrics.html

```
# Attach CloudWatchAgentServerPolicy to role of EKS cluster nodes
$ aws iam list-roles --query 'Roles[*].RoleName' | grep eksctl-eks-cicd-dr-primary-seoul-NodeInstanceRole
    "eksctl-eks-cicd-dr-primary-seoul-NodeInstanceRole-13AHXTXCBH24F",
    "eksctl-eks-cicd-dr-primary-seoul-NodeInstanceRole-2D7QZL7PDTAI",
    "eksctl-eks-cicd-dr-primary-seoul-NodeInstanceRole-FCQW0WEKNDPV",
$ aws iam attach-role-policy --role-name eksctl-eks-cicd-dr-primary-seoul-NodeInstanceRole-13AHXTXCBH24F --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
$ aws iam attach-role-policy --role-name eksctl-eks-cicd-dr-primary-seoul-NodeInstanceRole-2D7QZL7PDTAI --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
$ aws iam attach-role-policy --role-name eksctl-eks-cicd-dr-primary-seoul-NodeInstanceRole-FCQW0WEKNDPV --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
$ aws iam list-roles --query 'Roles[*].RoleName' | grep eksctl-eks-cicd-dr-primary-tyoko-NodeInstanceRole
    "eksctl-eks-cicd-dr-secondary-tyok-NodeInstanceRole-1QF0Y4R35AFZB",
    "eksctl-eks-cicd-dr-secondary-tyok-NodeInstanceRole-1SZ1M4E9OF6QJ",
    "eksctl-eks-cicd-dr-secondary-tyok-NodeInstanceRole-YUEZL16E7LBX",
$ aws iam attach-role-policy --role-name eksctl-eks-cicd-dr-secondary-tyok-NodeInstanceRole-1QF0Y4R35AFZB --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
$ aws iam attach-role-policy --role-name eksctl-eks-cicd-dr-secondary-tyok-NodeInstanceRole-1SZ1M4E9OF6QJ --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
$ aws iam attach-role-policy --role-name eksctl-eks-cicd-dr-secondary-tyok-NodeInstanceRole-YUEZL16E7LBX --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy

# Install to seoul cluster 
$ eksctl utils write-kubeconfig --region ap-northeast-2 --cluster eks-cicd-dr-primary-seoul
$ kubectl create namespace amazon-cloudwatch
$ kubectl apply -f https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/cwagent/cwagent-serviceaccount.yaml
$ kubectl apply -f eks_cloudwatch-agent/cwagent-configmap-seoul.yaml
$ kubectl apply -f https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/cwagent/cwagent-daemonset.yaml 

# Install to tyoko cluster
$ eksctl utils write-kubeconfig --region ap-northeast-1 --cluster eks-cicd-dr-secondary-tyoko
$ kubectl create namespace amazon-cloudwatch
$ kubectl apply -f https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/cwagent/cwagent-serviceaccount.yaml
$ kubectl apply -f eks_cloudwatch-agent/cwagent-configmap-tyoko.yaml
$ kubectl apply -f https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/cwagent/cwagent-daemonset.yaml 
```

* Install FluentBit
  * Ref : https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Container-Insights-setup-logs-FluentBit.html

```
# Install to seoul cluster 
$ eksctl utils write-kubeconfig --region ap-northeast-2 --cluster eks-cicd-dr-primary-seoul
$ ClusterName=eks-cicd-dr-secondary-seoul
RegionName=ap-northeast-2
FluentBitHttpPort='2020'
FluentBitReadFromHead='Off'
[[ ${FluentBitReadFromHead} = 'On' ]] && FluentBitReadFromTail='Off'|| FluentBitReadFromTail='On'
[[ -z ${FluentBitHttpPort} ]] && FluentBitHttpServer='Off' || FluentBitHttpServer='On'
kubectl create configmap fluent-bit-cluster-info \
--from-literal=cluster.name=${ClusterName} \
--from-literal=http.server=${FluentBitHttpServer} \
--from-literal=http.port=${FluentBitHttpPort} \
--from-literal=read.head=${FluentBitReadFromHead} \
--from-literal=read.tail=${FluentBitReadFromTail} \
--from-literal=logs.region=${RegionName} -n amazon-cloudwatch
$ kubectl apply -f https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/fluent-bit/fluent-bit.yaml

# Install to tyoko cluster
$ eksctl utils write-kubeconfig --region ap-northeast-1 --cluster eks-cicd-dr-secondary-tyoko
$ ClusterName=eks-cicd-dr-secondary-tyoko
RegionName=ap-northeast-1
FluentBitHttpPort='2020'
FluentBitReadFromHead='Off'
[[ ${FluentBitReadFromHead} = 'On' ]] && FluentBitReadFromTail='Off'|| FluentBitReadFromTail='On'
[[ -z ${FluentBitHttpPort} ]] && FluentBitHttpServer='Off' || FluentBitHttpServer='On'
kubectl create configmap fluent-bit-cluster-info \
--from-literal=cluster.name=${ClusterName} \
--from-literal=http.server=${FluentBitHttpServer} \
--from-literal=http.port=${FluentBitHttpPort} \
--from-literal=read.head=${FluentBitReadFromHead} \
--from-literal=read.tail=${FluentBitReadFromTail} \
--from-literal=logs.region=${RegionName} -n amazon-cloudwatch
$ kubectl apply -f https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/fluent-bit/fluent-bit.yaml
```

* Install ArgoCD

```
# Install ArgoCD
$ cd eks_argo-cd
$ eksctl utils write-kubeconfig --region ap-northeast-2 --cluster eks-cicd-dr-primary-seoul
$ helm install argo-cd --namespace argo-cd --create-namespace -f values-seoul.yaml . 
$ eksctl utils write-kubeconfig --region ap-northeast-1 --cluster eks-cicd-dr-secondary-tyoko
$ helm install argo-cd --namespace argo-cd --create-namespace -f values-tyoko.yaml .
```

* Install Botkube
  * Get slack token through below reference
  * Ref : https://docs.botkube.io/installation/slack/

```
# Install Botkube
$ cd eks_botkube
$ eksctl utils write-kubeconfig --region ap-northeast-2 --cluster eks-cicd-dr-primary-seoul
$ helm install botkube --namespace botkube --create-namespace --set communications.default-group.socketSlack.channels.default.name=aws_eks-cicd-dr_eks-seoul --set communications.default-group.socketSlack.appToken={app-token} --set communications.default-group.socketSlack.botToken={bot-token} --set settings.clusterName=eks-seoul . 
$ eksctl utils write-kubeconfig --region ap-northeast-1 --cluster eks-cicd-dr-secondary-tyoko
$ helm install botkube --namespace botkube --create-namespace --set communications.default-group.socketSlack.channels.default.name=aws_eks-cicd-dr_eks-tyoko --set communications.default-group.socketSlack.appToken={app-token} --set communications.default-group.socketSlack.botToken={bot-token} --set settings.clusterName=eks-tyoko . 
```


## FIS 

* Create IAM role for FIS
  * Ref : https://docs.aws.amazon.com/fis/latest/userguide/getting-started-iam-service-role.html#create-iam-role

```
$ cd aws_fis
$ aws iam create-role --role-name my-fis-role --assume-role-policy-document file://fis-role-trust-policy.json
$ aws iam attach-role-policy --role-name my-fis-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSFaultInjectionSimulatorNetworkAccess
```

* Create Template and Run
  * TODO

