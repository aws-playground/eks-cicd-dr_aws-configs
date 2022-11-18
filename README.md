# eks-cicd-dr_aws-configs

## EKS

* Create primary-seoul and secondary-osaka EKS clusters

```
$ cd eks-eksctl
$ eksctl create cluster -f primary-seoul.yaml
$ eksctl create cluster -f secondary-osaka.yaml
```

## FIS 

* Create IAM role for FIS
  * Reference : https://docs.aws.amazon.com/fis/latest/userguide/getting-started-iam-service-role.html#create-iam-role

```
$ cd fis-iam
$ aws iam create-role --role-name my-fis-role --assume-role-policy-document file://fis-role-trust-policy.json
$ aws iam attach-role-policy --role-name my-fis-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSFaultInjectionSimulatorNetworkAccess
```


