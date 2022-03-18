# Amazon EKS EBS CSI Driver

## Create AMI Policy

```
aws iam create-policy \
    --policy-name Amazon_EBS_CSI_Driver \
    --policy-document file://Amazon_EBS_CSI_Driver.json \
    --description "Policy for EC2 Instances to access Elastic Block Store" \
    --profile cloud-nation-dev

Important: Save the policy ARN
arn:aws:iam::568700646906:policy/Amazon_EBS_CSI_Driver

# Get the policy ARN
aws iam list-policies --profile cloud-nation-dev
```

## Get IAM role Worker Nodes ARN

```
# Get Worker node IAM Role ARN
kubectl -n kube-system describe configmap aws-auth

# from output check rolearn
rolearn: arn:aws:iam::568700646906:role/eksctl-eksdemo1-nodegroup-eksdemo-NodeInstanceRole-YJ3YAIILOYYI
```

## Attach Policy

```
aws iam attach-role-policy \
--role-name eksctl-eksdemo1-nodegroup-eksdemo-NodeInstanceRole-YJ3YAIILOYYI \
--policy-arn "arn:aws:iam::568700646906:policy/Amazon_EBS_CSI_Driver" \
--profile cloud-nation-dev
```

## Deploy Amazon EBS CSI Driver

```
# Verify kubectl version, it should be 1.14 or later
kubectl version --client --short
Client Version: v1.21.5

# Deploy Amazon EBS CSI Driver
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=master"

# Verify ebs-csi pods running
kubectl get pods -n kube-system

NAME                                 READY   STATUS    RESTARTS   AGE
aws-node-8n2sx                       1/1     Running   0          39m
aws-node-hdgf8                       1/1     Running   0          39m
coredns-66cb55d4f4-p9fmz             1/1     Running   0          49m
coredns-66cb55d4f4-pqldz             1/1     Running   0          49m
ebs-csi-controller-766ff9798-477kh   6/6     Running   0          38s
ebs-csi-controller-766ff9798-rxsvs   6/6     Running   0          38s
ebs-csi-node-6mfhr                   3/3     Running   0          37s
ebs-csi-node-z2hc6                   3/3     Running   0          37s
kube-proxy-ntstt                     1/1     Running   0          39m
kube-proxy-pm8ch                     1/1     Running   0          39m
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
eksctl get cluster --profile cloud-nation-dev
```

## Get nodegroups 

```
eksctl get nodegroups --cluster=eksdemo1 --profile cloud-nation-dev
```

## Connect to cluster

```
aws eks --region us-east-1 update-kubeconfig --name eksdemo1 --profile cloud-nation-dev
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
eksctl delete nodegroup --cluster=eksdemo1 --name=ng-sysctl-enabled --profile cloud-nation-dev
```

### Delete the cluster

```
eksctl delete cluster eksdemo1 --profile cloud-nation-dev
```

## Documentation

- https://eksctl.io/usage/customizing-the-kubelet/
- https://eksctl.io/usage/managing-nodegroups/
- https://itnext.io/of-kubernetes-unsafe-sysctls-performance-optimization-on-eks-d36cc0e3e894
