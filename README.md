# Tigera Deployment
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



## Deployment Procedure

### 1. Set variables
```shell
# Update with correct values!
SUPERVISOR_IP="10.139.8.6"
SUPERVISOR_USERNAME="user@showcase.tmm.broadcom.lab"
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
vcf context use $SUPERVISOR_CONTEXT:$SUPERVISOR_NAMESPACE_NAME
```

### 4. Create 'tigera-vks' VKS cluster
```shell
# Create a VKS cluster as defined in vks.yaml
kubectl apply -f vks.y

# Test commands
kubectl get cluster --watch
# wait until output shows "Available: True" 
```

### 5. Connect to 'tigera-vks' VKS cluster
```shell
# Connect to the VKS cluster
vcf context create vks --endpoint $SUPERVISOR_IP --insecure-skip-tls-verify -u $SUPERVISOR_USERNAME --workload-cluster-namespace=$SUPERVISOR_NAMESPACE_NAME --workload-cluster-name=$CLUSTER_NAME
vcf context use vks:$CLUSTER_NAME

# Test Commands
kubectl get nodes
# should show the VKS worker nodes
```

### 6. Create 'tigera-ns' namespace
```shell
# Create a namespace on the VKS cluster
kubectl create namespace $CLUSTER_NAMESPACE_NAME
kubectl config set-context --current --namespace=$CLUSTER_NAMESPACE_NAME

# Test commands
kubectl get all
# should show no resources
```

### 7. (Optional) Create Secret with Docker.io Credentials
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

# Delete VKS cluster as defined in vks.yaml
kubectl context use $SUPERVISOR_CONTEXT:$SUPERVISOR_NAMESPACE_NAME
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

# Get the control center external IP
kubectl get svc <service_name>-external 
```

