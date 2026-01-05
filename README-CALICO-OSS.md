# VKS Calico OSS Deployment
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
CLUSTER_NAMESPACE_NAME="tigera-ns"
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

# Set context
vcf context use $SUPERVISOR_CONTEXT:$SUPERVISOR_NAMESPACE_NAME

# Expected output
# [ok] Token is still active. Skipped the token refresh for context "tigera-ctx:tigera"
# [i] Successfully activated context 'tigera-ctx:tigera' (Type: kubernetes) 
# [i] Fetching recommended plugins for active context 'tigera-ctx:tigera'...
# [i] No image repository override information was found
# [ok] All recommended plugins are already installed and up-to-date. 
```

### 4. Create 'builtin-generic-v3.5.0-tigera' clusterclass
```shell
# Check current clusterclasses
kubectl get clusterclass 

# Expected output
# NAME                     VARIABLES READY   AGE
# builtin-generic-v3.1.0   True              xxd
# builtin-generic-v3.2.0   True              xxd
# builtin-generic-v3.3.0   True              xxd

# Add the tigera customclass
# Note: This custom class opens ports 179 and 5474 in the postKubeadm sections
kubectl apply -f builtin-generic-v3.5.0-tigera.yaml 

# Expected output
# clusterclass.cluster.x-k8s.io/builtin-generic-v3.5.0-tigera created


# Ensure clusterclass is created
kubectl get clusterclass 

# Expected output
NAME                            VARIABLES READY   AGE
# builtin-generic-v3.1.0          True              xxd
# builtin-generic-v3.2.0          True              xxd
# builtin-generic-v3.3.0          True              xxd
# builtin-generic-v3.5.0-tigera   True              xxd <--------

```

### 5. Create 'tigera-vks' VKS cluster using custom clusterclass
```shell
# Create a VKS cluster as defined in vks.yaml
kubectl apply -f vks-cni-calico-tigera.yaml

# Expected output:
# calicoconfig.cni.tanzu.vmware.com/tigera-calico-vks created
# clusterbootstrap.run.tanzu.vmware.com/tigera-calico-vks created
# cluster.cluster.x-k8s.io/tigera-calico-vks created
# 
# or:
# calicoconfig.cni.tanzu.vmware.com/tigera-calico-vks unchanged
# clusterbootstrap.run.tanzu.vmware.com/tigera-calico-vks unchanged
# cluster.cluster.x-k8s.io/tigera-calico-vks configured

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

### 6. Connect to 'tigera-vks' VKS cluster
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

### 7. Disable automatic reconciliation
```shell
# Get package names in 'vmware-system-tkg' namespace
kubectl get packageinstall -n vmware-system-tkg

# Expected output:
# Note: 'calico.tanzu.vmware.com' description shows 'Reconcile succeeded'

# NAME                                           PACKAGE NAME                                  PACKAGE VERSION              DESCRIPTION           AGE     PAUSED
# tigera-calico-vks-calico                       calico.tanzu.vmware.com                       3.30.0+vmware.2-fips-tkg.1   Reconcile succeeded   6m41s   
# tigera-calico-vks-gateway-api                  gateway-api.tanzu.vmware.com                  1.2.1+vmware.2-tkg.1         Reconcile succeeded   6m40s   
# tigera-calico-vks-guest-cluster-auth-service   guest-cluster-auth-service.tanzu.vmware.com   1.4.2+vmware.1-tkg.1         Reconcile succeeded   6m41s   
# tigera-calico-vks-metrics-server               metrics-server.tanzu.vmware.com               0.7.2+vmware.7-fips-tkg.1    Reconcile succeeded   6m40s   
# tigera-calico-vks-pinniped                     pinniped.tanzu.vmware.com                     0.39.0+vmware.2-tkg.1        Reconcile succeeded   6m38s   
# tigera-calico-vks-secretgen-controller         secretgen-controller.tanzu.vmware.com         0.19.1+vmware.2-fips-tkg.1   Reconcile succeeded   6m39s   
# tigera-calico-vks-vsphere-cpi                  vsphere-cpi.tanzu.vmware.com                  1.33.0+vmware.1-tkg.1        Reconcile succeeded   6m41s   
# tigera-calico-vks-vsphere-pv-csi               vsphere-pv-csi.tanzu.vmware.com               3.5.0+vmware.1-tkg.1         Reconcile succeeded   6m41s   

# Disable reconciliation process
kubectl patch packageinstall tigera-calico-vks-calico 
-n vmware-system-tkg \
--type='merge' \
-p '{"spec":{"paused":true}}'

# Expected output:
# packageinstall.packaging.carvel.dev/tigera-calico-vks-calico patched

# Verify reconciliation
kubectl get packageinstall -n vmware-system-tkg

# Expected output:
# NAME                                           PACKAGE NAME                                  PACKAGE VERSION              DESCRIPTION           AGE   PAUSED
# tigera-calico-vks-calico                       calico.tanzu.vmware.com                       3.30.0+vmware.2-fips-tkg.1   Reconcile succeeded   12m   true
# tigera-calico-vks-gateway-api                  gateway-api.tanzu.vmware.com                  1.2.1+vmware.2-tkg.1         Reconcile succeeded   12m   
# tigera-calico-vks-guest-cluster-auth-service   guest-cluster-auth-service.tanzu.vmware.com   1.4.2+vmware.1-tkg.1         Reconcile succeeded   12m   
# tigera-calico-vks-metrics-server               metrics-server.tanzu.vmware.com               0.7.2+vmware.7-fips-tkg.1    Reconcile succeeded   12m   
# tigera-calico-vks-pinniped                     pinniped.tanzu.vmware.com                     0.39.0+vmware.2-tkg.1        Reconcile succeeded   12m   
# tigera-calico-vks-secretgen-controller         secretgen-controller.tanzu.vmware.com         0.19.1+vmware.2-fips-tkg.1   Reconcile succeeded   12m   
# tigera-calico-vks-vsphere-cpi                  vsphere-cpi.tanzu.vmware.com                  1.33.0+vmware.1-tkg.1        Reconcile succeeded   12m   
# tigera-calico-vks-vsphere-pv-csi               vsphere-pv-csi.tanzu.vmware.com               3.5.0+vmware.1-tkg.1         Reconcile succeeded   12m   
```

### 8. Delete calico-node daemonset
```shell
# Check if 'calico-node' daemonset exists
kubectl get ds calico-node -n kube-system

# Expected output:
# NAME          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
# calico-node   5         5         5       5            5           kubernetes.io/os=linux   14m


# Delete daemonset
kubectl delete ds calico-node -n kube-system

# Expected output:
# daemonset.apps "calico-node" deleted from kube-system namespace


# Verify daemonset is deleted
kubectl get ds calico-node -n kube-system

# Expected output:
# Error from server (NotFound): daemonsets.apps "calico-node" not found
```


### 9. Create 'tigera-ns' namespace
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

### 10. (Optional) Create Secret with Docker.io Credentials
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

### 11. Install Calico OSS
Ref. https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart
```shell
# Create operator
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.3/manifests/tigera-operator.yaml

# Expected output:
# namespace/tigera-operator created
# serviceaccount/tigera-operator created
# clusterrole.rbac.authorization.k8s.io/tigera-operator-secrets created
# clusterrole.rbac.authorization.k8s.io/tigera-operator created
# clusterrolebinding.rbac.authorization.k8s.io/tigera-operator created
# rolebinding.rbac.authorization.k8s.io/tigera-operator-secrets created
# deployment.apps/tigera-operator created


# Create resources
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.3/manifests/custom-resources.yaml

# Expected output:
# installation.operator.tigera.io/default created
# apiserver.operator.tigera.io/default created
# goldmane.operator.tigera.io/default created
# whisker.operator.tigera.io/default created


# Check tigera status
# Note: Wait until all set to 'Available' and no items 'Degraded'
watch kubectl get tigerastatus

# Expected output:
# NAME        AVAILABLE   PROGRESSING   DEGRADED   SINCE
# apiserver   True        False         False      99s
# calico      True        False         False      74s
# goldmane    True        False         False      99s
# ippools     True        False         False      119s
# whisker     True        False         False      104s
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

