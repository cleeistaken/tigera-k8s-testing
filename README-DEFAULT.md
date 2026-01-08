# VKS Default Deployment
## Requirements
### CLI Tools
* kubectl cli v1.34: https://dl.k8s.io/release/v1.34.0/bin/linux/amd64/kubectl
* vcf cli v9.0.1: https://packages.broadcom.com/artifactory/vcf-distro/vcf-cli/linux/amd64/v9.0.1/
* helm cli v3.19: https://get.helm.sh/helm-v3.19.0-linux-amd64.tar.gz

### Get files
```shell
git clone https://github.com/cleeistaken/tigera-k8s-testing.git
```

### Required vSphere Steps
1. In WCP, create a name space names '**tigera**'
2. Add '**vsan-esa-default-policy-raid5**' storage policy to '**tigera**' namespace
3. Create and add a VM class named '**vks-8-32-class**' and add it to '**tigera**' namespace
   * Define the CPU and RAM. This class will be used for the VKS worker nodes
4. Add '**best-effort-medium**' VM class to '**tigera**' namespace

**Note**: These steps required access to vCenter and typically done by the infrastructure administrator.

## Deployment Procedure

### 1. Set variables
```shell
# Update with correct values!
SUPERVISOR_IP="10.138.216.198"
SUPERVISOR_USERNAME="<username>@showcase.tmm.broadcom.lab"
SUPERVISOR_NAMESPACE_NAME="tigera"
SUPERVISOR_CONTEXT="tigera-ctx"
CLUSTER_NAME="tigera-vks"
CLUSTER_NAMESPACE_NAME="tigera-vks-ns"
```

### 2. Clean kubectl and vcf configs
```shell
rm ~/.kube/config
rm -rf ~/.config/vcf/
```

### 3. Create context on supervisor
```shell
# Create a context named 'tigera-ctx'
vcf context create $SUPERVISOR_CONTEXT --endpoint $SUPERVISOR_IP --insecure-skip-tls-verify -u $SUPERVISOR_USERNAME 

# Expected output:
# [i] Some initialization of the CLI is required.
# [i] Let's set things up for you.  This will just take a few seconds.
# 
# [i] 
# [i] Initialization done!
# [i] ==
# [i] Auth type vSphere SSO detected. Proceeding for authentication...
# Provide Password: 
# 
# Logged in successfully.
#
# You have access to the following contexts:
#    tigera-ctx
#    tigera-ctx:tigera
#
# If the namespace context you wish to use is not in this list, you may need to
# refresh the context again, or contact your cluster administrator.
#
# To change context, use `vcf context use <context_name>`
# [ok] successfully created context: tigera-ctx
# [ok] successfully created context: tigera-ctx:tigera
```

### 4. Set supervisor context
```shell
# Set context
vcf context use $SUPERVISOR_CONTEXT:$SUPERVISOR_NAMESPACE_NAME

# Expected output
# [ok] Token is still active. Skipped the token refresh for context "tigera-ctx:tigera"
# [i] Successfully activated context 'tigera-ctx:tigera' (Type: kubernetes) 
# [i] Fetching recommended plugins for active context 'tigera-ctx:tigera'...
# [i] No image repository override information was found
# [ok] All recommended plugins are already installed and up-to-date. 
```

### 5. Create VKS cluster
```shell
# Create a VKS cluster using the default CNI
sed "s/tigera-vks/$CLUSTER_NAME/" vks.yaml  | kubectl apply -f -

# OR create a VKS cluster using the VKS version of Calico 

sed "s/tigera-vks/$CLUSTER_NAME/" vks-cni-calico.yaml  | kubectl apply -f -

# Expected output:
# calicoconfig.cni.tanzu.vmware.com/tigera-vks created
# clusterbootstrap.run.tanzu.vmware.com/tigera-vks created
# Warning: cluster.x-k8s.io/v1beta1 Cluster is deprecated; use cluster.x-k8s.io/v1beta2 Cluster
# cluster.cluster.x-k8s.io/tigera-vks created
# 
# or:
# calicoconfig.cni.tanzu.vmware.com/tigera-vks unchanged
# clusterbootstrap.run.tanzu.vmware.com/tigera-vks unchanged
# Warning: cluster.x-k8s.io/v1beta1 Cluster is deprecated; use cluster.x-k8s.io/v1beta2 Cluster
# cluster.cluster.x-k8s.io/tigera-vks created

# Test commands
kubectl get cluster --watch

# Expected output:
# NAME         CLUSTERCLASS             AVAILABLE   CP DESIRED   CP AVAILABLE   CP UP-TO-DATE   W DESIRED   W AVAILABLE   W UP-TO-DATE   PHASE         AGE   VERSION
# tigera-vks   builtin-generic-v3.5.0   False       1            0              1               4           0             4              Provisioned   67s   v1.34.1+vmware.1
# [...]
# tigera-vks   builtin-generic-v3.5.0   False       1            1              1               4           3             4              Provisioned   3m18s   v1.34.1+vmware.1
# tigera-vks   builtin-generic-v3.5.0   True        1            1              1               4           3             4              Provisioned   3m18s   v1.34.1+vmware.1
#
# Note: wait until output shows "Available: True" 
```

### 6. Connect to VKS cluster
```shell
# Connect to the VKS cluster
vcf context create vks --endpoint $SUPERVISOR_IP --insecure-skip-tls-verify -u $SUPERVISOR_USERNAME --workload-cluster-namespace=$SUPERVISOR_NAMESPACE_NAME --workload-cluster-name=$CLUSTER_NAME

# Expected output:
# [i] Logging in to Kubernetes cluster (tigera-vks) (tigera)
# [i] Successfully logged in to Kubernetes cluster 10.138.216.200
#
# You have access to the following contexts:
#    vks
#    vks:tigera-vks
#
# If the namespace context you wish to use is not in this list, you may need to
# refresh the context again, or contact your cluster administrator.
# 
# To change context, use `vcf context use <context_name>`
# [ok] successfully created context: vks
# [ok] successfully created context: vks:tigera-vks
```

### 7 Use VKS cluster context
```shell

# Use context
vcf context use vks:$CLUSTER_NAME

# Expected output: 
# [ok] Token is still active. Skipped the token refresh for context "vks:tigera-vks"
# [i] Successfully activated context 'vks:tigera-vks' (Type: kubernetes) 
# [i] Fetching recommended plugins for active context

# Test Commands

# Get Nodes
kubectl get nodes -o wide

# Expected output: 
# NAME                                       STATUS   ROLES           AGE   VERSION            INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
# tigera-vks-node-pool-1-8jzgf-bhrz2-vmnw7   Ready    <none>          36m   v1.34.1+vmware.1   172.26.0.5    <none>        Ubuntu 24.04.3 LTS   6.8.0-86-generic   containerd://2.1.4+vmware.3-fips
# tigera-vks-node-pool-2-zd45h-59jlx-mrr64   Ready    <none>          36m   v1.34.1+vmware.1   172.26.0.6    <none>        Ubuntu 24.04.3 LTS   6.8.0-86-generic   containerd://2.1.4+vmware.3-fips
# tigera-vks-node-pool-3-v8j28-bjm5t-w2h2q   Ready    <none>          36m   v1.34.1+vmware.1   172.26.0.4    <none>        Ubuntu 24.04.3 LTS   6.8.0-86-generic   containerd://2.1.4+vmware.3-fips
# tigera-vks-node-pool-4-btqp5-z4ts9-sx9bd   Ready    <none>          36m   v1.34.1+vmware.1   172.26.0.7    <none>        Ubuntu 24.04.3 LTS   6.8.0-86-generic   containerd://2.1.4+vmware.3-fips
# tigera-vks-trdx9-gmtbm                     Ready    control-plane   38m   v1.34.1+vmware.1   172.26.0.3    <none>        Ubuntu 24.04.3 LTS   6.8.0-86-generic   containerd://2.1.4+vmware.3-fips

# Get CNI
kubectl get daemonsets -n kube-system

# Expected output: 
# NAME          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
# calico-node   5         5         5       5            5           kubernetes.io/os=linux   38m
# kube-proxy    5         5         5       5            5           kubernetes.io/os=linux   38m
```

### 8. Create VKS namespace
```shell
# Create a namespace on the VKS cluster
kubectl create namespace $CLUSTER_NAMESPACE_NAME

# Expected output:
# namespace/tigera-ns created
#
# or:
# Error from server (AlreadyExists): namespaces "tigera-ns" already exists

# Set context
kubectl config set-context --current --namespace=$CLUSTER_NAMESPACE_NAME

# Expected output:
# Context "vks:tigera-vks" modified.

# Test commands
kubectl get all
```

### 9. (Optional) Create Secret with Docker.io Credentials
May be required if the deployment hits errors about the site hitting image pull limits.
```shell
# Create secret with Docker login credentials in Kubernetes
kubectl create secret docker-registry regcred \
  --docker-server=docker.io \
  --docker-username=<docker_username> \
  --docker-password=<docker_password> \
  --docker-email=<docker_email> 
  --namespace=$CLUSTER_NAMESPACE_NAME

# Automatically use credentials for all pods in namespace 
kubectl patch serviceaccount default \
  -p '{"imagePullSecrets": [{"name": "regcred"}]}'
```


## Cleanup Procedure

```shell
#Delete the namespace
kubectl delete namespace $CLUSTER_NAMESPACE_NAME

# Switch to supervisor context 
vcf context use $SUPERVISOR_CONTEXT:$SUPERVISOR_NAMESPACE_NAME

# Delete VKS cluster as defined in vks.yaml
kubectl delete -f vks.yaml
```


## Troubleshooting

### Useful Commands
```shell
# Refresh contexts
vcf context refresh $SUPERVISOR_CONTEXT:$SUPERVISOR_NAMESPACE_NAME
vcf context refresh vks:$CLUSTER_NAME

# List all
kubectl get all

# Get detailed pod information
kubectl get pods -o wide

# Get container logs
kubectl logs -f <container name>

# Get all services
kubectl get svc

# Expose a service
kubectl expose service <service_name>  --type=LoadBalancer --name=<service_name>-external

# Get the external IP
kubectl get svc <service_name>-external

# Get TKR releases
kubectl get tkr -l '!kubernetes.vmware.com/kubernetesrelease'

# Get TKE releaase specs
# e.g. kubectl get tkr 'v1.34.1---vmware.1-vkr.4' -o yaml
kubectl get tkr TKR_NAME -o yaml  

# Get all cluster classes
kubectl -n tigera get clusterclass -A

kubectl get secret 
kubectl get secret tigera-vks-no-cni-1-33-cc-ssh-password -o yaml
echo "" | base64 -d 

kubectl get secret tigera-calico-vks-ssh-password -o json | jq -r '.data."ssh-passwordkey"' | base64 -d


kubectl patch packageinstall <pkgi-name> -n <namespace> --type='merge' -p '{"spec":{"paused":true}}'
```

