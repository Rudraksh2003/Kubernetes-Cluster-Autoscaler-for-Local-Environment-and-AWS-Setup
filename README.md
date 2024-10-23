### **Kubernetes Cluster Autoscaler for Local Environment and AWS Setup**
---
This repository demonstrates how to set up **Kubernetes Cluster Autoscaler** for both **local environments** (e.g., Minikube or custom Kubernetes setups) and **AWS EKS** clusters. 

In both scenarios, the **Cluster Autoscaler** automatically adjusts the number of nodes in your cluster based on the current workload, ensuring efficient resource usage by scaling up when demand increases and scaling down when resources are no longer needed.

---

### **Features**

- **Dynamic Scaling**: Automatically scales your Kubernetes cluster up or down based on the resource demands of your workloads.
- **Scale Down Support**: Reduces the number of nodes when they become underutilized, optimizing costs and resource usage.
- **Local Environment and AWS Support**: Works both for local Kubernetes environments (e.g., Minikube, on-prem clusters) and for AWS EKS (Elastic Kubernetes Service).

---

### **Architecture Overview**

The **Kubernetes Cluster Autoscaler** works by monitoring the state of the cluster and making scaling decisions based on the following key conditions:

1. **Pending Pods**: If there are pods that cannot be scheduled due to insufficient resources (like CPU or memory), the autoscaler adds nodes to the cluster to meet the demand.
   
2. **Idle Nodes**: If there are nodes that are not running any user workloads (only system pods or no pods), the autoscaler can safely remove them to reduce costs and resource usage.

#### **Key Components:**

- **Cluster Autoscaler**: The main component responsible for scaling the cluster up or down.
- **Kubernetes Control Plane**: Interacts with the autoscaler and the node group to ensure the cluster's state reflects the current workload requirements.
- **Cloud Provider Integration**: In the case of AWS, the autoscaler directly interacts with EKS to request new EC2 instances or remove existing ones.

#### **Local Environment Setup:**

In a local environment like Minikube or custom on-prem Kubernetes clusters, the autoscaler does not rely on cloud-specific APIs. Instead, it adjusts the number of nodes based on available resources and the configured node limits.

#### **AWS Setup:**

For AWS EKS clusters, the **Cluster Autoscaler** communicates with the **EKS API** to manage EC2 instances. When additional resources are required, the autoscaler increases the size of the EKS node group; when resources are no longer needed, it scales the node group down.

---

### **Cluster Autoscaler Workflow**

1. **Scale Up**:
   - When the Kubernetes scheduler cannot place a pod due to a lack of available resources, the autoscaler will scale the cluster up by adding more nodes. For AWS, this means launching additional EC2 instances within the node group.
   
2. **Scale Down**:
   - When nodes become idle (i.e., no active workloads other than system pods), the autoscaler will remove those nodes, either deprovisioning local resources or terminating EC2 instances in AWS.

3. **Node Group Limits**:
   - The number of nodes in the cluster is limited by the minimum and maximum values specified in the autoscaler configuration (e.g., `--nodes=1:10` for both local and AWS environments).

---

### **Local Setup:**

#### **Prerequisites:**

- A running Kubernetes cluster (e.g., Minikube, on-prem clusters).
- `kubectl` CLI installed and configured.

#### **Deployment Steps:**

1. **Apply the Cluster Autoscaler deployment for local setup**:
   
   ```bash
   kubectl apply -f cluster-autoscaler-deployment.yaml
   ```

2. **Verify the Autoscaler**:
   
   ```bash
   kubectl get pods -n kube-system
   ```

   Ensure that the `cluster-autoscaler` pod is running.

3. **Monitor Autoscaling**:
   - Check logs to see scaling decisions:
     ```bash
     kubectl logs -n kube-system deployment/cluster-autoscaler
     ```

4. **View Nodes**:
   - Check the number of nodes:
     ```bash
     kubectl get nodes
     ```

---

### **AWS EKS Setup:**

#### **Prerequisites:**

- An AWS EKS cluster running with at least one node group.
- `kubectl` CLI installed and configured.
- Proper IAM permissions for the Cluster Autoscaler to modify the node group.

#### **Deployment Steps for AWS**:

1. **Modify the deployment YAML for AWS**:
   
   Use the following configuration to set up autoscaling for AWS:

   ```yaml
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
         containers:
         - image: k8s.gcr.io/autoscaling/cluster-autoscaler:v1.22.1
           name: cluster-autoscaler
           command:
             - ./cluster-autoscaler
             - --cloud-provider=aws
             - --nodes=1:10:<your-node-group-name>
             - --scale-down-enabled=true
             - --skip-nodes-with-local-storage=false
             - --skip-nodes-with-system-pods=false
           resources:
             requests:
               cpu: 100m
               memory: 300Mi
           env:
             - name: AWS_REGION
               value: <your-aws-region>
           volumeMounts:
             - name: ssl-certs
               mountPath: /etc/ssl/certs/ca-certificates.crt
               readOnly: true
         volumes:
           - name: ssl-certs
             hostPath:
               path: /etc/ssl/certs/ca-certificates.crt
   ```

   Replace `<your-node-group-name>` and `<your-aws-region>` with your AWS-specific information.

2. **Apply the Deployment**:

   ```bash
   kubectl apply -f aws-cluster-autoscaler-deployment.yaml
   ```

3. **Verify the Deployment**:

   Ensure that the `cluster-autoscaler` is running in the `kube-system` namespace:

   ```bash
   kubectl get pods -n kube-system
   ```

4. **Monitor Autoscaling**:

   Check the logs to see the autoscaler activity:

   ```bash
   kubectl logs -n kube-system deployment/cluster-autoscaler
   ```

   Also, monitor your EKS node group in the **AWS Console** to verify scaling events.

---

### **Configuration File:**

For local environments:

```yaml
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
      containers:
      - image: k8s.gcr.io/autoscaling/cluster-autoscaler:v1.22.1
        name: cluster-autoscaler
        command:
          - ./cluster-autoscaler
          - --cloud-provider=cluster-autoscaler
          - --nodes=1:10
          - --scale-down-enabled=true
          - --skip-nodes-with-local-storage=false
          - --skip-nodes-with-system-pods=false
        resources:
          requests:
            cpu: 100m
            memory: 300Mi
        volumeMounts:
          - name: ssl-certs
            mountPath: /etc/ssl/certs/ca-certificates.crt
            readOnly: true
      volumes:
        - name: ssl-certs
          hostPath:
            path: /etc/ssl/certs/ca-certificates.crt
```

For AWS EKS:

```yaml
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
      containers:
      - image: k8s.gcr.io/autoscaling/cluster-autoscaler:v1.22.1
        name: cluster-autoscaler
        command:
          - ./cluster-autoscaler
          - --cloud-provider=aws
          - --nodes=1:10:<your-node-group-name>
          - --scale-down-enabled=true
          - --skip-nodes-with-local-storage=false
          - --skip-nodes-with-system-pods=false
        resources:
          requests:
            cpu: 100m
            memory: 300Mi
        env:
          - name: AWS_REGION
            value: <your-aws-region>
        volumeMounts:
          - name: ssl-certs
            mountPath: /etc/ssl/certs/ca-certificates.crt
            readOnly: true
      volumes:
        - name: ssl-certs
          hostPath:
            path: /etc/ssl/certs/ca-certificates.crt
```

---

### **Contributing**

If you'd like to contribute, feel free to open an issue or submit a pull request. All contributions are welcome!

---

### **License**

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

---
