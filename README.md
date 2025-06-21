# aws-eks-secret-csi-demo


### Lets Understand what is Secrets Store CSI Driver

```
The Secrets Store CSI Driver secrets-store.csi.k8s.io allows Kubernetes to mount multiple secrets, keys, and certs stored in enterprise-grade external secrets stores into their pods as a volume. Once the Volume is attached, the data in it is mounted into the container's file system.
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
            "Resource": "arn:aws:secretsmanager:us-east-2:251620460948:secret:secret-store-IXeV0p"
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
                "Federated": "arn:aws:iam::251620460948:oidc-provider/oidc.eks.us-east-2.amazonaws.com/id/D13D2AB696B7BC7F2F9F3DD856F23F2F"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "oidc.eks.us-east-2.amazonaws.com/id/D13D2AB696B7BC7F2F9F3DD856F23F2F:sub": "system:serviceaccount:default:my-sa"
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
aws iam attach-role-policy --role-name role-aws-eks-secret-csi-demo --policy-arn=arn:aws:iam::251620460948:policy/aws-eks-secret-csi-demo
```

### Appending Annotations in the ServiceAccount we have to use.

```
kubectl annotate serviceaccount my-sa eks.amazonaws.com/role-arn=arn:aws:iam::251620460948:role/role-aws-eks-secret-csi-demo
```
