# Configuring Kubernetes ingress with a wildcard DNS certificate, single TLS secret and applications in multiple namespaces

## Scenario

An organisation wanted to deploy each application into a  separate Kubernetes namespace. Each application will be available at a subdomain of example.com, via a wildcard DNS entry of \*.example.com pointing to the ingress controller's service IP address.  A single wildcard TLS certificate ( \*.example.com ) will be used to protect all applications using the ingress controller. It was desired that only a single TLS secret should exist on the cluster to facilitate certificate renewal.

## Challenges

1. Kubernetes secrets are only accessible from the namespace in which they are created.

We discussed having a single namespace with all ingress resources and the secret, however still keep each application's services and deployments in separate namespaces.

2. nginx Ingress resources can only route to backend services within their own namespace.

Looked at [http://voyager.appscode.com/](http://voyager.appscode.com/) which can route to other namespaces, however this creates a new IP address for each application, which is not compatible with using a wildcard DNS name.

Another potential solution was was to use an service of type ExternalName to route traffic to the other namespace. Here's how this worked:

## Deployment

### Install ingress controller into an ingress namespace


```bash
helm install --namespace ingress --name ingress stable/nginx-ingress --set rbac.create=false --set rbac.createRole=false --set rbac.createClusterRole=false
```

### Create TLS secret in ingress Namespace

As this was not a live deployment we created a self signed wildcard certificate. In production a certificate would  be acquired from a trusted certification authority:

```bash
openssl req -new -x509 -keyout wildcard.example.com.key -out wildcard.example.com.pem -days 365 -nodes -key wildcard.example.com.key
```

Then create a secret using that certificate:

```bash
kubectl create secret tls wildcard-example-com --key  wildcard.example.com.key --cert wildcard.example.com.pem  -n ingress
```

### Create Helm Chart

Run the following command to create a new Helm chart in a directory named sample-app:

```bash
helm create sample-app
```

### Add External Service

Added a ```service-ingress.yaml``` file to the Helm chart templates directory to create a service of type ExternalName that deploys into the ingress namespace to the Helm chart:

```yaml

{{- $fullName := include "sampleapp.fullname" . -}}
apiVersion: v1
kind: Service
metadata:
  name: {{ $fullName }}
  namespace: ingress
spec:
  ports:
  - name: http
    port: 80
  type: ExternalName
  externalName: {{ $fullName }}.{{ .Release.Namespace }}

```

### Configure Ingress resource to deploy into the Ingress namespace

Specified a namespace to deploy the Ingress resource to by modifying the ```ingress.yaml``` file

```yaml
metadata:
  name: {{ $fullName }}
  namespace:  ingress

```

### Update the helm chart values.yaml

Specify image:

```yaml

image:
  repository: marrobi/linuxwebsite
  tag: latest
  pullPolicy: Always

```

Specify TLS settings:

```yaml

  tls:
    - secretName:  wildcard-example-com
      hosts:
        - sample-app.example.com

```

### Deploy sample application

Deploy application into a new namespace called **sample**:

```bash
helm upgrade --install sample ./sample-app/ --namespace sample
```

### Create DNS name

Retrieve the ingress controller's IP address:

```bash
kubectl get service -n ingress
```

Record the EXTERNAL-IP of the ingress controller:

```bash
NAME                                    CLUSTER-IP     EXTERNAL-IP               PORT(S)                      AGE
ingress-nginx-ingress-controller        10.0.57.175    51.140.163.143            80:32164/TCP,443:32638/TCP   11m
ingress-nginx-ingress-default-backend   10.0.254.194   <none>                    80/TCP                       11m
sample-sampleapp                                       sample-sampleapp.sample   80/TCP                       32s
```

For testing purposes I added a entry to the hosts file on my local machine, in the real world this would be an A record with your public DNS provider:

```bash
51.140.163.143  sample-app.example.com
```

### Test the solution

Navigate to [https://sample-app.example.com](https://sample-app.example.com) in a browser:

<img src="https://github.com/marrobi/blog-posts/raw/master/k8s-wildcard-ingress/success.png" width="500" />

This solution works, however there may be other, "better" solutions. Do let me know if you a have an alternative solution.