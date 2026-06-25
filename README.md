# Cluster Autoscaler Setup for Amazon EKS

## Prerequisites

- AWS CLI configured with appropriate permissions

- Install kubectl
  Refer https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/

- Install eksctl
  ```
  curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
  sudo mv /tmp/eksctl /usr/local/bin
  ```

- Install helm
  ```
  curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
  chmod 700 get_helm.sh
  ./get_helm.sh
  ```

- Create an EKS Cluster
  ```
  eksctl create cluster --name my-cluster --region ap-south-1 --node-type t3.medium --version 1.35
  ```

---

## Step 1: Create an IAM Policy for Cluster Autoscaler

Create a file called `cluster-autoscaler-policy.json`:

```json
{
    "Version":"2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "autoscaling:SetDesiredCapacity",
                "autoscaling:TerminateInstanceInAutoScalingGroup"
            ],
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "aws:ResourceTag/k8s.io/cluster-autoscaler/enabled": "true",
                    "aws:ResourceTag/k8s.io/cluster-autoscaler/my-cluster": "owned"
                }
            }
        },
        {
            "Effect": "Allow",
            "Action": [
                "autoscaling:DescribeAutoScalingGroups",
                "autoscaling:DescribeAutoScalingInstances",
                "autoscaling:DescribeLaunchConfigurations",
                "autoscaling:DescribeScalingActivities",
                "autoscaling:DescribeTags",
                "ec2:DescribeImages",
                "ec2:DescribeInstanceTypes",
                "ec2:DescribeLaunchTemplateVersions",
                "ec2:GetInstanceTypesFromInstanceRequirements",
                "eks:DescribeNodegroup"
            ],
            "Resource": "*"
        }
    ]
}
```

Create the policy:

```bash
aws iam create-policy \
  --policy-name ClusterAutoscalerPolicy \
  --policy-document file://cluster-autoscaler-policy.json
```

Note the ARN from the output (e.g., `arn:aws:iam::<ACCOUNT_ID>:policy/ClusterAutoscalerPolicy`).

---

## Step 2: Create an IAM Role with IRSA (IAM Roles for Service Accounts)

### Option A: Using eksctl

```bash
eksctl create iamserviceaccount \
  --cluster=<CLUSTER_NAME> \
  --namespace=kube-system \
  --name=cluster-autoscaler \
  --attach-policy-arn=arn:aws:iam::<ACCOUNT_ID>:policy/ClusterAutoscalerPolicy \
  --approve \
  --override-existing-serviceaccounts
```

### Option B: Manual Setup

1. Get your OIDC provider URL:

```bash
aws eks describe-cluster \
  --name <CLUSTER_NAME> \
  --query "cluster.identity.oidc.issuer" \
  --output text
```

2. Create a trust policy (`trust-policy.json`):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/oidc.eks.<REGION>.amazonaws.com/id/<OIDC_ID>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.<REGION>.amazonaws.com/id/<OIDC_ID>:sub": "system:serviceaccount:kube-system:cluster-autoscaler",
          "oidc.eks.<REGION>.amazonaws.com/id/<OIDC_ID>:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
```

3. Create the IAM role:

```bash
aws iam create-role \
  --role-name ClusterAutoscalerRole \
  --assume-role-policy-document file://trust-policy.json

aws iam attach-role-policy \
  --role-name ClusterAutoscalerRole \
  --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/ClusterAutoscalerPolicy
```

---

## Step 3: Tag Your Auto Scaling Groups

Your node group ASGs must have these tags for autodiscovery:

| Key | Value |
|-----|-------|
| `k8s.io/cluster-autoscaler/<CLUSTER_NAME>` | `owned` |
| `k8s.io/cluster-autoscaler/enabled` | `true` |

```bash
aws autoscaling create-or-update-tags \
  --tags \
    ResourceId=<ASG_NAME>,ResourceType=auto-scaling-group,Key=k8s.io/cluster-autoscaler/<CLUSTER_NAME>,Value=owned,PropagateAtLaunch=true \
    ResourceId=<ASG_NAME>,ResourceType=auto-scaling-group,Key=k8s.io/cluster-autoscaler/enabled,Value=true,PropagateAtLaunch=true
```

> If you created your node group with `eksctl`, these tags are added automatically.

---

## Step 4: Deploy the Cluster Autoscaler

Create a file called `cluster-autoscaler-deployment.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/ClusterAutoscalerRole
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-autoscaler
rules:
  - apiGroups: [""]
    resources: ["events", "endpoints"]
    verbs: ["create", "patch"]
  - apiGroups: [""]
    resources: ["pods/eviction"]
    verbs: ["create"]
  - apiGroups: [""]
    resources: ["pods/status"]
    verbs: ["update"]
  - apiGroups: [""]
    resources: ["endpoints"]
    resourceNames: ["cluster-autoscaler"]
    verbs: ["get", "update"]
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["watch", "list", "get", "update"]
  - apiGroups: [""]
    resources: ["namespaces", "pods", "services", "replicationcontrollers", "persistentvolumeclaims", "persistentvolumes"]
    verbs: ["watch", "list", "get"]
  - apiGroups: ["batch", "extensions"]
    resources: ["jobs"]
    verbs: ["watch", "list", "get"]
  - apiGroups: ["policy"]
    resources: ["poddisruptionbudgets"]
    verbs: ["watch", "list"]
  - apiGroups: ["apps"]
    resources: ["statefulsets", "replicasets", "daemonsets"]
    verbs: ["watch", "list", "get"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses", "csinodes", "csidrivers", "csistoragecapacities"]
    verbs: ["watch", "list", "get"]
  - apiGroups: ["coordination.k8s.io"]
    resources: ["leases"]
    verbs: ["create"]
  - apiGroups: ["coordination.k8s.io"]
    resources: ["leases"]
    resourceNames: ["cluster-autoscaler"]
    verbs: ["get", "update"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-autoscaler
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-autoscaler
subjects:
  - kind: ServiceAccount
    name: cluster-autoscaler
    namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: cluster-autoscaler
  namespace: kube-system
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["create", "list", "watch"]
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["cluster-autoscaler-status", "cluster-autoscaler-priority-expander"]
    verbs: ["delete", "get", "update"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cluster-autoscaler
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: cluster-autoscaler
subjects:
  - kind: ServiceAccount
    name: cluster-autoscaler
    namespace: kube-system
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    app: cluster-autoscaler
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
    spec:
      priorityClassName: system-cluster-critical
      serviceAccountName: cluster-autoscaler
      containers:
        - name: cluster-autoscaler
          image: registry.k8s.io/autoscaling/cluster-autoscaler:v1.31.0
          command:
            - ./cluster-autoscaler
            - --v=4
            - --stderrthreshold=info
            - --cloud-provider=aws
            - --skip-nodes-with-local-storage=false
            - --expander=least-waste
            - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/<CLUSTER_NAME>
            - --balance-similar-node-groups
            - --skip-nodes-with-system-pods=false
          resources:
            limits:
              cpu: 100m
              memory: 600Mi
            requests:
              cpu: 100m
              memory: 600Mi
          volumeMounts:
            - name: ssl-certs
              mountPath: /etc/ssl/certs/ca-certificates.crt
              readOnly: true
      volumes:
        - name: ssl-certs
          hostPath:
            path: /etc/ssl/certs/ca-bundle.crt
```

> **Important:** Replace `<CLUSTER_NAME>` and `<ACCOUNT_ID>` with your actual values. Match the Cluster Autoscaler image version to your EKS Kubernetes version (see the [releases page](https://github.com/kubernetes/autoscaler/releases)).

Apply the manifest:

```bash
kubectl apply -f cluster-autoscaler-deployment.yaml
```

---

## Step 5: Add the `cluster-autoscaler.kubernetes.io/safe-to-evict` Annotation

Prevent the autoscaler pod from being evicted by itself:

```bash
kubectl -n kube-system annotate deployment.apps/cluster-autoscaler \
  cluster-autoscaler.kubernetes.io/safe-to-evict="false"
```

---

## Step 6: Verify the Deployment

```bash
# Check the pod is running
kubectl -n kube-system get pods -l app=cluster-autoscaler

# Check the logs
kubectl -n kube-system logs -l app=cluster-autoscaler -f
```

You should see log lines indicating that the autoscaler discovered your ASGs and is monitoring for unschedulable pods.

---

## Step 7: Test Scale-Up

Deploy a sample workload to trigger scaling:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: scale-test
spec:
  replicas: 50
  selector:
    matchLabels:
      app: scale-test
  template:
    metadata:
      labels:
        app: scale-test
    spec:
      containers:
        - name: pause
          image: registry.k8s.io/pause:3.9
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
```

```bash
kubectl apply -f scale-test.yaml

# Watch nodes being added
kubectl get nodes -w

# Clean up after testing
kubectl delete deployment scale-test
```

---

## Common Configuration Flags

| Flag | Default | Description |
|------|---------|-------------|
| `--scale-down-delay-after-add` | `10m` | Wait time after scale-up before considering scale-down |
| `--scale-down-delay-after-delete` | `0s` | Wait time after node deletion before further scale-down |
| `--scale-down-unneeded-time` | `10m` | How long a node must be unneeded before removal |
| `--scale-down-utilization-threshold` | `0.5` | Node utilization below this triggers scale-down consideration |
| `--max-graceful-termination-sec` | `600` | Max seconds to wait for pod termination during scale-down |
| `--expander` | `random` | Strategy for selecting node group (`random`, `least-waste`, `most-pods`, `priority`) |
| `--max-node-provision-time` | `15m` | Max time to wait for a node to become ready |

---

## Troubleshooting

### Autoscaler is not scaling up

```bash
# Check for pending pods
kubectl get pods --field-selector=status.phase=Pending -A

# Check autoscaler status configmap
kubectl -n kube-system get configmap cluster-autoscaler-status -o yaml

# Check autoscaler logs for errors
kubectl -n kube-system logs deployment/cluster-autoscaler | grep -i error
```

### Autoscaler is not scaling down

```bash
# Check for pods with PDBs blocking eviction
kubectl get pdb -A

# Check for pods with local storage
kubectl get pods -A -o json | jq '.items[] | select(.spec.volumes[]?.emptyDir != null) | .metadata.name'

# Check for pods with restrictive anti-affinity or the safe-to-evict annotation
kubectl get pods -A -o json | jq '.items[] | select(.metadata.annotations["cluster-autoscaler.kubernetes.io/safe-to-evict"] == "false") | .metadata.name'
```

### Common issues

| Issue | Cause | Fix |
|-------|-------|-----|
| `Failed to regenerate ASG cache` | Missing IAM permissions | Verify the IAM policy is attached correctly |
| `No ASG found` | Missing ASG tags | Ensure both autodiscovery tags are present on the ASG |
| Nodes scale up but pods stay pending | Instance type has insufficient resources | Check resource requests vs. instance type capacity |
| Scale-down blocked | PDB, local storage, or safe-to-evict annotation | Check pod annotations and PDB configuration |

---

## Cleanup

```bash
kubectl delete -f cluster-autoscaler-deployment.yaml

# If created with eksctl
eksctl delete iamserviceaccount \
  --cluster=<CLUSTER_NAME> \
  --namespace=kube-system \
  --name=cluster-autoscaler

# Delete the IAM policy
aws iam delete-policy \
  --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/ClusterAutoscalerPolicy
```
