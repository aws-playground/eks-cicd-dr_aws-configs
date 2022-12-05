# eks-cicd-dr_aws-configs

## Certificate Manager

* Create letsencrypt IAM User, Access Key

```
# Create letsencrypt IAM user
$ aws iam create-user --user-name letsencrypt
{
    "User": {
        "Path": "/",
        "UserName": "letsencrypt",
        "UserId": "AIDA2S2LVTEOIYTEXDCLM",
        "Arn": "arn:aws:iam::727618787612:user/letsencrypt",
        "CreateDate": "2022-12-05T03:37:20+00:00"
    }
}

# Create & attach policy for letsencrypt IAM user
$ cat <<EOF > policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "route53:ListHostedZones",
        "route53:GetChange",
        "route53:ChangeResourceRecordSets"
      ],
      "Resource": [
        "*"
      ]
    }
  ]
}
EOF 
$ aws iam create-policy --policy-name letsencrypt-policy --policy-document file://policy.json
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
$ mkdir certbot cd certbot
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
$ aws acm import-certificate --region ap-northeast-3 --certificate fileb://cert.pem --private-key fileb://privkey.pem --certificate-chain fileb://chain.pem
{
    "CertificateArn": "arn:aws:acm:ap-northeast-3:727618787612:certificate/dcd9c7f6-60a2-4871-a15d-cd757831b498"
}
```

## EKS

* Create primary-seoul and secondary-osaka EKS clusters

```
$ cd eks-eksctl
$ eksctl create cluster -f primary-seoul.yaml
$ eksctl create cluster -f secondary-osaka.yaml
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

# Create aws-load-balancer-controller service account in EKS clusters and assign role AmazonEKSLoadBalancerControllerRole to aws-load-balancer-controller service account
$ eksctl create iamserviceaccount --cluster=cicd-dr-primary-seoul --region ap-northeast-2 --namespace=kube-system --name=aws-load-balancer-controller --role-name "AmazonEKSLoadBalancerControllerRoleSeoul" --attach-policy-arn=arn:aws:iam::727618787612:policy/AWSLoadBalancerControllerIAMPolicy --approve
$ eksctl create iamserviceaccount --cluster=cicd-dr-secondary-osaka --region ap-northeast-3 --namespace=kube-system --name=aws-load-balancer-controller --role-name "AmazonEKSLoadBalancerControllerRoleOsaka" --attach-policy-arn=arn:aws:iam::727618787612:policy/AWSLoadBalancerControllerIAMPolicy --approve

# Install AWS Load Balancer Controller   
$ helm repo add eks https://aws.github.io/eks-charts
$ helm repo update
$ eksctl utils write-kubeconfig --region ap-northeast-2 --cluster cicd-dr-primary-seoul
$ helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=cicd-dr-primary-seoul --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller --set nodeSelector."alpha\.eksctl\.io/nodegroup-name"=manage
$ eksctl utils write-kubeconfig --region ap-northeast-3 --cluster cicd-dr-secondary-osaka
$ helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=cicd-dr-secondary-osaka --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller --set nodeSelector."alpha\.eksctl\.io/nodegroup-name"=manage
```

* Install EFS CSI driver
  * Ref : https://docs.aws.amazon.com/eks/latest/userguide/efs-csi.html

```
# Create IAM policy for EFS CSI driver
$ curl -o iam-policy-example.json https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/docs/iam-policy-example.json
$ aws iam create-policy --policy-name AmazonEKS_EFS_CSI_Driver_Policy --policy-document file://iam-policy-example.json
{
    "Policy": {
        "PolicyName": "AmazonEKS_EFS_CSI_Driver_Policy",
        "PolicyId": "ANPA2S2LVTEONW4N3TXGH",
        "Arn": "arn:aws:iam::727618787612:policy/AmazonEKS_EFS_CSI_Driver_Policy",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "CreateDate": "2022-11-18T05:56:58+00:00",
        "UpdateDate": "2022-11-18T05:56:58+00:00"
    }
}

# Create EKS cluster's oidc-provider
$ eksctl utils associate-iam-oidc-provider --cluster cicd-dr-primary-seoul --approve --region ap-northeast-2
$ eksctl utils associate-iam-oidc-provider --cluster cicd-dr-secondary-osaka --approve --region ap-northeast-3

# Create efs-csi-controller-sa service account in EKS clusters and assign role AmazonEKS_EFS_CSI_Driver_Policy to efs-csi-controller-sa service account
$ eksctl create iamserviceaccount --cluster cicd-dr-primary-seoul --namespace kube-system --name efs-csi-controller-sa --attach-policy-arn arn:aws:iam::727618787612:policy/AmazonEKS_EFS_CSI_Driver_Policy --approve --region ap-northeast-2
$ eksctl create iamserviceaccount --cluster cicd-dr-secondary-osaka --namespace kube-system --name efs-csi-controller-sa --attach-policy-arn arn:aws:iam::727618787612:policy/AmazonEKS_EFS_CSI_Driver_Policy --approve --region ap-northeast-3

# Install EFS CSI driver
$ helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver/
$ helm repo update
$ eksctl utils write-kubeconfig --region ap-northeast-2 --cluster cicd-dr-primary-seoul
$ helm upgrade -i aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver --namespace kube-system --set controller.serviceAccount.create=false --set controller.serviceAccount.name=efs-csi-controller-sa --set controller.nodeSelector."alpha\.eksctl\.io/nodegroup-name"=manage --set node.nodeSelector."alpha\.eksctl\.io/nodegroup-name"=app-server
$ eksctl utils write-kubeconfig --region ap-northeast-3 --cluster cicd-dr-secondary-osaka
$ helm upgrade -i aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver --namespace kube-system --set controller.serviceAccount.create=false --set controller.serviceAccount.name=efs-csi-controller-sa --set controller.nodeSelector."alpha\.eksctl\.io/nodegroup-name"=manage --set node.nodeSelector."alpha\.eksctl\.io/nodegroup-name"=app-server

# Create EFS storage class
$ cd eks-efs-csi 
$ eksctl utils write-kubeconfig --region ap-northeast-2 --cluster cicd-dr-primary-seoul
$ kubectl patch storageclass gp2 -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
$ kubectl apply -f primary-seoul-storageclass.yaml
$ eksctl utils write-kubeconfig --region ap-northeast-3 --cluster cicd-dr-secondary-osaka
$ kubectl patch storageclass gp2 -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

* Install ArgoCD

```
# Install ArgoCD
$ cd argo-cd
$ eksctl utils write-kubeconfig --region ap-northeast-2 --cluster cicd-dr-primary-seoul
$ 
$ eksctl utils write-kubeconfig --region ap-northeast-3 --cluster cicd-dr-secondary-osaka
$ 
```

* Install Botkube
  * Get slack token through below reference
  * Ref : https://docs.botkube.io/installation/slack/

```
# Install Botkube
$ cd botkube
$ eksctl utils write-kubeconfig --region ap-northeast-2 --cluster cicd-dr-primary-seoul
$ helm install botkube --namespace botkube --create-namespace --set communications.default-group.socketSlack.channels.default.name=aws_eks-cicd-dr_eks-seoul --set communications.default-group.socketSlack.appToken={app-token} --set communications.default-group.socketSlack.botToken={bot-token} --set settings.clusterName=eks-seoul . 
$ eksctl utils write-kubeconfig --region ap-northeast-3 --cluster cicd-dr-secondary-osaka
$ helm install botkube --namespace botkube --create-namespace --set communications.default-group.socketSlack.channels.default.name=aws_eks-cicd-dr_eks-osaka --set communications.default-group.socketSlack.appToken={app-token} --set communications.default-group.socketSlack.botToken={bot-token} --set settings.clusterName=eks-osaka . 
```

## FIS 

* Create IAM role for FIS
  * Ref : https://docs.aws.amazon.com/fis/latest/userguide/getting-started-iam-service-role.html#create-iam-role

```
$ cd fis-iam
$ aws iam create-role --role-name my-fis-role --assume-role-policy-document file://fis-role-trust-policy.json
$ aws iam attach-role-policy --role-name my-fis-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSFaultInjectionSimulatorNetworkAccess
```


