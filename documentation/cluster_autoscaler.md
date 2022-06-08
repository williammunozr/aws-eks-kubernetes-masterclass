# Amazon EKS Configuration

## Cluster Autoscaler

Cluster Autoscaler is a tool that automatically adjusts the size of a Kubernetes cluster when one of the following conditions is true:

- There are pods that failed to run in the cluster due to insufficient resources.
- There are nodes in the cluster that have been `underutilized` for an extended period of time and their pods can be placed on other existing nodes.
- Cluster Autoscaler modifies the worker node groups so they `scale out` when we need more resources and `scale in` when we have underutilized resources.

## Create Cluster

```
eksctl create cluster --name=calico \
                      --region=us-east-1 \
                      --zones=us-east-1a,us-east-1b,us-east-1c \
                      --without-nodegroup \
		      --version=1.22 \
                      --profile cloud-nation-dev

# Get List of Clusters
eksctl get cluster --profile cloud-nation-dev
```

## Create & Associate IAM OIDC Provider for EKS Cluster

```
# Replace with region & cluster name
eksctl utils associate-iam-oidc-provider \
    --region us-east-1 \
    --cluster calico \
    --approve \
    --profile cloud-nation-dev
```

## Create a node group

Validate if the node group has the option `--asg-access` is enabled.

```
# Create Public Node Group   
eksctl create nodegroup --cluster=calico \
                       --region=us-east-1 \
                       --name=calico-ng-public1 \
                       --node-type=t3.medium \
                       --nodes=1 \
                       --nodes-min=1 \
                       --nodes-max=5 \
                       --node-volume-size=20 \
                       --ssh-access \
                       --ssh-public-key=cloud-station \
                       --managed \
                       --asg-access \
                       --external-dns-access \
                       --full-ecr-access \
                       --appmesh-access \
                       --alb-ingress-access \
                       --profile cloud-nation-dev
```

## Create a private node group

```
eksctl create nodegroup \
	--alb-ingress-access \
	--asg-access \
	--cluster=calico \
	--external-dns-access \
	--full-ecr-access \
	--managed \
	--name=lab-ng \
	--node-type=t3.medium \
	--nodes=1 \
	--nodes-max=5 \
	--nodes-min=0 \
	--node-volume-size=20 \
	--region=us-east-1 \
	--node-private-networking
```

## Deploy Cluster Autoscaler

```
# Deploy the Cluster Autoscaler
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml

# Add the cluster-autoscaler.kubernetes.io/safe-to-evict annotation to the deployment
kubectl -n kube-system annotate deployment.apps/cluster-autoscaler cluster-autoscaler.kubernetes.io/safe-to-evict="false"
```

### Edit Cluster Autoscaler Deployment

- Add cluster name
- Add two parameters

```
kubectl -n kube-system edit deployment.apps/cluster-autoscaler
```

Search the following line and change `<YOUR CLUSTER NAME>` with the name of the cluster, in this example is `calico`.

```
- --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/<YOUR CLUSTER NAME>

Should be:

- --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/calico

And add the following lines:

    - --balance-similar-node-groups
    - --skip-nodes-with-system-pods=false
```

### Validate cluster autoscaler pod

```
kubectl get pods -n kube-system
NAME                                  READY   STATUS    RESTARTS   AGE
aws-node-8dgtl                        1/1     Running   0          16m
cluster-autoscaler-5d5f8d7676-spjdm   1/1     Running   0          85s
coredns-66cb55d4f4-f8fb2              1/1     Running   0          29m
coredns-66cb55d4f4-n7ktp              1/1     Running   0          29m
kube-proxy-9lgz8                      1/1     Running   0          16m
```

### Get cluster version

```
kubectl version --short
Client Version: v1.21.4
Server Version: v1.21.2-eks-06eac09
```

### Set cluster autoscaler image to current EKS cluster version

- Open https://github.com/kubernetes/autoscaler/releases
- I found: `k8s.gcr.io/autoscaling/cluster-autoscaler:v1.21.1`

```
# Update Cluster Autoscaler Image Version
kubectl -n kube-system set image deployment.apps/cluster-autoscaler cluster-autoscaler=us.gcr.io/k8s-artifacts-prod/autoscaling/cluster-autoscaler:v1.21.1
```

### Verify image version

```
kubectl -n kube-system get deployment.apps/cluster-autoscaler -o yaml
```

Sample partial output:

```yaml
    spec:
      containers:
      - command:
        - ./cluster-autoscaler
        - --v=4
        - --stderrthreshold=info
        - --cloud-provider=aws
        - --skip-nodes-with-local-storage=false
        - --expander=least-waste
        - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/calico
        - --balance-similar-node-groups
        - --skip-nodes-with-system-pods=false
        image: us.gcr.io/k8s-artifacts-prod/autoscaling/cluster-autoscaler:v1.21.1
```

### View cluster autoscaler logs

```
kubectl -n kube-system logs -f deployment.apps/cluster-autoscaler
```

## Cleaning

### Delete the node group

```
eksctl delete nodegroup --cluster=calico --name=ng-sysctl-enabled --profile cloud-nation-dev
```

### Delete the cluster

```
eksctl delete cluster calico --profile cloud-nation-dev
```

## Documentation

- https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/FAQ.md
