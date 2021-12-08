# Karpenter Install

## Environment Variables

```
export CLUSTER_NAME=$USER-karpenter-demo
export AWS_PROFILE=cloud-nation-production
export AWS_DEFAULT_REGION=us-east-1
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
```

## Create a cluster using eksctl

```
cat <<EOF > cluster.yaml
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: ${CLUSTER_NAME}
  region: ${AWS_DEFAULT_REGION}
  version: "1.21"
managedNodeGroups:
  - instanceType: t2.medium
    amiFamily: AmazonLinux2
    name: ${CLUSTER_NAME}-ng
    desiredCapacity: 1
    minSize: 1
    maxSize: 5
iam:
  withOIDC: true
EOF

eksctl create cluster -f cluster.yaml
```

## Connect to EKS cluster

```
aws eks --region us-east-1 update-kubeconfig --name william-karpenter-demo
```

## Tag subnets

```
SUBNET_IDS=$(aws cloudformation describe-stacks \
    --stack-name eksctl-${CLUSTER_NAME}-cluster \
    --query 'Stacks[].Outputs[?OutputKey==`SubnetsPrivate`].OutputValue' \
    --output text)

aws ec2 create-tags \
    --resources $(echo $SUBNET_IDS | tr ',' '\n') \
    --tags Key="kubernetes.io/cluster/${CLUSTER_NAME}",Value=
```


## Create the Karpenter IAM Role

```
TEMPOUT=$(mktemp)
curl -fsSL https://karpenter.sh/docs/getting-started/cloudformation.yaml > $TEMPOUT \
&& aws cloudformation deploy \
  --stack-name Karpenter-${CLUSTER_NAME} \
  --template-file ${TEMPOUT} \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides ClusterName=${CLUSTER_NAME}
```

## Grant the instances using the profile to connect to the cluster

```
eksctl create iamidentitymapping \
  --username system:node:{{EC2PrivateDNSName}} \
  --cluster  ${CLUSTER_NAME} \
  --arn arn:aws:iam::${AWS_ACCOUNT_ID}:role/KarpenterNodeRole-${CLUSTER_NAME} \
  --group system:bootstrappers \
  --group system:nodes
```

## Create the Karpenter controller IAM Role

```
eksctl create iamserviceaccount \
  --cluster $CLUSTER_NAME --name karpenter --namespace karpenter \
  --attach-policy-arn arn:aws:iam::$AWS_ACCOUNT_ID:policy/KarpenterControllerPolicy-$CLUSTER_NAME \
  --approve
```

## Create the EC2 Spot Service Linked Role

```
aws iam create-service-linked-role --aws-service-name spot.amazonaws.com

# If the role has already been successfully created, you will see:
# An error occurred (InvalidInput) when calling the CreateServiceLinkedRole operation: Service role name AWSServiceRoleForEC2Spot has been taken in this account, please try a different suffix.
```

## Install Karpenter Helm Chart

```
helm repo add karpenter https://charts.karpenter.sh
helm repo update
helm upgrade --install karpenter karpenter/karpenter --namespace karpenter \
  --create-namespace --set serviceAccount.create=false --version 0.5.1 \
  --set controller.clusterName=${CLUSTER_NAME} \
  --set controller.clusterEndpoint=$(aws eks describe-cluster --name ${CLUSTER_NAME} --query "cluster.endpoint" --output json) \
  --wait
```

## Provisioner Example

```
cat <<EOF | kubectl apply -f -
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: default-provisioner
spec:
  requirements:
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["spot"]
      
  # Taints may prevent pods from scheduling if they are not tolerated
  taints:
    - key: cloud-nation.net/default-provisioner
      effect: NoSchedule
      
  limits:
    resources:
      cpu: 1000
  provider:
    instanceProfile: KarpenterNodeInstanceProfile-${CLUSTER_NAME}
  ttlSecondsAfterEmpty: 30
EOF
```

## Testing Karpenter

### Getting Karpenter Pods

```
k get pods -n karpenter                                                                                                                                                                  ─╯
NAME                                   READY   STATUS    RESTARTS   AGE
karpenter-controller-9698d9bdc-9jkct   1/1     Running   0          93s
karpenter-webhook-597d94ff4b-9trs4     1/1     Running   0          93s
```

### Getting Karpenter Logs

```
k logs karpenter-controller-9698d9bdc-9jkct -n karpenter                                                                                                                                 ─╯
2021-12-06T20:52:40.450Z	INFO	Successfully created the logger.
2021-12-06T20:52:40.450Z	INFO	Logging level set to: info
{"level":"info","ts":1638823960.5593593,"logger":"fallback","caller":"injection/injection.go:61","msg":"Starting informers..."}
2021-12-06T20:52:40.614Z	INFO	controller	starting metrics server	{"commit": "6984094", "path": "/metrics"}
I1206 20:52:40.715255       1 leaderelection.go:243] attempting to acquire leader lease karpenter/karpenter-leader-election...
I1206 20:52:40.755759       1 leaderelection.go:253] successfully acquired lease karpenter/karpenter-leader-election
2021-12-06T20:52:40.756Z	INFO	controller.controller.counter	Starting EventSource	{"commit": "6984094", "reconciler group": "karpenter.sh", "reconciler kind": "Provisioner", "source": "kind source: /, Kind="}
2021-12-06T20:52:40.756Z	INFO	controller.controller.counter	Starting EventSource	{"commit": "6984094", "reconciler group": "karpenter.sh", "reconciler kind": "Provisioner", "source": "kind source: /, Kind="}
2021-12-06T20:52:40.756Z	INFO	controller.controller.counter	Starting Controller	{"commit": "6984094", "reconciler group": "karpenter.sh", "reconciler kind": "Provisioner"}
2021-12-06T20:52:40.756Z	INFO	controller.controller.provisioning	Starting EventSource	{"commit": "6984094", "reconciler group": "karpenter.sh", "reconciler kind": "Provisioner", "source": "kind source: /, Kind="}
2021-12-06T20:52:40.756Z	INFO	controller.controller.provisioning	Starting Controller	{"commit": "6984094", "reconciler group": "karpenter.sh", "reconciler kind": "Provisioner"}
2021-12-06T20:52:40.757Z	INFO	controller.controller.termination	Starting EventSource	{"commit": "6984094", "reconciler group": "", "reconciler kind": "Node", "source": "kind source: /, Kind="}
2021-12-06T20:52:40.757Z	INFO	controller.controller.termination	Starting Controller	{"commit": "6984094", "reconciler group": "", "reconciler kind": "Node"}
2021-12-06T20:52:40.757Z	INFO	controller.controller.node	Starting EventSource	{"commit": "6984094", "reconciler group": "", "reconciler kind": "Node", "source": "kind source: /, Kind="}
2021-12-06T20:52:40.757Z	INFO	controller.controller.node	Starting EventSource	{"commit": "6984094", "reconciler group": "", "reconciler kind": "Node", "source": "kind source: /, Kind="}
2021-12-06T20:52:40.757Z	INFO	controller.controller.node	Starting EventSource	{"commit": "6984094", "reconciler group": "", "reconciler kind": "Node", "source": "kind source: /, Kind="}
2021-12-06T20:52:40.757Z	INFO	controller.controller.node	Starting Controller	{"commit": "6984094", "reconciler group": "", "reconciler kind": "Node"}
2021-12-06T20:52:40.757Z	INFO	controller.controller.metrics	Starting EventSource	{"commit": "6984094", "reconciler group": "karpenter.sh", "reconciler kind": "Provisioner", "source": "kind source: /, Kind="}
2021-12-06T20:52:40.757Z	INFO	controller.controller.metrics	Starting Controller	{"commit": "6984094", "reconciler group": "karpenter.sh", "reconciler kind": "Provisioner"}
2021-12-06T20:52:40.857Z	INFO	controller.controller.counter	Starting workers	{"commit": "6984094", "reconciler group": "karpenter.sh", "reconciler kind": "Provisioner", "worker count": 10}
2021-12-06T20:52:40.857Z	INFO	controller.controller.provisioning	Starting workers	{"commit": "6984094", "reconciler group": "karpenter.sh", "reconciler kind": "Provisioner", "worker count": 10}
2021-12-06T20:52:40.874Z	INFO	controller.controller.termination	Starting workers	{"commit": "6984094", "reconciler group": "", "reconciler kind": "Node", "worker count": 10}
2021-12-06T20:52:40.875Z	INFO	controller.controller.node	Starting workers	{"commit": "6984094", "reconciler group": "", "reconciler kind": "Node", "worker count": 10}
2021-12-06T20:52:40.877Z	INFO	controller.controller.metrics	Starting workers	{"commit": "6984094", "reconciler group": "karpenter.sh", "reconciler kind": "Provisioner", "worker count": 10}
2021-12-06T20:53:33.311Z	INFO	controller.provisioning	Starting provisioner	{"commit": "6984094", "provisioner": "default"}
2021-12-06T20:53:33.311Z	INFO	controller.provisioning	Waiting for unschedulable pods	{"commit": "6984094", "provisioner": "default"}
```

### Automatic Node Provisioning

```
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inflate
spec:
  replicas: 0
  selector:
    matchLabels:
      app: inflate
  template:
    metadata:
      labels:
        app: inflate
    spec:
      terminationGracePeriodSeconds: 0
      containers:
        - name: inflate
          image: public.ecr.aws/eks-distro/kubernetes/pause:3.2
          resources:
            requests:
              cpu: 1
      tolerations:
      - key: "cloud-nation.net/default-provisioner"
        operator: "Exists"
        effect: "NoSchedule"
EOF

kubectl scale deployment inflate --replicas 5
kubectl logs -f -n karpenter $(kubectl get pods -n karpenter -l karpenter=controller -o name)
```

### Automatic Node Scale Down

```
k scale deploy inflate --replicas=3
deployment.apps/inflate scaled
```

```
k logs karpenter-controller-9698d9bdc-9jkct -n karpenter

2021-12-06T21:05:04.257Z	INFO	controller.provisioning	Launched instance: i-01376780e2d25467e, hostname: ip-192-168-127-113.ec2.internal, type: c5.xlarge, zone: us-east-1d, capacityType: spot	{"commit": "6984094", "provisioner": "default"}
2021-12-06T21:05:04.299Z	INFO	controller.provisioning	Bound 3 pod(s) to node ip-192-168-127-113.ec2.internal{"commit": "6984094", "provisioner": "default"}
2021-12-06T21:05:04.299Z	INFO	controller.provisioning	Waiting for unschedulable pods	{"commit": "6984094", "provisioner": "default"}
2021-12-06T21:41:00.039Z	INFO	controller.node	Added TTL to empty node	{"commit": "6984094", "node": "ip-192-168-107-49.ec2.internal"}
2021-12-06T21:41:00.065Z	INFO	controller.node	Added TTL to empty node	{"commit": "6984094", "node": "ip-192-168-107-49.ec2.internal"}
2021-12-06T21:41:30.066Z	INFO	controller.node	Triggering termination after 30s for empty node	{"commit": "6984094", "node": "ip-192-168-107-49.ec2.internal"}
2021-12-06T21:41:30.113Z	INFO	controller.termination	Cordoned node	{"commit": "6984094", "node": "ip-192-168-107-49.ec2.internal"}
2021-12-06T21:41:30.328Z	INFO	controller.termination	Deleted node	{"commit": "6984094", "node": "ip-192-168-107-49.ec2.internal"}
```

## Adding other Karpenter Provisioner

The following Karpenter Provisioner can be used only if the Deployment has an specific taint.

```
cat <<EOF | kubectl apply -f -
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: nginx-provisioner
spec:
  requirements:
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["on-demand"]

  # Taints may prevent pods from scheduling if they are not tolerated
  taints:
    - key: cloud-nation.net/nginx-provisioner
      effect: NoSchedule
      
  provider:
    instanceProfile: KarpenterNodeInstanceProfile-${CLUSTER_NAME}
    
  # If nil, the feature is disabled, nodes will never scale down
  ttlSecondsAfterEmpty: 30
EOF
```

### Provisioners List

```
kubectl get provisioner
NAME                AGE
default-provisioner   2m1s
nginx-provisioner     8m49s
```

### Using the Karpenter nginx-provisioner



### Delete deployment

```
k delete deploy inflate
deployment.apps "inflate" deleted
```

```
k get nodes
NAME                             STATUS   ROLES    AGE   VERSION
ip-192-168-36-138.ec2.internal   Ready    <none>   62m   v1.21.5-eks-bc4871b
```

## Setting Instance Type

### Create a default node provisioner

```
cat <<EOF | kubectl apply -f -
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: default
spec:
  requirements:
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["on-demand"]
  provider:
    instanceProfile: KarpenterNodeInstanceProfile-${CLUSTER_NAME}
  ttlSecondsAfterEmpty: 30
EOF
```

### Deployment with m5.large instance type

```
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inflate
spec:
  replicas: 0
  selector:
    matchLabels:
      app: inflate
  template:
    metadata:
      labels:
        app: inflate
    spec:
      nodeSelector:
        node.kubernetes.io/instance-type: m5.large
      terminationGracePeriodSeconds: 0
      containers:
        - name: inflate
          image: public.ecr.aws/eks-distro/kubernetes/pause:3.2
          resources:
            requests:
              cpu: 1
EOF

kubectl scale deploy inflate --replicas=2
```

### Deployment with t3.xlarge instance type

```
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 0
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      nodeSelector:
        node.kubernetes.io/instance-type: t3.xlarge
      terminationGracePeriodSeconds: 0
      containers:
        - name: nginx
          image: nginx
EOF

kubectl scale deploy nginx --replicas=30
```
