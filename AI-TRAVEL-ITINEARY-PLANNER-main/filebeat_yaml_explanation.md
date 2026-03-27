# Filebeat Kubernetes Manifest - Line by Line Explanation

This is a comprehensive Kubernetes manifest for deploying Filebeat as a DaemonSet to collect container logs and ship them to Logstash. Let me explain each section in detail.

## **ConfigMap Section**

```yaml
apiVersion: v1
kind: ConfigMap
```
- Creates a ConfigMap to store Filebeat configuration
- ConfigMaps separate configuration from container images

```yaml
metadata:
  name: filebeat-config
  namespace: logging
  labels:
    k8s-app: filebeat
```
- **name: filebeat-config** - Names the ConfigMap
- **namespace: logging** - Deploys in the logging namespace
- **labels: k8s-app: filebeat** - Labels for identification

```yaml
data:
  filebeat.yml: |-
```
- **data** section contains the configuration file
- **filebeat.yml: |-** - Multi-line string containing Filebeat configuration

```yaml
    filebeat.inputs:
    - type: container
      paths:
        - /var/log/containers/*.log
```
- **filebeat.inputs** - Defines input sources
- **type: container** - Specifically for container logs
- **paths** - Monitors all log files in Kubernetes container logs directory

```yaml
      processors:
        - add_kubernetes_metadata:
            host: ${NODE_NAME}
            matchers:
            - logs_path:
                logs_path: "/var/log/containers/"
```
- **processors** - Data processing pipeline
- **add_kubernetes_metadata** - Enriches logs with Kubernetes metadata (pod name, namespace, labels, etc.)
- **host: ${NODE_NAME}** - Uses node name from environment variable
- **logs_path matcher** - Maps file paths to Kubernetes pod metadata

```yaml
    processors:
      - add_cloud_metadata:
      - add_host_metadata:
```
- Global processors (applies to all inputs)
- **add_cloud_metadata** - Adds cloud provider metadata (AWS, GCP, Azure, etc.)
- **add_host_metadata** - Adds host-level information

```yaml
    output.logstash:
      hosts: ["logstash.logging.svc.cluster.local:5044"]
```
- **output.logstash** - Configures Logstash as the output destination
- **hosts** - Points to Logstash service using Kubernetes DNS naming convention

## **DaemonSet Section**

```yaml
apiVersion: apps/v1
kind: DaemonSet
```
- **DaemonSet** ensures one Filebeat pod runs on each node in the cluster
- Perfect for log collection as logs are node-local

```yaml
metadata:
  name: filebeat
  namespace: logging
  labels:
    k8s-app: filebeat
```
- Names and identifies the DaemonSet

```yaml
spec:
  selector:
    matchLabels:
      k8s-app: filebeat
```
- Selector to manage pods with matching labels

```yaml
  template:
    metadata:
      labels:
        k8s-app: filebeat
```
- Pod template with required labels

```yaml
    spec:
      serviceAccountName: filebeat
```
- Uses custom service account with RBAC permissions

```yaml
      terminationGracePeriodSeconds: 30
```
- Allows 30 seconds for graceful shutdown before force killing

```yaml
      hostNetwork: true
```
- Uses host's network namespace
- Allows Filebeat to access node-level network information

```yaml
      dnsPolicy: ClusterFirstWithHostNet
```
- DNS policy that works with hostNetwork: uses cluster DNS first, then falls back to host's DNS

```yaml
      containers:
      - name: filebeat
        image: docker.elastic.co/beats/filebeat:7.17.28
```
- Container definition using Filebeat version 7.17.28

```yaml
        args: [
          "-c", "/etc/filebeat.yml",
          "-e",
        ]
```
- Command-line arguments:
  - **-c** - Specifies configuration file path
  - **-e** - Logs to stderr (enables container logging)

```yaml
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
```
- **NODE_NAME** - Injects the node name where the pod runs
- Used in Kubernetes metadata enrichment

```yaml
        securityContext:
          runAsUser: 0
```
- **runAsUser: 0** - Runs as root user
- Required to read container logs from host directories
- Comment suggests OpenShift may need privileged: true

```yaml
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 100Mi
```
- **limits** - Maximum resources:
  - Memory capped at 200 MiB
- **requests** - Guaranteed resources:
  - 100 millicores CPU (0.1 core)
  - 100 MiB memory

```yaml
        volumeMounts:
        - name: config
          mountPath: /etc/filebeat.yml
          readOnly: true
          subPath: filebeat.yml
```
- **config volume** - Mounts ConfigMap as configuration file
- **readOnly: true** - Prevents modification
- **subPath** - Mounts single file instead of directory

```yaml
        - name: data
          mountPath: /usr/share/filebeat/data
```
- **data volume** - Stores registry file (tracks which logs have been shipped)
- Prevents re-shipping logs on pod restart

```yaml
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
```
- Mounts Docker container data directory
- Source of container logs

```yaml
        - name: varlog
          mountPath: /var/log
          readOnly: true
```
- Mounts host's /var/log directory
- Contains Kubernetes container logs (symlinked from /var/lib/docker)

```yaml
      volumes:
      - name: config
        configMap:
          defaultMode: 0640
          name: filebeat-config
```
- **config volume** - References the ConfigMap created earlier
- **defaultMode: 0640** - Sets file permissions (owner read/write, group read)

```yaml
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```
- **hostPath volume** - Mounts host directory directly
- Path to Docker container logs

```yaml
      - name: varlog
        hostPath:
          path: /var/log
```
- Mounts host's /var/log directory

```yaml
      - name: data
        hostPath:
          path: /var/lib/filebeat-data
          type: DirectoryOrCreate
```
- **data volume** - Persistent storage for Filebeat registry
- **DirectoryOrCreate** - Creates directory if it doesn't exist
- Prevents data loss on pod restarts

## **RBAC Configuration - ClusterRoleBinding**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: filebeat
subjects:
- kind: ServiceAccount
  name: filebeat
  namespace: logging
roleRef:
  kind: ClusterRole
  name: filebeat
  apiGroup: rbac.authorization.k8s.io
```
- **ClusterRoleBinding** - Binds ClusterRole to ServiceAccount at cluster level
- Grants permissions across all namespaces
- References filebeat ServiceAccount and ClusterRole

## **RBAC Configuration - RoleBindings**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: filebeat
  namespace: logging
subjects:
  - kind: ServiceAccount
    name: filebeat
    namespace: logging
roleRef:
  kind: Role
  name: filebeat
  apiGroup: rbac.authorization.k8s.io
```
- **RoleBinding** - Binds Role to ServiceAccount within logging namespace
- Grants permissions for lease management (leader election)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: filebeat-kubeadm-config
  namespace: logging
subjects:
  - kind: ServiceAccount
    name: filebeat
    namespace: logging
roleRef:
  kind: Role
  name: filebeat-kubeadm-config
  apiGroup: rbac.authorization.k8s.io
```
- **RoleBinding for kubeadm-config** - Grants access to specific ConfigMap
- Allows reading kubeadm-config for cluster metadata

## **RBAC Configuration - ClusterRole**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: filebeat
  labels:
    k8s-app: filebeat
rules:
- apiGroups: [""] # "" indicates the core API group
  resources:
  - namespaces
  - pods
  - nodes
  verbs:
  - get
  - watch
  - list
- apiGroups: ["apps"]
  resources:
    - replicasets
  verbs: ["get", "list", "watch"]
```
- **ClusterRole** - Defines cluster-wide permissions
- **Core API group** permissions:
  - Can get, watch, list namespaces, pods, and nodes
- **Apps API group** permissions:
  - Can get, list, watch ReplicaSets
- Enables Kubernetes metadata enrichment for logs

## **RBAC Configuration - Roles**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: filebeat
  namespace: logging
  labels:
    k8s-app: filebeat
rules:
  - apiGroups:
      - coordination.k8s.io
    resources:
      - leases
    verbs: ["get", "create", "update"]
```
- **Role for leases** - Namespace-scoped permissions
- Grants access to Kubernetes leases (used for leader election)
- Ensures only one Filebeat instance handles certain tasks in multi-pod setups

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: filebeat-kubeadm-config
  namespace: logging
  labels:
    k8s-app: filebeat
rules:
  - apiGroups: [""]
    resources:
      - configmaps
    resourceNames:
      - kubeadm-config
    verbs: ["get"]
```
- **Role for kubeadm-config** - Specific ConfigMap access
- Can only "get" the ConfigMap named "kubeadm-config"
- Used to retrieve cluster configuration information

## **ServiceAccount**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: filebeat
  namespace: logging
  labels:
    k8s-app: filebeat
```
- **ServiceAccount** - Creates identity for Filebeat pods
- All RBAC permissions are bound to this ServiceAccount
- Pods use this ServiceAccount via `serviceAccountName: filebeat`

## **Summary**

This complete manifest deploys Filebeat as a DaemonSet that:

1. **Collects logs** from all containers on every node
2. **Enriches logs** with Kubernetes, cloud, and host metadata
3. **Ships logs** to Logstash on port 5044
4. **Maintains state** using host-mounted data directory
5. **Has proper RBAC** permissions to access Kubernetes API
6. **Runs on each node** using DaemonSet and hostNetwork
7. **Uses ConfigMap** for configuration management
8. **Tracks shipped logs** to prevent duplicates on restart

This is a production-ready configuration for centralized logging in Kubernetes clusters.