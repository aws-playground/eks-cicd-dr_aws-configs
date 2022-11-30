# eks-cicd-dr_aws-configs

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

## FIS 

* Create IAM role for FIS
  * Ref : https://docs.aws.amazon.com/fis/latest/userguide/getting-started-iam-service-role.html#create-iam-role

```
$ cd fis-iam
$ aws iam create-role --role-name my-fis-role --assume-role-policy-document file://fis-role-trust-policy.json
$ aws iam attach-role-policy --role-name my-fis-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSFaultInjectionSimulatorNetworkAccess
```


