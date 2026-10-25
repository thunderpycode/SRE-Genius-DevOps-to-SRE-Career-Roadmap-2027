# ☸️ Kubernetes (K8s) Production Master Reference for SREs

A complete, production-grade guide and reference manual covering cluster operations, workload debugging, network diagnostics, storage, security, and real-time incident resolution.

---

## 📑 Quick Navigation
1. [Context, Configuration & Cluster State](#1-context-configuration--cluster-state)
2. [Node Management & Health Checks](#2-node-management--health-checks)
3. [Pod Operations & Triage Lifecycle](#3-pod-operations--triage-lifecycle)
4. [Deployments, Rollouts & DaemonSets](#4-deployments-rollouts--daemonsets)
5. [Service Networking, Ingress & DNS](#5-service-networking-ingress--dns)
6. [ConfigMaps, Secrets & Volumes](#6-configmaps-secrets--volumes)
7. [Resource Optimization & HPA](#7-resource-optimization--hpa)
8. [RBAC, Security & ServiceAccounts](#8-rbac-security--serviceaccounts)
9. [Advanced Live Debugging Techniques](#9-advanced-live-debugging-techniques)
10. [Production Incident Triage Matrix](#10-production-incident-triage-matrix)

---

## 1. Context, Configuration & Cluster State

### Context & Configuration Switching
```bash
# View active context and cluster endpoints
kubectl config current-context
kubectl cluster-info

# List all available contexts in ~/.kube/config
kubectl config get-contexts

# Switch context (cluster/environment)
kubectl config use-context <context-name>

# Set default namespace for current context to avoid typing -n <namespace>
kubectl config set-context --current --namespace=<namespace>

# View merged kubeconfig
kubectl config view --minify
```

### API Resources & Versions
```bash
# Check API server compatibility and available resource types
kubectl api-resources

# Check API versions supported by the cluster
kubectl api-versions

# Explain resource specifications and field schemas (e.g., Pod spec)
kubectl explain pod.spec.containers.livenessProbe
```

---

## 2. Node Management & Health Checks

### Node Inspection
```bash
# View all nodes with IPs, OS image, kernel, and container runtime version
kubectl get nodes -o wide

# Display node resource utilization (CPU & Memory)
kubectl top nodes

# Inspect conditions (MemoryPressure, DiskPressure, PIDPressure, Ready)
kubectl describe node <node-name> | grep -A 5 Conditions:

# View taints applied to a node
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints
```

### Node Maintenance & Drainage
```bash
# Prevent new pods from scheduling onto a node
kubectl cordon <node-name>

# Safely evict running pods for node maintenance (OS upgrade / patch)
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data --force

# Restore scheduling to the node after maintenance
kubectl uncordon <node-name>
```

---

## 3. Pod Operations & Triage Lifecycle

### Querying & Filtering
```bash
# List pods across all namespaces sorted by restart count
kubectl get pods -A --sort-by='.status.containerStatuses[0].restartCount'

# List pods running on a specific node
kubectl get pods -A --field-selector spec.nodeName=<node-name> -o wide

# Find all pods that are not in 'Running' or 'Completed' state
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded

# Filter pods by label selector
kubectl get pods -n <namespace> -l app=payment-service,env=production
```

### Log Streaming & Pod Inspection
```bash
# Stream live logs from a specific container inside a multi-container pod
kubectl logs -f <pod-name> -c <container-name> -n <namespace>

# View logs from a previous container instance that crashed/restarted
kubectl logs <pod-name> -n <namespace> --previous --tail=200

# Stream logs from all pods matching a label selector
kubectl logs -f -l app=api-gateway -n <namespace> --max-log-requests=10

# Inspect pod events (scheduling, image pulls, probe failures, OOM kills)
kubectl describe pod <pod-name> -n <namespace>
```

---

## 4. Deployments, Rollouts & DaemonSets

### Deployments & Scaling
```bash
# Check status of deployment workloads
kubectl get deployments -n <namespace> -o wide

# Scale deployment replicas immediately
kubectl scale deployment/<deployment-name> --replicas=10 -n <namespace>

# Trigger a zero-downtime rolling restart (re-reads secrets & configs)
kubectl rollout restart deployment/<deployment-name> -n <namespace>

# Check rollout progress in real time
kubectl rollout status deployment/<deployment-name> -n <namespace>

# View rollout history (revisions)
kubectl rollout history deployment/<deployment-name> -n <namespace>

# Roll back to the immediately previous revision
kubectl rollout undo deployment/<deployment-name> -n <namespace>

# Roll back to a specific revision number
kubectl rollout undo deployment/<deployment-name> --to-revision=3 -n <namespace>
```

---

## 5. Service Networking, Ingress & DNS

### Services & Ingress Inspection
```bash
# List all services, internal ClusterIPs, and exposed ports
kubectl get svc -A -o wide

# Verify Service Endpoints (checks if pod selectors are correctly matching pods)
kubectl get endpoints <service-name> -n <namespace>
kubectl get endpointslices -n <namespace>

# Inspect Ingress routing rules and associated controllers
kubectl get ingress -A
kubectl describe ingress <ingress-name> -n <namespace>
```

### CoreDNS & In-Cluster Connectivity Testing
```bash
# Forward a remote service or pod port directly to your localhost
kubectl port-forward svc/<service-name> 8080:<service-port> -n <namespace>

# Verify CoreDNS pods are running healthy
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Run an ephemeral pod to test internal DNS resolution
kubectl run dns-test --rm -it --image=busybox:1.28 --restart=Never -- \
  nslookup <service-name>.<namespace>.svc.cluster.local
```

---

## 6. ConfigMaps, Secrets & Volumes

### Managing Config & Secrets
```bash
# View ConfigMap details
kubectl get configmap <cm-name> -n <namespace> -o yaml

# View decoded Secret values directly from terminal
kubectl get secret <secret-name> -n <namespace> -o jsonpath="{.data}" | jq 'map_values(@base64d)'

# Create a Secret from a file
kubectl create secret generic app-creds --from-file=credentials.json -n <namespace>
```

### Persistent Volumes (PV) & PVCs
```bash
# List PersistentVolumeClaims and binding statuses
kubectl get pvc -A

# Check for unbound or failing PersistentVolumes
kubectl get pv | grep -v Bound
```

---

## 7. Resource Optimization & HPA

### Monitoring Metrics & Autoscalers
```bash
# Check pod CPU and memory utilization against limits/requests
kubectl top pods -n <namespace> --sort-by=memory
kubectl top pods -n <namespace> --sort-by=cpu

# Inspect Horizontal Pod Autoscaler (HPA) metrics, targets, and current replicas
kubectl get hpa -A
kubectl describe hpa <hpa-name> -n <namespace>
```

---

## 8. RBAC, Security & ServiceAccounts

### Auth & Permissions Auditing
```bash
# Test if your current user or a ServiceAccount can perform an action
kubectl auth can-i create deployments --namespace=<namespace>
kubectl auth can-i delete pods --as=system:serviceaccount:<namespace>:<serviceaccount-name>

# Inspect ClusterRoles and RoleBindings
kubectl get rolebindings,clusterrolebindings -A -o wide
```

---

## 9. Advanced Live Debugging Techniques

### Exec & Ephemeral Diagnostic Containers
```bash
# Interactive shell into a running container
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh

# Attach an ephemeral debug container with full diagnostic toolset to a broken pod
kubectl debug -it <target-pod> -n <namespace> \
  --image=nicolaka/netshoot \
  --target=<target-container>

# Copy a configuration file from a pod to your local machine
kubectl cp <namespace>/<pod-name>:/etc/app/config.yaml ./config.yaml
```

---

## 10. Production Incident Triage Matrix

| Status / Failure Mode | Root Cause | Immediate Diagnostic Command | Mitigation Steps |
| :--- | :--- | :--- | :--- |
| **CrashLoopBackOff** | Application panic, missing configuration, failed probe | `kubectl logs <pod> -n <ns> --previous` | Check stdout errors; verify environment variables and ConfigMap keys. |
| **OOMKilled (Exit Code 137)** | Process exceeded container memory limit | `kubectl describe pod <pod> -n <ns> \| grep -i oom` | Increase `resources.limits.memory` or profile application memory leak. |
| **ImagePullBackOff / ErrImagePull** | Invalid image tag, wrong registry URL, missing ImagePullSecret | `kubectl describe pod <pod> -n <ns> \| grep -A 8 Events` | Verify image exists on registry; ensure correct `imagePullSecrets` are bound. |
| **Pending** | Cluster has insufficient CPU/memory, or node selectors/taints mismatch | `kubectl describe pod <pod> -n <ns> \| grep -A 5 Events` | Add nodes to node pool, relax resource requests, or verify node taints. |
| **CreateContainerConfigError** | Missing ConfigMap or Secret referenced as `env` or `volumeMount` | `kubectl get events -n <ns> --field-selector reason=Failed` | Verify referenced ConfigMap or Secret name and key exist in the namespace. |
| **Endpoints 0 / 502 Bad Gateway** | Readiness probe failing or service label selector mismatch | `kubectl get endpoints <svc-name> -n <ns>` | Compare `spec.selector` in Service with `metadata.labels` in Pod spec; check probe endpoint. |