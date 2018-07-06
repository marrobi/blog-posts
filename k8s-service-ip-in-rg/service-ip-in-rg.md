# Deploy a Kubernetes Service on Azure with IP address that is in a different resource group to the cluster

When a service is deployed to Kubernetes we often need to specift a load balancerIP address. this means that if the service gets recreated it retains the same IP. By default when you deploy a service in Kubernetes on Azure that static IP address must reside in the same resource group as the cluster nodes. This causes a couple of potential problems:

- If delete an Azure Kubernetes Service Cluster cluster then the cluster resource group (starts MC_) gets deleted and lose the IP address in the resource group.  **
- If need to reassign the IP address to a different cluster then you need to move it to the new MC_ resource group.  
- Many customers want to manage properties such as IP addresses in a dedicated resource grou

In Kubernetes 1.10 we have a new feature that resolves these issues. A service annotation can be defined that specifies the resource group that the IP addresses reside in:

https://github.com/kubernetes/kubernetes/blob/v1.10.0/pkg/cloudprovider/providers/azure/azure_loadbalancer.go#L71

```
    // ServiceAnnotationLoadBalancerResourceGroup is the annotation used on the service
    // to specify the resource group of load balancer objects that are not in the same resource group as the cluster.

    ServiceAnnotationLoadBalancerResourceGroup = "service.beta.kubernetes.io/azure-load-balancer-resource-group"
```

In the following example I have specified the above annotation, and also a LoadBalancerIP. As I want to inject environment variables in the YAML prior to deployment I've used $IP_RG and $PUBLIC_IP. Along with the service is a vanilla deployment of nginx.

```yaml
kind: Service
apiVersion: v1
metadata:
  name: my-service
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-resource-group: $IP_RG
spec:
  selector:
    app: my-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  loadBalancerIP: $PUBLIC_IP
  type: LoadBalancer

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80

```

The first step is to ensure we have a cluster running a Kubernetes cluster greater than version 1.10:

```bash

CLUSTER_RG="MyAKSCluster"

az group create -n $CLUSTER_RG -l westeurope

az aks create -g $CLUSTER_RG -n cluster1 --kubernetes-version 1.10.3

```

Once we have a cluster we need to create a resource group for the IP addresses:

```bash

export IP_RG="MyIPRG"
az group create -n $IP_RG -l westeurope

```

To enable Kubernetes to attach the IP to a load balancer the Azure Service Principal used by the cluster must be granted "Network Contributor" rights to the resource group:

```bash

CLIENT_ID=$(az aks show -n cluster1 -g $CLUSTER_RG --query servicePrincipalProfile.clientId | sed 's/"//g')
az role assignment create --role "Network Contributor" --assignee $CLIENT_ID --resource-group $IP_RG

```

Create an Public IP address and retrieve the IP:

```bash

az network public-ip create -g $IP_RG -n service-ip --dns-name demo-service-ip  --allocation-method Static

export PUBLIC_IP=$(az  network public-ip show -n service-ip -g $IP_RG --query ipAddress | sed 's/"//g')

```

Retrieve the cluster credentials and deploy the service:

```bash

az aks get-credentials -g $CLUSTER_RG -n cluster1

envsubst < service-ip-in-rg.yaml | kubectl apply -f -

```

We can check the status of the service with:

```bash

kubectl describe service my-service

```

After a few minutes we can see that the load balancer is created and IP address assigned to the service:

```text

Name:                     my-service
Namespace:                default
Labels:                   <none>
Annotations:              kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"v1","kind":"Service","metadata":{"annotations":{"service.beta.kubernetes.io/azure-load-balancer-resource-group":"MyIPRG"},"name":"my-ser...
                          service.beta.kubernetes.io/azure-load-balancer-resource-group=MyIPRG
Selector:                 app=my-app
Type:                     LoadBalancer
IP:                       10.0.197.154
IP:                       13.73.182.XX
LoadBalancer Ingress:     13.73.182.XX
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  30512/TCP
Endpoints:                10.244.0.4:80,10.244.1.3:80,10.244.2.6:80
Session Affinity:         None
External Traffic Policy:  Cluster
Events:
  Type     Reason                      Age              From                Message
  ----     ------                      ----             ----                -------
  Warning  CreatingLoadBalancerFailed  6m (x3 over 7m)  service-controller  Error creating load balancer (will retry): failed to ensure load balancer for service default/my-service: network.PublicIPAddressesClient#List: Failure responding to
request: StatusCode=403 -- Original Error: autorest/azure: Service returned an error. Status=403 Code="AuthorizationFailed" Message="The client 'XXX' with object id 'XXX' does
not have authorization to perform action 'Microsoft.Network/publicIPAddresses/read' over scope '/subscriptions/XXX/resourceGroups/MyIPRG/providers/Microsoft.Network'."
  Normal   EnsuringLoadBalancer        6m (x4 over 7m)  service-controller  Ensuring load balancer
  Normal   EnsuredLoadBalancer         4m               service-controller  Ensured load balancer

```

We can now verify access to nginx using our newly created IP:

```bash

curl $PUBLIC_IP


<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

```