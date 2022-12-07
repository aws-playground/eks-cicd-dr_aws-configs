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
$ cd codebuild
$ aws iam create-role --role-name CodeBuildRole --assume-role-policy-document file://codebuild-iam-role-trust.json
$ aws iam put-role-policy --role-name CodeBuildRole --policy-name CodeBuildPolicy --policy-document file://codebuild-iam-policy.json 
```

* Create CodeBuild project
  * Ref : https://docs.aws.amazon.com/codebuild/latest/userguide/create-project-cli.html
  * TODO


## Certificate Manager

* Create letsencrypt IAM User, Access Key

```
# Create letsencrypt IAM user
$ aws iam create-user --user-name letsencrypt

# Create & attach policy for letsencrypt IAM user
$ cd certificate_maanger
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
$ cd eks-eksctl
$ eksctl create cluster -f primary-seoul.yaml
$ eksctl create cluster -f secondary-tokyo.yaml
```

* Install AWS Load Balancer Controller
  * Ref : https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html

```
# Create IAM policy
$ cd eks-loadbalancer-controller
$ aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json
{
    "Policy": {
        "PolicyName": "AWSLoadBalancerControllerIAMPolicy",
        "PolicyId": "ANPA2S2LVTEOBQJHOCHII",
        "Arn": "arn:aws:iam::727618787612:policy/AWSLoadBalancerControllerIAMPolicy",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "CreateDate": "2022-11-30T03:23:14+00:00",
        "UpdateDate": "2022-11-30T03:23:14+00:00"
    }
}

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
$ helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=eks-cicd-dr-primary-seoul --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller --set nodeSelector."alpha\.eksctl\.io/nodegroup-name"=manage
$ eksctl utils write-kubeconfig --region ap-northeast-1 --cluster eks-cicd-dr-secondary-tyoko
$ helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=eks-cicd-dr-secondary-tyoko --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller --set nodeSelector."alpha\.eksctl\.io/nodegroup-name"=manage
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
$ helm upgrade -i aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver --namespace kube-system --set controller.serviceAccount.create=false --set controller.serviceAccount.name=efs-csi-controller-sa --set controller.nodeSelector."alpha\.eksctl\.io/nodegroup-name"=manage --set node.nodeSelector."alpha\.eksctl\.io/nodegroup-name"=app-server
$ eksctl utils write-kubeconfig --region ap-northeast-1 --cluster eks-cicd-dr-secondary-tyoko
$ helm upgrade -i aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver --namespace kube-system --set controller.serviceAccount.create=false --set controller.serviceAccount.name=efs-csi-controller-sa --set controller.nodeSelector."alpha\.eksctl\.io/nodegroup-name"=manage --set node.nodeSelector."alpha\.eksctl\.io/nodegroup-name"=app-server

# Create EFS PV
$ eksctl utils write-kubeconfig --region ap-northeast-2 --cluster eks-cicd-dr-primary-seoul
$ kubectl apply -f storageclass.yaml -f pv-seoul.yaml -f pvc.yaml
$ eksctl utils write-kubeconfig --region ap-northeast-1 --cluster eks-cicd-dr-secondary-tyoko
$ kubectl apply -f storageclass.yaml -f pv-tyoko.yaml -f pvc.yaml
```

* Install ArgoCD

```
# Install ArgoCD
$ cd argo-cd
$ eksctl utils write-kubeconfig --region ap-northeast-2 --cluster eks-cicd-dr-primary-seoul
$ helm install argo-cd --namespace argo-cd --create-namespace -f values-seoul.yaml . 
$ eksctl utils write-kubeconfig --region ap-northeast-1 --cluster eks-cicd-dr-secondary-tyoko
$ helm install argo-cd --namespace argo-cd --create-namespace -f values-tyoko.yaml .

# Set Route53 Alias
$ TODO
```

* Install Botkube
  * Get slack token through below reference
  * Ref : https://docs.botkube.io/installation/slack/

```
# Install Botkube
$ cd botkube
$ eksctl utils write-kubeconfig --region ap-northeast-2 --cluster eks-cicd-dr-primary-seoul
$ helm install botkube --namespace botkube --create-namespace --set communications.default-group.socketSlack.channels.default.name=aws_eks-cicd-dr_eks-seoul --set communications.default-group.socketSlack.appToken={app-token} --set communications.default-group.socketSlack.botToken={bot-token} --set settings.clusterName=eks-seoul . 
$ eksctl utils write-kubeconfig --region ap-northeast-1 --cluster eks-cicd-dr-secondary-tyoko
$ helm install botkube --namespace botkube --create-namespace --set communications.default-group.socketSlack.channels.default.name=aws_eks-cicd-dr_eks-tyoko --set communications.default-group.socketSlack.appToken={app-token} --set communications.default-group.socketSlack.botToken={bot-token} --set settings.clusterName=eks-tyoko . 
```


## FIS 

* Create IAM role for FIS
  * Ref : https://docs.aws.amazon.com/fis/latest/userguide/getting-started-iam-service-role.html#create-iam-role

```
$ cd fis-iam
$ aws iam create-role --role-name my-fis-role --assume-role-policy-document file://fis-role-trust-policy.json
$ aws iam attach-role-policy --role-name my-fis-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSFaultInjectionSimulatorNetworkAccess
```


