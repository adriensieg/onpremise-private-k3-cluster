# Kubernetes Concepts

https://spacelift.io/blog/kubernetes-fedramp#3-apply-network-segmentation-and-enforce-traffic-policies

How does FedRAMP impact your Kubernetes clusters?
- Secure container images
- Harden your Kubernetes control plane
- Apply network segmentation and enforce traffic policies
- Control deployments with Admission Controllers
- Maintain pod and namespace isolation
- Enforce pod security standards
- Implement cluster runtime security and continuous monitoring

[SUSE 269 Stupid Simple Kubernetes v07.pdf](https://github.com/user-attachments/files/24552746/SUSE.269.Stupid.Simple.Kubernetes.v07.pdf)

### Kubernetes Hardware Structure
- Nodes
- Cluster
- Persistent Volumes

### Kubernetes Software Components
- Container
- Pods
- Deployments
- Stateful Sets
- DaemonSets
- Services
- ConfigMaps

### External Traffic
- ClusterIP
- NodePort
- LoadBalancer
- Ingress

### Scalability Explained
- Horizontal Pod Autoscaling (HPA)
- Vertical Pod Autoscaling (VPA)
- Cluster Autoscaling (CA)

### Service Mesh
- Ingress Controller vs.
- API Gateway vs. Service
- Mesh


## 1. Cluster Basics
- Cluster – A group of machines managed by Kubernetes
- Node – One machine in the cluster
- Control Plane – Decides what runs and where
- Worker Node – Runs application Pods

## 2. Control Plane Components
- kube-apiserver – REST API for all cluster actions
- etcd – Stores all cluster data
- kube-scheduler – Chooses which node runs a Pod
- kube-controller-manager – Keeps desired state
- cloud-controller-manager – Talks to cloud APIs

## 3. Node Components
- kubelet – Runs Pods on the node
- kube-proxy – Handles Service networking
- Container Runtime – Runs containers (containerd, CRI-O)

## 4. Pods & Containers
- Pod – Smallest deployable unit
- Container – Application process
- Init Container – Runs before main containers
- Ephemeral Container – Temporary debug container

## 5. Metadata & Selection
- Label – Key-value tag for objects
- Selector – Matches labels
- Annotation – Extra metadata, not used for selection
- Namespace – Logical object isolation

## 6. Workload Controllers
- Deployment – Manages stateless Pods
- ReplicaSet – Keeps Pod count stable
- StatefulSet – Stable Pods with storage
- DaemonSet – One Pod per node
- Job – Runs until completion
- CronJob – Runs Jobs on schedule

## 7. Configuration
- ConfigMap – Non-sensitive configuration data
- Secret – Sensitive data (passwords, tokens)
- Environment Variable – Config injected into container
- Volume Mount – File-based config injection

## 8. Networking Basics
- Pod IP – Unique IP per Pod
- CNI – Plugin for Pod networking
- Service – Stable access to Pods
- ClusterIP – Internal Service
- NodePort – Exposes Service on node port
- LoadBalancer – Cloud external load balancer
- ExternalName – DNS alias Service

## 9. Traffic Routing
- Endpoints – Pod IPs behind a Service
- EndpointSlice – Scalable Endpoints
- Session Affinity – Sends same client to same Pod

## 10. Ingress
- Ingress – HTTP/HTTPS routing rules
- Ingress Controller – Implements Ingress logic
- IngressClass – Selects controller
- Path Routing – Route by URL path
- Host Routing – Route by domain
- TLS Termination – HTTPS ends at Ingress

## 11. Storage Basics
- Volume – Storage attached to Pod
- emptyDir – Temporary Pod storage
- hostPath – Node filesystem mount

## 12. Persistent Storage
- PersistentVolume (PV) – Cluster storage resource
- PersistentVolumeClaim (PVC) – Request for storage
- StorageClass – Defines storage type
- Access Modes – How storage is mounted
- Reclaim Policy – What happens after release
- CSI – Storage driver interface

## 13. Scheduling & Placement
- Scheduler – Assigns Pods to nodes
- Node Selector – Simple node constraint
- Node Affinity – Advanced node selection
- Pod Affinity – Place Pods together
- Pod Anti-Affinity – Keep Pods apart
- Taint – Blocks Pods from node
- Toleration – Allows Pod on tainted node
- Topology Spread – Even Pod distribution

## 14. Resource Management
- Requests – Guaranteed resources
- Limits – Max allowed resources
- QoS Class – Pod priority level
- OOMKilled – Container killed for memory overuse
- ResourceQuota – Namespace resource limit
- LimitRange – Default/min/max per Pod

## 15. Autoscaling
- HPA – Scales Pods horizontally
- VPA – Adjusts Pod resources
- Cluster Autoscaler – Scales nodes
- Metrics Server – Provides resource metrics

## 16. Security & Access
- Authentication – Identity verification
- Authorization – Permission check
- RBAC – Role-based access control
- Role / ClusterRole – Permission set
- RoleBinding – Assigns permissions
- ServiceAccount – Pod identity

## 17. Pod Security
- SecurityContext – Pod security settings
- Pod Security Admission – Enforces security rules
- Capabilities – Linux privilege control
- Seccomp – Syscall filtering
- AppArmor – Process access control

## 18. Network Security
- NetworkPolicy – Pod traffic rules
- Ingress Policy – Allowed incoming traffic
- Egress Policy – Allowed outgoing traffic
- Default Deny – Block all traffic by default

## 19. Observability
- Logs – Application output
- Events – Cluster activity records
- Metrics – Resource usage data
- Liveness Probe – Restarts unhealthy container
- Readiness Probe – Controls traffic flow
- Startup Probe – Handles slow startups

## 20. Updates & Lifecycle
- Rolling Update – Gradual Pod replacement
- Recreate Strategy – Stop then start Pods
- Graceful Termination – Clean shutdown
- PreStop Hook – Runs before shutdown
- PostStart Hook – Runs after start

## 21. Extensibility
- Custom Resource (CR) – Custom Kubernetes object
- CRD – Defines custom resources
- Operator – Manages complex apps
- Admission Controller – Intercepts API requests
- Mutating Webhook – Modifies requests
- Validating Webhook – Validates requests

## 22. Advanced Topics
- Service Mesh – L7 traffic management
- Sidecar – Helper container
- mTLS – Encrypted Pod traffic
- Canary Deployment – Gradual release
- Blue-Green Deployment – Two environments
- Multi-Cluster – Multiple Kubernetes clusters
- GPU Scheduling – GPU workload support
- Device Plugin – Hardware integration
- RuntimeClass – Select container runtime
- eBPF – High-performance networking








