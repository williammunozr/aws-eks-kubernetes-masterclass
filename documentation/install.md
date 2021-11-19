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
        allowedUnsafeSysctls:
        - net.ipv4.tcp_keepalive_time
        - net.ipv4.tcp_keepalive_intvl
        - net.ipv4.tcp_keepalive_probes
```

## Get cluster

```
eksctl get cluster --profile cloud-nation-production
```

## Get nodegroups 

```
eksctl get nodegroups --cluster=eksdemo1 --profile cloud-nation-production
```

## Connect to cluster

```
aws eks --region us-east-1 update-kubeconfig --name eksdemo1 --profile cloud-nation-production
```

## Edit Pod Security Policy

```
kubectl edit psp eks.privileged

Add after:

allowPrivilegeEscalation: true
allowedUnsafeSysctls:
- net.ipv4.tcp_keepalive_time
- net.ipv4.tcp_keepalive_intvl
- net.ipv4.tcp_keepalive_probes
```

## Pod sysctl configuration

```
apiVersion: v1
kind: Pod
metadata:
  name: sysctl-pod
spec:
  securityContext:
    sysctls:
    - name: net.ipv4.tcp_keepalive_time
      value: "120"
    - name: net.ipv4.tcp_keepalive_intvl
      value: "60"
    - name: net.ipv4.tcp_keepalive_probes
      value: "30" 
  containers:
  - name: sdk
    image: mcr.microsoft.com/dotnet/runtime:5.0-alpine3.13
```

## Busybox test

```
apiVersion: v1
kind: Pod
metadata:
  name: sysctl-busybox
spec:
  securityContext:
    sysctls:
    - name: net.ipv4.tcp_keepalive_time
      value: "120"
    - name: net.ipv4.tcp_keepalive_intvl
      value: "60"
    - name: net.ipv4.tcp_keepalive_probes
      value: "30" 
  containers:
  - name: busybox
    image: busybox
    command: ['sh', '-c', 'sleep 1048']
```

## Deploy a pod with sysctl configuration

```
kubectl apply -f pod-sysctl.yaml
kubectl apply -f busybox-sysctl.yaml
```

### Read parameters in the busybox container

```
kubectl exec sysctl-busybox -it -- sh

/ # sysctl net.ipv4.tcp_keepalive_time
net.ipv4.tcp_keepalive_time = 120
/ # sysctl net.ipv4.tcp_keepalive_intvl
net.ipv4.tcp_keepalive_intvl = 60
/ # sysctl net.ipv4.tcp_keepalive_probes
net.ipv4.tcp_keepalive_probes = 30
```

## Cleaning

### Delete the node group

```
eksctl delete nodegroup --cluster=eksdemo1 --name=ng-sysctl-enabled --profile cloud-nation-production
```

## Documentation

- https://eksctl.io/usage/customizing-the-kubelet/
- https://eksctl.io/usage/managing-nodegroups/
