# Application routing add-on for AKS training

The application routing add-on with NGINX delivers the following:

* Easy configuration of managed NGINX Ingress controllers based on Kubernetes NGINX Ingress controller.
* Integration with Azure DNS for public and private zone management
* SSL termination with certificates stored in Azure Key Vault.

The add-on runs the [application routing operator](https://github.com/Azure/aks-app-routing-operator).

## What is an ingress?
In Kubernetes, an ingress is an API object that manages external access to services within a cluster. It acts as a traffic controller, routing incoming requests to the appropriate services based on rules defined in the ingress resource. It provides a way to expose HTTP and HTTPS routes to services, enabling external access to applications running in the cluster.

## Create a cluster with the add-on enabled

### Create a resource group

```bash
az group create -n <ResourceGroupName> -l <Location>
```

### Create a cluster

This will create a cluster with the add-on enabled. The default controller that comes with this cluster will have a public IP address.


```bash
az aks create -g <ResourceGroupName> -n <ClusterName> -l <Location> --enable-app-routing
```

### Get the cluster credentials

```bash
az aks get-credentials -g <ResourceGroupName> -n <ClusterName>
```


## Deploy an application

### Create a deployment

Create a file `deployment.yaml` with the following contents.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: aks-helloworld  
spec:
  replicas: 1
  selector:
    matchLabels:
      app: aks-helloworld
  template:
    metadata:
      labels:
        app: aks-helloworld
    spec:
      containers:
      - name: aks-helloworld
        image: mcr.microsoft.com/azuredocs/aks-helloworld:v1
        ports:
        - containerPort: 80
        env:
        - name: TITLE
          value: "Welcome to Azure Kubernetes Service (AKS)"
```          

### Create a service

Create a file `service.yaml` with the following contents.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: aks-helloworld
spec:
  type: ClusterIP
  ports:
  - port: 80
  selector:
    app: aks-helloworld
```          

### Create an ingress

The application routing add-on creates an Ingress class on the cluster named `webapprouting.kubernetes.azure.com`. When you create an Ingress object with this class, it activates the add-on.

Create a file `ingress.yaml` with the following contents.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: aks-helloworld
spec:
  ingressClassName: webapprouting.kubernetes.azure.com
  rules:
  - http:
      paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: frontend
              port:
                number: 80
```          

### Create a namespace and apply the files

```bash
kubectl create ns helloworld
kubectl apply -n helloworld -f deployment.yaml
kubectl apply -n helloworld -f service.yaml
kubectl apply -n helloworld -f ingress.yaml
```

### Retrieve the public IP address of the ingress

Check for a public IP address for the application's ingress. Monitor progress using the `kubectl get ingress` command with the `--watch` argument.

```bash
kubectl get ingress aks-helloworld -n helloworld --watch
```

The **ADDRESS** output for the `aks-helloworld` ingress initially shows empty:

```output
NAME             CLASS                                HOSTS   ADDRESS        PORTS   AGE
aks-helloworld   webapprouting.kubernetes.azure.com   *                      80      12m
```

Once the **ADDRESS** changes from blank to an actual public IP address, use `CTRL-C` to stop the `kubectl` watch process.

The following sample output shows a valid public IP address assigned to the ingress:

```output
NAME              CLASS                                HOSTS   ADDRESS        PORTS   AGE
aks-helloworld    webapprouting.kubernetes.azure.com   *       4.255.22.196   80      12m
```

Open a web browser to the external IP address of your ingress to see the app in action.

## Use a custom domain name hosted as an Azure DNS zone

You can configure the application routing add-on to automatically configure DNS entries on Azure DNS for ingresses that you create.

### Create a public Azure DNS Zone

```bash
az network dns zone create -g <ResourceGroupName> -n <ZoneName>
```

### Attach Azure DNS zone to the application routing add-on

Retrieve the resource ID for the DNS zone using the `az network dns zone show` command and set the output to a variable named `ZONEID`.

```bash
ZONEID=$(az network dns zone show -g <ResourceGroupName> -n <ZoneName> --query "id" --output tsv)
```

Update the add-on to enable the integration with Azure DNS using the `az aks approuting zone` command. You can pass a comma-separated list of DNS zone resource IDs.

```bash
az aks approuting zone add -g <ResourceGroupName> -n <ClusterName> --ids=${ZONEID} --attach-zones
```

### Update the ingress object to use a hostname

Update the file `ingress.yaml` with the following contents. Replace `<ZoneName>` with a name of the zone you created earlier.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: aks-helloworld
spec:
  ingressClassName: webapprouting.kubernetes.azure.com
  rules:
  - http:
      host: helloworld.<ZoneName>
      path: /
        pathType: Prefix
      paths:
      - backend:
          service:
            name: aks-helloworld
            port:
              number: 80
```

### Apply the modified ingress

```bash
kubectl apply -n helloworld -f ingress.yaml
```

### Verify the ingress was changed


```bash
NAME             CLASS                                HOSTS               ADDRESS       PORTS     AGE
aks-helloworld   webapprouting.kubernetes.azure.com   myapp.contoso.com   20.51.92.19   80, 443   4m
```

### Verify that the Azure DNS has been reconfigured with a new `A` record

In a few minutes, the Azure DNS zone will be reconfigured with a new `A` record pointing to the IP address of the NGINX ingress controller.

```bash
az network dns record-set a list -g <ResourceGroupName> -n <ZoneName>
```

The following example output shows the created record:

```output
[
  {
    "aRecords": [
      {
        "ipv4Address": "20.51.92.19"
      }
    ],
    "etag": "188f0ce5-90e3-49e6-a479-9e4053f21965",
    "fqdn": "helloworld.contoso.com.",
    "id": "/subscriptions/xxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxx/resourceGroups/foo/providers/Microsoft.Network/dDnsZones/contoso.com/A/helloworld",
    "isAutoRegistered": false,
    "name": "helloworld",
    "resourceGroup": "foo",
    "ttl": 300,
    "type": "Microsoft.Network/dnsZones/A"
  }
]
```

If this Azure DNS zone is configured to resolve to an actual public domain that you purchased and you've configured its top level nameservers, you should be able to use your browser to access the hostname.

## Terminate HTTPS traffic with certificates from Azure Key Vault

The application routing add-on can be integrated with Azure Key Vault to retrieve SSL certificates to use with ingresses.

### Create an Azure Key Vault to store the certificate

```bash
az keyvault create -g <ResourceGroupName> -l <Location> -n <KeyVaultName> --enable-rbac-authorization true
```

### Create and export a self-signed SSL certificate

For testing, you can use a self-signed public certificate instead of a Certificate Authority (CA)-signed certificate. Create a self-signed SSL certificate to use with the Ingress using the `openssl req` command. Make sure you replace <Hostname> with the DNS name you're using.

```bash
openssl req -new -x509 -nodes -out aks-ingress-tls.crt -keyout aks-ingress-tls.key -subj "/CN=<Hostname>" -addext "subjectAltName=DNS:<Hostname>"
```

Export the SSL certificate and skip the password prompt using the `openssl pkcs12 -export` command.

```bash
openssl pkcs12 -export -in aks-ingress-tls.crt -inkey aks-ingress-tls.key -out aks-ingress-tls.pfx
```

### Import certificate into Azure Key Vault

Import the SSL certificate into Azure Key Vault using the `az keyvault certificate import` command. If your certificate is password protected, you can pass the password through the `--password` flag.

```bash
az keyvault certificate import --vault-name <KeyVaultName> -n <KeyVaultCertificateName> -f aks-ingress-tls.pfx [--password <certificate password if specified>]
```

### Enable Azure Key Vault integration

Retrieve the Azure Key Vault resource ID.

```bash
KEYVAULTID=$(az keyvault show --name <KeyVaultName> --query "id" --output tsv)
```

Then update the app routing add-on to enable the Azure Key Vault secret store CSI driver and apply the role assignment.

```bash
az aks approuting update -g <ResourceGroupName> -n <ClusterName> --enable-kv --attach-kv ${KEYVAULTID}
```

### Retrieve the certificate URI from Azure Key Vault

Get the certificate URI to use in the Ingress from Azure Key Vault using the `az keyvault certificate show` command.

```bash
az keyvault certificate show --vault-name <KeyVaultName> -n <KeyVaultCertificateName> --query "id" --output tsv
```

The following example output shows the certificate URI returned from the command:

```output
https://KeyVaultName.vault.azure.net/certificates/KeyVaultCertificateName/ea62e42260f04f17a9309d6b87aceb44
```

### Update the ingress object to use a hostname

Update the file `ingress.yaml` with the following contents. Replace `<ZoneName>` with a name of the zone you created earlier and `<KeyVaultCertificateUri>` with the certificate URI. You can ommit the version of the certificate and just use that portion `https://KeyVaultName.vault.azure.net/certificates/KeyVaultCertificateName`.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: aks-helloworld
  annotations:
    kubernetes.azure.com/tls-cert-keyvault-uri: <KeyVaultCertificateUri>
spec:
  ingressClassName: webapprouting.kubernetes.azure.com
  rules:
  - http:
      host: helloworld.<ZoneName>
      path: /
        pathType: Prefix
      paths:
      - backend:
          service:
            name: aks-helloworld
            port:
              number: 80
  tls:
  - hosts:
    - helloworld.<ZoneName>
    secretName: keyvault-aks-helloworld
```

### Apply the modified ingress

```bash
kubectl apply -n helloworld -f ingress.yaml
```

## Understand the `NginxIngressController` custom resource

### List the NGINX ingress controllers in the cluster

```bash
kubectl get nginxingresscontroller -n app-routing-system
```

```output
NAME      INGRESSCLASS                         CONTROLLERNAMEPREFIX   AVAILABLE
default   webapprouting.kubernetes.azure.com   nginx                  True
```

### Review the default controller

```bash
 kubectl get nginxingresscontroller default -n app-routing-system -o yaml
 ```

The default controller has an ingress class name of `webapprouting.kubernetes.azure.com` and its lifecycle is controlled with the `NginxIngressController` custom resource.

 ```yaml
 apiVersion: approuting.kubernetes.azure.com/v1alpha1
kind: NginxIngressController
metadata:
  creationTimestamp: "2024-04-11T19:17:40Z"
  generation: 1
  name: default
  resourceVersion: "483521"
  uid: 515626ac-1c45-498b-bdfb-ece88a5cccd6
spec:
  controllerNamePrefix: nginx
  ingressClassName: webapprouting.kubernetes.azure.com
 ```

The app routing operator uses the `NginxIngressController` custom resource to create the actual ingress controller resources in the `app-routing-system` namespace.

You can view the created ingress controller resources as follows.

```bash
kubectl get deployment,service,hpa nginx -n app-routing-system
```

```output
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx   2/2     2            2           22h

NAME            TYPE           CLUSTER-IP    EXTERNAL-IP    PORT(S)                                      AGE
service/nginx   LoadBalancer   10.0.62.104   4.255.88.186   80:30700/TCP,443:31178/TCP,10254:31788/TCP   22h

NAME                                        REFERENCE          TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/nginx   Deployment/nginx   0%/80%    2         100       2          22h
```

## Try additional configuration scenarios

You can create additional NGINX ingress controllers with different properties, like private IP addresses or static IP addresses by creating additional `NginxIngressController` custom resources Follow the documentation for additional scenarios.

 - [Configure Azure private DNS zone support with a private NGINX ingress controller](https://learn.microsoft.com/en-us/azure/aks/create-nginx-ingress-private-controller).
 - [Configure a static IP address for an ingress controller](https://learn.microsoft.com/en-us/azure/aks/app-routing-nginx-configuration#create-an-nginx-ingress-controller-with-a-static-ip-address)
 - [Configure ingress annotations for custom max body size, connection timeout, backend protocol, CORS, SSL Rediret, and URL rewriting](https://learn.microsoft.com/en-us/azure/aks/app-routing-nginx-configuration#configuration-per-ingress-resource-through-annotations)
 - [Monitor the ingress-nginx controller metrics in the application routing add-on with Prometheus in Grafana](https://learn.microsoft.com/en-us/azure/aks/app-routing-nginx-prometheus)


## Coming soon in an upcoming AKS weekly release

The following features are part of the [0.2.2  release of the app routing operator](https://github.com/Azure/aks-app-routing-operator/releases/tag/v0.2.2) and will be available in an upcoming AKS weekly release.

### Configuring NGINX ingress controller minimum and maximum replicas

Setting
`scaling.minReplicas` and `scaling.maxReplicas` on the `NginxIngressController` custom resource to override lower and upper limits of scaling the NGINX ingress controller deployment.

### Configuring NGINX ingress controller scale threshold

Setting
`threshold` on the `NginxIngressController` custom resource to control how aggressively the horizontal pod autoscaler (HPA) scales the NGINX ingress controller deployment. `Rapid` means the deployment will scale quickly and aggressively, which is the best choice for handling sudden and significant traffic spikes. `Steady` is the opposite, prioritizing cost-effectiveness, which is the best choice when fewer replicas handling more work is desired or when traffic isn't expected to fluctuate. `Balanced` is a good mix between the two that works for most use-cases, and is the default.

### Overriding the default SSL certificate

Setting
`defaultSSLCertificate.secret.name` and `defaultSSLCertificate.secret.namespace` or `defaultSSLCertificate.keyVaultURI` on the `NginxIngressController` custom resource to override the default certificate used across all ingresses for that ingress class.

### Automatic TLS secret reconciler for Azure Key Vault certificates

For ingresses that are annotated with `kubernetes.azure.com/tls-cert-keyvault-managed: true` and `kubernetes.azure.com/tls-cert-keyvault-uri: <uri>`, the operator will automatically update the ingress object to inject the name of the Kubernetes secret that holds the certificate synced from Azure Key Vault.

