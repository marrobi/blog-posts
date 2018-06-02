# Enforcing Network Policies using kube-router on AKS

Corporate security policy often requires the flow of traffic to be restricted between between Kubernetes pods. This is similar to how switch access control lists restrict traffic between physical servers. This functionality Kubernetes the traffic flow is configured using network policies.

There are a number of projects that support [network policy](https://kubernetes.io/docs/concepts/services-networking/network-policies/) enforcement. The majority require a specific [network plugin](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/) to be deployed. As the Azure Kubernetes Service is a managed service we do not have the flexibility to choose the network plug in that is deployed. The default is kubenet, or if using advanced networking AKS uses the Azure CNI plugin, for more details see [https://docs.microsoft.com/en-us/azure/aks/networking-overview](https://docs.microsoft.com/en-us/azure/aks/networking-overview). At the time of writing the Azure CNI plugin does not support network policy enforcement.

With the above in mind I often get asked how network policies can be enforced on AKS. This led me to search for a project that can enforce network policies without requiring a specific CNI plugin. Kube-router [https://github.com/cloudnativelabs/kube-router](https://github.com/cloudnativelabs/kube-router) can be deployed as a daemonset and offers is this functionality.

*Please note:* The steps below are a result of my attempts to enable kube-router to work on AKS. My initial impressions are that it functions correctly, but please be aware kube-router is still in beta and no support is offered by Microsoft for the project, see [https://docs.microsoft.com/en-us/azure/aks/networking-overview#frequently-asked-questions](https://docs.microsoft.com/en-us/azure/aks/networking-overview#frequently-asked-questions).

I took one of [sample daemonset deployment files](https://github.com/cloudnativelabs/kube-router/tree/master/daemonset), and removed any references to CNI configuration and modified the volume mounts to match the configuration of AKS nodes:

```yaml
        - name: kubeconfig
            hostPath:
                path: /var/lib/kubelet/kubeconfig
        - name: kubecerts
            hostPath:
                path: /etc/kubernetes/certs/
```

I also added a node selector to ensure kube-router only gets installed on Linux nodes (it does not support Windows):

```yaml
    nodeSelector:
        beta.kubernetes.io/os: linux
```

After some trial and error I came up with a configuration that would deploy successfully. My deployment file can be found here: [https://github.com/marrobi/kube-router/blob/marrobi/aks-yaml/daemonset/kube-router-firewall-daemonset-aks.yaml](https://github.com/marrobi/kube-router/blob/marrobi/aks-yaml/daemonset/kube-router-firewall-daemonset-aks.yaml)

This can be installed on to your AKS cluster using the following command:

```bash
kubectl apply -f https://raw.githubusercontent.com/marrobi/kube-router/marrobi/aks-yaml/daemonset/kube-router-firewall-daemonset-aks.yaml
```

The status of the kube router daemonset can be seen by running:

```bash
kubectl get daemonset kube-router -n kube-system
```

Once all pods in the daemonset are running:

```bash
NAME          DESIRED   CURRENT   READY     UP-TO-DATE   AVAILABLE   NODE-SELECTOR                 AGE
kube-router   3         3         3         3            3           beta.kubernetes.io/os=linux   4m
```

I verified functionality using examples in Ahmet Alp Balkan's [Network Policy recipes repository](https://github.com/ahmetb/kubernetes-network-policy-recipes).

The samples, covering both ingress and egress policies, all performed as expected.

I’d be interested in feedback from others with regards to and pros and cons of using kube-router to enforce network policies on AKS.