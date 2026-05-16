# Cluster Autoscaler Setup on Amazon EKS

This guide explains how to install and configure Cluster Autoscaler on an Amazon EKS cluster using the AWS cloud provider.

---

# 1. Deploy Cluster Autoscaler

Apply the official Cluster Autoscaler manifest for your Kubernetes version.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/cluster-autoscaler-1.29.0/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml
```

---

# 2. Verify the Pod

Check whether the Cluster Autoscaler pod is running in the `kube-system` namespace.

```bash
kubectl -n kube-system get pods -l app=cluster-autoscaler
```

Expected output:

```bash
NAME                                   READY   STATUS    RESTARTS   AGE
cluster-autoscaler-6889f6cf54-7pcsh   1/1     Running   0          2m
```

---

# 3. Edit Deployment and Add Cluster Name

Edit the Cluster Autoscaler deployment.

```bash
kubectl -n kube-system edit deployment.apps/cluster-autoscaler
```

Inside the deployment manifest, locate the container arguments section and update the cluster name.

```yaml
containers:
  - name: cluster-autoscaler
    command:
      - ./cluster-autoscaler
      - --v=4
      - --stderrthreshold=info
      - --cloud-provider=aws
      - --skip-nodes-with-local-storage=false
      - --expander=least-waste
      - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/naresh
    image: registry.k8s.io/autoscaling/cluster-autoscaler:v1.26.2
    imagePullPolicy: Always
```

Replace `naresh` with your EKS cluster name if different.

Save and exit the editor.

---

# 4. Configure IAM Permissions

Cluster Autoscaler requires IAM permissions to scale worker nodes.

Attach either:

* `AmazonEKSClusterAutoscalerPolicy` (AWS managed policy)

OR

* A custom IAM policy using the JSON below.

Example IAM policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": [
        "autoscaling:DescribeAutoScalingGroups",
        "autoscaling:DescribeAutoScalingInstances",
        "autoscaling:DescribeLaunchConfigurations",
        "autoscaling:DescribeTags",
        "autoscaling:SetDesiredCapacity",
        "autoscaling:TerminateInstanceInAutoScalingGroup",
        "ec2:DescribeLaunchTemplateVersions"
      ],
      "Effect": "Allow",
      "Resource": "*"
    }
  ]
}
```

Attach this policy to your EKS Node Group IAM Role.

---

# 5. Update Node Group Scaling Configuration

Configure the minimum, maximum, and desired node counts.

```bash
aws eks update-nodegroup-config \
  --cluster-name naresh \
  --nodegroup-name ng-af5ac006 \
  --scaling-config minSize=2,maxSize=6,desiredSize=3
```

Explanation:

* `minSize=2` → Minimum number of nodes always running
* `maxSize=6` → Maximum nodes autoscaler can create
* `desiredSize=3` → Initial desired node count

---

# 6. Check Cluster Autoscaler Logs

View the autoscaler logs to confirm it is working correctly.

```bash
kubectl -n kube-system logs -f deployment/cluster-autoscaler
```

Example scale-up logs:

```bash
I0828 17:36:38.403432       1 scale_up.go:422] Pod default/nginx-deployment-12345 is unschedulable ...
I0828 17:36:38.403451       1 scale_up.go:423] Scale-up triggered ...
```

Explanation:

* Pod could not be scheduled because resources were insufficient
* Cluster Autoscaler triggered a scale-up event
* A new worker node will be added automatically

Example scale-down logs:

```bash
I0516 13:16:01.132868       1 static_autoscaler.go:541] Calculating unneeded nodes
I0516 13:16:01.132902       1 eligibility.go:144] Node is not suitable for removal - cpu utilization too big
I0516 13:16:01.132995       1 static_autoscaler.go:598] Starting scale down
I0516 13:16:01.133013       1 legacy.go:298] No candidates for scale down
```

Explanation:

* Autoscaler checked nodes for removal
* Nodes were heavily utilized
* No nodes qualified for scale down

---

# 7. Validate Cluster Autoscaler

Create a workload with many replicas to trigger autoscaling.

```bash
kubectl create deployment nginx --image=nginx --replicas=50
```

Watch for new nodes being added.

```bash
kubectl get nodes -w
```

Scale down the deployment.

```bash
kubectl scale deployment nginx --replicas=1
```

Observe whether unused nodes are removed automatically.

---

# 8. Important Notes

* Only one Cluster Autoscaler pod should run per cluster
* Cluster Autoscaler uses leader election internally
* Ensure Node Group IAM Role has autoscaling permissions
* Incorrect IAM permissions can prevent scaling operations
* Pods may remain in `Pending` state if scaling fails
* Scale down happens only when utilization is low
* Scale up occurs when pods become unschedulable
