---
title: Securing NGINX-ingress
description: 'cert-manager tutorials: Using ingress-nginx to solve an ACME HTTP-01 challenge'
---

This tutorial will detail how to install and secure ingress to your cluster
using NGINX.

## Prerequisites

Before you begin, ensure you have the following: 

1. A running Kubernetes Cluster
    - You can use any Kubernetes setup, such as a managed service (e.g., Amazon EKS, Google GKE) or a local cluster (e.g., Minikube, MicroK8s).
    - Ensure the cluster is functional and accessible using `kubectl.`
  
3. Kubectl installed and configured
    - The `kubectl` CLI should be installed and set up to connect to your Kubernetes cluster.
    - Verify connectivity using:
    ```bash
    kubectl get nodes
    ```
    You should see a list of nodes in your cluster.
   
3. A domain name
   
   In this tutorial, we will demonstrate how to expose your service using an Ingress resource, which typically requires a domain name to route traffic to your service.  
    - For production environments: You will need to have a domain name that you can map to your Ingress resources (e.g.,`www.cert-manager.io`), ensuring that external traffic can reach your service securely. 
   - For this tutorial: We’ll use a placeholder domain, such as `example.com,` to simplify the setup. You can replace it with your own domain name in a real-world scenario

## Step 1 - Install Helm

> *Skip this section if you have helm installed.*

The easiest way to install `cert-manager` is to use [`Helm`](https://helm.sh), a Kubernetes package manager that simplifies resource deployment.

To install Helm, follow the [official Helm installation guide](https://helm.sh/docs/intro/install/) or use one of the quick installation commands below.

For example, on MacOS:

```bash
brew install kubernetes-helm
```
After installing, verify that Helm is accessible:

```bash
helm version
```
if you're new to Helm, consult the official documentation for additional details.

## Step 2 - Deploy an Example Web Service

To demonstrate how to secure your services with `ingress` and `cert-manager`, we will deploy a simple NGINX web server called `hello-app`. This server would initially respond to HTTP requests with "hello world!" and will later be configured to handle HTTPS traffic securely using TLS certificates from cert-manager.

Before applying the manifest, ensure you understand its components. The deployment defines a pod running the web server, while the service exposes it within the cluster.

### Deployment Manifest

The following manifest deploys the web server:

```yaml file=./example/hello-app-deployment.yaml
```

### Apply the Deployment
Use the following command to deploy the web server:
```bash
kubectl apply -f deployment.yaml
```
This command will create the deployment resource for the web server. After running it, verify that the pod is running successfully by executing:
```bash
kubectl get pods
```

Once the web server is running, we need to expose it within the cluster. The service resource ensures that the server can be accessed on a stable network endpoint within the Kubernetes environment.

```yaml file=./example/hello-app-service.yaml
```
### Apply the Service
Use the command to deploy the service:
```bash
kubectl apply -f service.yaml
```
Once the service deployment is successful, you can check that the service has been created and is exposing the web server correctly by running:
```bash
kubectl get svc hello-app
```
This command will display information about the service, including the assigned ClusterIP and port, confirming that the service is ready to route traffic to the web server pod.

You can create, download and reference these files locally, or you can
reference them from the GitHub source repository for this documentation.
To install the example service from the tutorial files straight from GitHub, do
the following:

```bash
kubectl apply -f https://raw.githubusercontent.com/cert-manager/website/master/content/docs/tutorials/acme/example/hello-app-deployment.yaml
# expected output: deployment.extensions "hello-app" created

kubectl apply -f https://raw.githubusercontent.com/cert-manager/website/master/content/docs/tutorials/acme/example/hello-app-service.yaml
# expected output: service "hello-app" created
```

So far, we’ve deployed a web server and ensured it is accessible within the Kubernetes cluster through a service. However, to make the application accessible from outside the cluster, we need to expose it using an `Ingress Resource`.  

The Ingress acts as a gateway that manages external HTTP and HTTPS traffic and directs it to the appropriate services within your cluster.  

To use an ingress resource, we need an `Ingress Controller`. This tutorial will guide you through deploying the NGINX Ingress Controller and configuring an Ingress resource to expose your service.


## Step 3 - Deploy the NGINX Ingress Controller and Ingress Resource

To expose the hello-app externally, we need an Ingress Controller and an Ingress resource.

A [`kubernetes Ingress Controller`](https://kubernetes.io/docs/concepts/services-networking/ingress) acts as an access point for HTTP and HTTPS traffic, directing it to the services running within your Kubernetes cluster. In this tutorial, we’ll use the NGINX Ingress Controller to manage traffic routing. This controller is deployed as a Kubernetes service, handling HTTP(S) requests and forwarding them to the appropriate backend services.

You can learn more about `NGINX Ingress` from the [official documentation](https://kubernetes.github.io/ingress-nginx/).

### 3a Install the Nginx Ingress Controller

First, add the Helm repository for the NGINX Ingress Controller:
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
```

Update the helm repository with the latest charts:
```bash
$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Skip local chart repository
...Successfully got an update from the "ingress-nginx" chart repository
Update Complete. ⎈ Happy Helming!⎈
```
Install the NGINX Ingress controller using Helm:

```bash
$ helm install nginx-ingress ingress-nginx/ingress-nginx

NAME: nginx-ingress
... lots of output ...
```

Once the NGINX Ingress Controller is deployed, it can take a minute or two for the cluster to configure the service and provide an external IP address (if using a cloud provider's LoadBalancer). For on-premise or local Kubernetes environments like MicroK8s, you will need to check the ClusterIP or manually expose the service.

You can verify the status of the ingress controller service with the following command:
```bash
kubectl get svc
```
This will output a table similar to the following:
```bash
NAME                                               TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)                      AGE
hello-app                                          ClusterIP      10.152.183.154   <none>           80/TCP                       30m
kubernetes                                         ClusterIP      10.152.183.1     <none>           443/TCP                      60m
nginx-ingress-ingress-nginx-controller             LoadBalancer   10.152.183.166   <pending>        80:32482/TCP,443:32483/TCP   14m
nginx-ingress-ingress-nginx-controller-admission   ClusterIP      10.152.183.240   <none>           443/TCP                      14m
```

This command shows you all the services in your cluster (in the `default`
namespace), and any external IP addresses they have. When you first create the
controller, your cloud provider won't have assigned and allocated an IP address
through the `LoadBalancer` yet. Until it does, the external IP address for the
service will be listed as `<pending>`.

Your cloud provider may have options for reserving an IP address prior to
creating the ingress controller and using that IP address rather than assigning
an IP address from a pool. Read through the documentation from your cloud
provider on how to arrange that.

### 3b Configure an Ingress Resource for the hello-app
With the Ingress Controller in place, we can create an Ingress resource to expose the hello-app externally. In this step, you will modify the example manifest to reflect the domain you own or control. This will ensure that your service is accessible from the outside world.

Here’s an example Ingress manifest you can use as a starting point:

```yaml file=./example/hello-app-ingress.yaml
```
You can download the sample manifest from GitHub , edit it, and submit the manifest to Kubernetes with the command below. Once you have edited the file in your editor, save the changes, and the command will automatically apply the updated file:
```bash
kubectl create --edit -f https://raw.githubusercontent.com/cert-manager/website/master/content/docs/tutorials/acme/example/hello-app-ingress.yaml
# expected output: ingress.networking.k8s.io/hello-app-ingress created
```
Alternatively, copy the manifest file above, save it as `ingress.yaml`, and apply it directly:
```bash
kubectl apply -f ingress.yaml
```

> Note: The ingress example we show above has a `host` definition within it. The
> `ingress-nginx-controller` will route traffic when the hostname requested
> matches the definition in the ingress. You *can* deploy an ingress without a
> `host` definition in the rule, but that pattern isn't usable with a TLS
> certificate, which expects a fully qualified domain name.

Once it is deployed, you can use the command `kubectl get ingress` to see the status
 of the ingress:

```text
NAME                CLASS   HOSTS                 ADDRESS          PORTS   AGE
hello-app-ingress   nginx   example.example.com                    80      20s
```

It may take a few minutes, depending on your service provider, for the ingress
to be fully created. When it has been created and linked into place, the
ingress will show an address as well:

```text
NAME                CLASS   HOSTS                 ADDRESS          PORTS   AGE
hello-app-ingress   nginx   example.example.com   192.168.64.100   80      20s
```

> Note: The IP address on the ingress *may not* match the IP address that the
> `ingress-nginx-controller` has. This is fine, and is a quirk/implementation detail
> of the service provider hosting your Kubernetes cluster. Since we are using
> the `ingress-nginx-controller` instead of any cloud-provider specific ingress
> backend, use the IP address that was defined and allocated for the
> `nginx-ingress-ingress-nginx-controller ` `LoadBalancer` resource as the primary  access point for
> your service.

Make sure the service is reachable at the domain name you added above, for
example `http://example.example.com`. While the DNS configuration is not yet in place, you can check if your ingress is correctly routing traffic using a command line tool like curl.

Here’s how you can check the status of your ingress:
```bash
curl -kivL -H 'Host: example.example.com' 'http://192.168.64.100'
```
The options in this curl command will:
 - Provide verbose output (-v), so you can see detailed request and response headers.
 - Follow redirects (-L) if necessary.
 - Display TLS headers (-i) and not error on insecure certificates (-k).
    
If the ingress is functioning correctly, you should see a response similar to this:
```bash
*   Trying 192.168.64.100:80...
* Connected to 192.168.64.100 (192.168.64.100) port 80
> GET / HTTP/1.1
> Host: example.example.com
> User-Agent: curl/8.5.0
> Accept: */*
> 
< HTTP/1.1 200 OK
HTTP/1.1 200 OK
< Date: Mon, 25 Nov 2024 14:02:15 GMT
Date: Mon, 25 Nov 2024 14:02:15 GMT
< Content-Type: text/plain; charset=utf-8
Content-Type: text/plain; charset=utf-8
< Content-Length: 66
Content-Length: 66
< Connection: keep-alive
Connection: keep-alive

< 
Hello, world!
Version: 1.0.0
Hostname: hello-app-5fdb69cd94-lxtv4
* Connection #0 to host 192.168.64.100 left intact
```
### What to Look for:

 - Status Code: You should see `HTTP/1.1 200 OK,` indicating that the ingress is correctly routing traffic to your application.
 - Response Content: You should see the expected content, such as `Hello, world!` and details about the app version and hostname, confirming that the application is running properly.
 - IP Address: The IP address here is the external IP of the ingress, which may differ from the Ingress controller’s internal IP. Ensure you're using the correct IP from the `LoadBalancer` service.
   
If curl confirms that the ingress is routing traffic correctly, the next step is to set up DNS to route traffic to the correct address in a user-friendly way, i.e., via a browser.

## Step 4 - Configure DNS

The external IP allocated to the ingress controller is where all incoming traffic should be routed. To make this work, you'll need to add this IP address to a DNS zone that you control. For example, you could map it to a domain like `example.example.com.` (Note: this is an example domain used for illustration. You will need to use your own domain name here).

This tutorial assumes you're familiar with assigning a DNS entry to an IP address. If you're unsure, you can check the documentation provided by your DNS hosting provider for more detailed instructions on how to set up the DNS record.

Once the DNS record is set up and propagated, you can access your `hello-app` application via your web browser using your domain name. For example, after completing the DNS setup, you would visit:
`http://example.example.com` (again, replace this with your actual domain).

At this point, your application is accessible via `HTTP.` However, it is recommended that you use `HTTPS` to ensure secure communication between your users and your application.
Next, we will configure `TLS certificates` using `cert-manager` and `Let's Encrypt,` allowing us to serve the application securely over HTTPS.

## Step 5 - Deploy cert-manager

To manage TLS certificates in Kubernetes, cert-manager is required. It automates the process of requesting certificates and responding to validation challenges. You can install cert-manager using Helm or plain Kubernetes manifests.

Since we installed Helm earlier, we'll proceed with installing cert-manager using Helm; follow the steps in this
[Helm guide](../../installation/helm.md). For other methods, refer to the [cert-manager installation documentation](../../installation/README.md) .

### Custom Resources Used by cert-manager

cert-manager mainly uses two different custom Kubernetes resources - known as
[`CRDs`](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) -
to configure and control certificate issuance and operations: Issuers and Certificates. 

### Issuers

Issuers define how cert-manager requests TLS certificates. They can either be namespace-scoped(`Issuers`) or cluster-scoped(`ClusterIssuer`).

Here are the key points to understand:

 - Issuer: Scoped to a single namespace. Certificates and Issuers must reside in the same namespace.
 - ClusterIssuer: Accessible across all namespaces in the cluster. Useful for centralized management, especially when certificates are required for resources in multiple namespaces.

Since we will deploy cert-manager resources in a separate namespace, we'll use a ClusterIssuer. If you decide to use ClusterIssuer, update the annotations in your Ingress manifests accordingly:
```bash
annotations:
  cert-manager.io/cluster-issuer: <name-of-cluster-issuer>
```
Alternatively, if you prefer to scope certificate issuance to specific namespaces, you can use an `Issuer.` The `Issuer` must be created in the same namespace as the `Certificate` and potentially other related resources like the Ingress. Since these resources are usually in the `default` namespace or a namespace of your choice, ensure that both the `Issuer` and `Certificate` are located in that namespace.
To reference the `Issuer` in your Ingress, use the annotation `cert-manager.io/issuer.` For example:
```bash
annotations:
  cert-manager.io/issuer: my-issuer
```

For troubleshooting or guidance on how Issuers work, see the [Issuer Troubleshooting Guide](../../troubleshooting/acme.md) guide.

More information on the differences between `Issuers` and `ClusterIssuers` - including
when you might choose to use each can be found on [Issuer concepts](../../concepts/issuer.md#namespaces).

### Certificates

Certificates specify the details of the TLS certificate you want to request. These resources reference an Issuer (or ClusterIssuer) to manage certificate issuance. 
For example:
 - The spec.issuerRef field in a Certificate resource determines whether to use an Issuer or ClusterIssuer.

For more information, see [Certificate Concepts](../../usage/certificate.md).

## Step 6 - Configure a Let's Encrypt ClusterIssuer

We'll set up two `ClusterIssuers` for Let's Encrypt in this example: staging and production.

The Let's Encrypt production issuer has [very strict rate limits](https://letsencrypt.org/docs/rate-limits/).
When you're experimenting and learning, it's easy to hit those limits. Because of that risk,
we'll start with the Let's Encrypt staging issuer. Once we're happy it's working,
we'll switch to the production issuer.

> Note: You may see a warning about untrusted certificates from the staging issuer, but this is expected.

### Define the ClusterIssuer
   
Create this definition locally and update the email address to your own. This
email is required by Let's Encrypt and used to notify you of certificate
expiration and updates.

```yaml file=./example/hello-staging-issuer.yaml
```

Once edited, apply the custom resource:

```bash
kubectl create --edit -f https://raw.githubusercontent.com/cert-manager/website/master/content/docs/tutorials/acme/example/hello-staging-issuer.yaml
# expected output: clusterissuer.cert-manager.io/letsencrypt-staging created
```

Similarly, define and deploy the production ClusterIssuer. Be sure to update the email address:

```yaml file=./example/hello-prod-issuer.yaml
```

```bash
kubectl create --edit -f https://raw.githubusercontent.com/cert-manager/website/master/content/docs/tutorials/acme/example/hello-prod-issuer.yaml
# expected output: clusterissuer.cert-manager.io/letsencrypt-prod created
```
Both `ClusterIssuers` are configured to use the [`HTTP01`](../../configuration/acme/http01/README.md) challenge provider.


### Verify the ClusterIssuer

After creating the ClusterIssuer, check its status with the following command:
```bash
kubectl describe clusterissuer letsencrypt-staging
```

```bash
Name:         letsencrypt-staging
Namespace:    
Labels:       <none>
Annotations:  <none>
API Version:  cert-manager.io/v1
Kind:         ClusterIssuer
Metadata:
  Creation Timestamp:  2024-11-27T19:03:38Z
  Generation:          1
  Resource Version:    950735
  UID:                 2dd95072-532a-4b6f-b9c6-cf63072bef95
Spec:
  Acme:
    Email:  testemail@gmail.com
    Private Key Secret Ref:
      Name:  letsencrypt-staging
    Server:  https://acme-staging-v02.api.letsencrypt.org/directory
    Solvers:
      http01:
        Ingress:
          Ingress Class Name:  nginx
Status:
  Acme:
    Last Private Key Hash:  mquANXPyfarpDtTfcCq4gOIbM7X3NtCPXkTqT0Ht8Us=
    Last Registered Email:  testemail@gmail.com
    Uri:                    https://acme-staging-v02.api.letsencrypt.org/acme/acct/173498384
  Conditions:
    Last Transition Time:  2024-11-27T19:03:40Z
    Message:               The ACME account was registered with the ACME server
    Observed Generation:   1
    Reason:                ACMEAccountRegistered
    Status:                True
    Type:                  Ready
Events:                    <none>
```
You should see a `True` status under the `ACMEAccountRegistered` condition, indicating that the ClusterIssuer has successfully registered with the ACME server and is ready to issue certificates.

## Step 7 - Deploy a TLS Ingress Resource

Now that the required configurations are in place, we can proceed to request a TLS certificate for our application.  There are two primary ways to do this: using
annotations on the ingress with [`ingress-shim`](../../usage/ingress.md) or directly creating a certificate resource.

In this example, we will add annotations to the ingress, and take advantage
of ingress-shim to have it create the certificate resource on our behalf.
After creating a certificate, cert-manager will update or create an ingress
resource and use that to validate the domain. Once verified and issued,
cert-manager will create or update the secret defined in the certificate.

> Note: The secret that is used in the ingress should match the secret defined
> in the certificate.  There isn't any explicit checking, so a typo will result
> in the `ingress-nginx-controller` falling back to its self-signed certificate.
> In our example, we are using annotations on the ingress (and ingress-shim)
> which will create the correct secrets on your behalf.

### Deploy Staging Ingress

Edit the ingress by adding the necessary annotations and TLS configurations to your ingress for staging. 
For example:
```yaml file=./example/hello-ingress-staging-tls.yaml
```

and apply it:
```bash
kubectl apply -f ingress.yaml
```


Cert-manager will process the annotations in your ingress resource to create a certificate. After applying the configuration, you can verify the status using the following command:
```bash
kubectl get certificate
```

For example, the output might look like this:
```bash
NAME              READY   SECRET            AGE
staging-app-tls   True    staging-app-tls   4m56s
```

> Note: It may take some time for the certificate to be issued and for the READY status to change to True. This delay depends on factors like DNS propagation and the ACME challenge processing by cert-> 
> manager. If the status does not update immediately, wait a few minutes and check again. You can also monitor the process by describing the certificate resource:

```bash
$ kubectl describe certificate staging-app-tls
Name:         staging-app-tls
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  cert-manager.io/v1
Kind:         Certificate
Metadata:
  Creation Timestamp:  2024-12-02T18:32:22Z
  Generation:          1
  Owner References:
    API Version:           networking.k8s.io/v1
    Block Owner Deletion:  true
    Controller:            true
    Kind:                  Ingress
    Name:                  hello-app-ingress
    UID:                   3693c9c6-6b86-452e-ac44-09aca0472a81
  Resource Version:        10477
  UID:                     eea055e8-48b4-4b2a-af41-bcf4f2eb86d6
Spec:
  Dns Names:
    example.example.com
  Issuer Ref:
    Group:      cert-manager.io
    Kind:       ClusterIssuer
    Name:       letsencrypt-staging
  Secret Name:  staging-app-tls
  Usages:
    digital signature
    key encipherment
Status:
  Conditions:
    Last Transition Time:  2024-12-02T18:32:57Z
    Message:               Certificate is up to date and has not expired
    Observed Generation:   1
    Reason:                Ready
    Status:                True
    Type:                  Ready
  Not After:               2025-03-02T17:34:22Z
  Not Before:              2024-12-02T17:34:23Z
  Renewal Time:            2025-01-31T17:34:22Z
  Revision:                1
Events:
  Type    Reason     Age    From                                       Message
  ----    ------     ----   ----                                       -------
  Normal  Issuing    6m32s  cert-manager-certificates-trigger          Issuing certificate as Secret does not exist
  Normal  Generated  6m32s  cert-manager-certificates-key-manager      Stored new private key in temporary Secret resource "staging-app-tls-xw4j4"
  Normal  Requested  6m32s  cert-manager-certificates-request-manager  Created new CertificateRequest resource "staging-app-tls-1"
  Normal  Issuing    5m57s  cert-manager-certificates-issuing          The certificate has been successfully issued
```

The events associated with this resource and listed at the bottom
of the `describe` results show the state of the request. In the above
example the certificate was validated and issued within a couple of minutes.

Once complete, cert-manager will have created a secret with the details of
the certificate based on the secret used in the ingress resource. You can
use the describe command as well to see some details:

```bash
$ kubectl describe secret staging-app-tls
Name:         staging-app-tls
Namespace:    default
Labels:       controller.cert-manager.io/fao=true
Annotations:  cert-manager.io/alt-names: example.example.com
              cert-manager.io/certificate-name: staging-app-tls
              cert-manager.io/common-name: example.example.com
              cert-manager.io/ip-sans: 
              cert-manager.io/issuer-group: cert-manager.io
              cert-manager.io/issuer-kind: ClusterIssuer
              cert-manager.io/issuer-name: letsencrypt-staging
              cert-manager.io/uri-sans: 

Type:  kubernetes.io/tls

Data
====
tls.crt:  3769 bytes
tls.key:  1679 bytes
```

Now that we have confidence that everything is configured correctly, you
can update the annotations and TLS configuration in the ingress to specify the production issuer:

```yaml file=./example/hello-ingress-prod-tls.yaml
```
Apply the updated configuration
```bash
kubectl apply -f ingress.yaml
```

Since we are using distinct secret names for staging (staging-app-tls) and production (prod-app-tls), there is no need to delete the existing secret. Cert-manager will create a new secret for the production configuration.

This will start the process to get a new certificate, and using describe
you can see the status. Once the production certificate has been updated,
you should see the example hello app running at your domain with a signed TLS
certificate.

```bash
$ kubectl describe certificate prod-hello-app-tls
Name:         prod-hello-app-tls
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  cert-manager.io/v1
Kind:         Certificate
Metadata:
  Creation Timestamp:  2024-12-02T18:43:28Z
  Generation:          1
  Owner References:
    API Version:           networking.k8s.io/v1
    Block Owner Deletion:  true
    Controller:            true
    Kind:                  Ingress
    Name:                  hello-app-ingress
    UID:                   3693c9c6-6b86-452e-ac44-09aca0472a81
  Resource Version:        12475
  UID:                     c886a3d1-cf7d-4d58-8af8-478ec5ff27bd
Spec:
  Dns Names:
    example.example.com
  Issuer Ref:
    Group:      cert-manager.io
    Kind:       ClusterIssuer
    Name:       letsencrypt-prod
  Secret Name:  prod-hello-app-tls
  Usages:
    digital signature
    key encipherment
Status:
  Conditions:
    Last Transition Time:  2024-12-02T18:44:03Z
    Message:               Certificate is up to date and has not expired
    Observed Generation:   1
    Reason:                Ready
    Status:                True
    Type:                  Ready
  Not After:               2025-03-02T17:45:29Z
  Not Before:              2024-12-02T17:45:30Z
  Renewal Time:            2025-01-31T17:45:29Z
  Revision:                1
Events:
  Type    Reason     Age   From                                       Message
  ----    ------     ----  ----                                       -------
  Normal  Issuing    67s   cert-manager-certificates-trigger          Issuing certificate as Secret does not exist
  Normal  Generated  66s   cert-manager-certificates-key-manager      Stored new private key in temporary Secret resource "prod-hello-app-tls-824gl"
  Normal  Requested  66s   cert-manager-certificates-request-manager  Created new CertificateRequest resource "prod-hello-app-tls-1"
  Normal  Issuing    32s   cert-manager-certificates-issuing          The certificate has been successfully issued
```

You can see the current state of the ACME Order by running `kubectl describe`
on the Order resource that cert-manager has created for your Certificate:

```bash
$ kubectl describe order prod-hello-app-tls-1-2985368724 
...
Events:
  Type    Reason    Age    From                 Message
  ----    ------    ----   ----                 -------
  Normal  Created   3m42s  cert-manager-orders  Created Challenge resource "prod-hello-app-tls-1-2985368724-1406537188" for domain "example.example.com"
  Normal  Complete  3m9s   cert-manager-orders  Order completed successfully
```
## Conclusion

Congratulations! You’ve successfully configured cert-manager to issue TLS certificates for both staging and production environments using Let's Encrypt. By following this tutorial, you've not only secured your application's ingress but also ensured that traffic to your cluster is encrypted and routed through NGINX with the proper TLS configuration.

With cert-manager automatically managing certificate renewals, you no longer need to worry about the validity of your certificates. This setup guarantees secure and trusted HTTPS connections to your services, enhancing the security of both internal and external traffic.

If you wish to explore more advanced cert-manager features or need help debugging challenges, feel free to explore the official documentation. For additional support, you can also join the community forums or Slack channels.
