# Amazon EKS Configuration

## Create Cluster

```
eksctl create cluster --name=eksdemo1 \
                      --region=us-east-1 \
                      --zones=us-east-1a,us-east-1b \
                      --without-nodegroup \
                      --profile cloud-nation-production

# Get List of Clusters
eksctl get cluster
```

## Create & Associate IAM OIDC Provider for EKS Cluster

```
# Replace with region & cluster name
eksctl utils associate-iam-oidc-provider \
    --region us-east-1 \
    --cluster eksdemo1 \
    --approve \
    --profile cloud-nation-production
```


