## Deploying a Kubernetes service on Azure with a specific IP addresses

Each time a Kubernetes service is created within an ACS or AKS cluster a static Azure IP address is assigned. If an IP address exists in the resource group that is not assigned to a service this will be used, otherwise a new address is requested. This means if a service is deleted and recreated it is not guaranteed to get the same IP address. Should you wish to configure the service to always receive the same IP address the load balancer can be provisioned to use a specific IP address.

This is described in the Kubernetes docs:  https://kubernetes.io/docs/concepts/services-networking/service/#type-loadbalancer.

I have been asked how to do this on Azure a number of times, the steps to follow are:

1. Create a new Reserved IP address in the same Resource Group as your Azure Container Service deployment.

    ```
    az network public-ip create -g MyClusterResourceGroup -n MyIpName --dns-name MyLabel --allocation-method Static
    ```

2. Retrieve the IP address of the newly provisioned Reserved IP:

    ```
    az network public-ip show -g MyClusterResourceGroup -n MyIpName --query "{ address: ipAddress }"
    ```

3. In the YAML file defining the service before the line `type: LoadBalancer` add:

    ```
    loadBalancerIP: 52.179.14.59
    ```

Where the IP address, in the example above `52.179.14.59`,  matches the IP address retrieved in step 2.

The YAML will now look similar to the following image:

![Load balancer in service configuration with reserved IP](https://raw.githubusercontent.com/marrobi/blog-posts/master/k8s-loadbalancer/guestbook-frontend-loadbalance-reserved.png)

4. Save the file, and redeploy the service by running the following command:

    ```
    kubectl create -f my-service.yaml
    ```

5. Type `kubectl get svc` to see the state of the services in the cluster. While the load balancer configures the rule, the `EXTERNAL-IP` of the `frontend` service appears as `<pending>`. After a few minutes, the external IP address is configured to match your reserved IP:

    ![Configure Azure load balancer](https://raw.githubusercontent.com/marrobi/blog-posts/master/k8s-loadbalancer/guestbook-external-ip-reserved.png)

### Considerations
* The Reserved IP address must be in the same resource group as the cluster.
* The IP address must be provisioned using Azure Resource Manager. If using the Azure portal select `Public IP` and choose allocation method as `Static`. Do not deploy a `Reserved IP` from the portal as this does not use Azure Resource Manager.
* If the service remains with an `EXTERNAL-IP` of `<pending>` to get more details run `kubectl describe service frontend` where `frontend` is the name of the service being deployed. If you see an error such as `User supplied IP Address XXX.XXX.XXX.XXX was not found`, run `az network public-ip list -g MyClusterResourceGroup -o table` and ensure that the reserved IP address is listed.