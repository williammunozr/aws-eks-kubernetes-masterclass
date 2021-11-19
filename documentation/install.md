# Amazon EKS Configuration

## Create Cluster

```
eksctl create cluster --name=eksdemo1 \
                      --region=us-east-1 \
                      --zones=us-east-1a,us-east-1b \
                      --without-nodegroup \
                      --profile cloud-nation-production

# Get List of Clusters
eksctl get cluster --profile cloud-nation-production
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

## Create a node group

```
# Create Public Node Group   
eksctl create nodegroup --cluster=eksdemo1 \
                       --region=us-east-1 \
                       --name=eksdemo1-ng-public1 \
                       --node-type=t3.medium \
                       --nodes=2 \
                       --nodes-min=2 \
                       --nodes-max=4 \
                       --node-volume-size=20 \
                       --ssh-access \
                       --ssh-public-key=cloud-station \
                       --managed \
                       --asg-access \
                       --external-dns-access \
                       --full-ecr-access \
                       --appmesh-access \
                       --alb-ingress-access \
                       --profile cloud-nation-production
```

## Create a node group customizing kubelet configuration

```
eksctl create nodegroup --config-file=ng-sysctl-enabled.yaml --profile cloud-nation-production
```

### ng-sysctl-enabled.yaml file

```
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: eksdemo1
  region: us-east-1

nodeGroups:
  - name: ng-sysctl-enabled
    instanceType: t3.medium
    desiredCapacity: 1
    kubeletExtraConfig:
        allowedUnsafeSysctl:
        - net.ipv4.tcp_keepalive_time
        - net.ipv4.tcp_keepalive_intvl
        - net.ipv4.tcp_keepalive_probes
```
