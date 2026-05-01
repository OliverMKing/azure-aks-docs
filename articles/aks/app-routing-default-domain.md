---
title: Azure Kubernetes Service (AKS) application routing add-on with a managed default domain and automatic TLS (preview)
description: Use the application routing add-on to expose Ingress traffic on Azure Kubernetes Service (AKS) over a managed default domain with automatic TLS certificate provisioning.
ms.subservice: aks-networking
ms.custom: devx-track-azurecli, biannual
author: kingoliver
ms.topic: how-to
ms.date: 04/30/2026
ms.author: kingoliver 
# Customer intent: As a cloud engineer, I want to expose workloads on Azure Kubernetes Service over a managed default domain with automatic TLS, so that I can publish HTTPS endpoints without owning my own DNS zone or certificate.
---

# Configure ingress with a managed default domain via the application routing add-on (preview)

[!INCLUDE [preview features callout](~/reusable-content/ce-skilling/azure/includes/aks/includes/preview/preview-callout.md)]

The application routing add-on can provision a managed *default domain* for your cluster. When the default domain is enabled, AKS assigns the cluster a unique public DNS-resolvable subdomain that you can use as the host name on `Ingress` resources, and the add-on automatically issues and rotates a TLS certificate for that domain. This lets you publish HTTPS endpoints from your cluster without owning a custom DNS zone or managing certificates yourself.

The default domain works with both forms of the application routing add-on:

* The [Application Routing Gateway API implementation][app-routing-gateway-api] (`gatewayClassName: approuting-istio`), used with Kubernetes `Gateway` and `HTTPRoute` resources.
* The [Application Routing Managed NGINX Ingress][app-routing-nginx], used with Kubernetes `Ingress` resources ([this is actively being retired and is not recommended](https://blog.aks.azure.com/2025/11/13/ingress-nginx-update)).

In both cases, AKS provisions the DNS record for the default domain and the add-on issues the TLS certificate — you only choose which ingress surface (Ingress or Gateway API) to attach it to.

The default domain differs from [bring-your-own DNS zone and TLS][app-routing-dns-tls] in the following ways:

| Feature | Default domain | Bring-your-own DNS zone and TLS |
|---------|----------------|---------------------------------|
| DNS zone | Managed by AKS, generated per cluster | You attach an Azure DNS zone you own |
| Hostname | `*.<generated>.<region>.aksapp.io` style subdomain assigned to your cluster | Any name in your DNS zone |
| TLS certificate | Issued, signed by a publicly trusted CA, and renewed by the add-on | Synced from your Azure Key Vault and self managed |
| Use case | Quickly expose a workload over HTTPS without external dependencies | Workloads that must be served from your own domain |

## Prerequisites

* Install the `aks-preview` Azure CLI extension or update to the latest version using the [`az extension add`][az-extension-add] and [`az extension update`][az-extension-update] commands:

    ```azurecli-interactive
    # Install the aks-preview extension
    az extension add --name aks-preview

    # Update the aks-preview extension to the latest version
    az extension update --name aks-preview
    ```

* Register the `DefaultDomainPreview` feature flag using the [`az feature register`](/cli/azure/feature#az-feature-register) command:

    ```azurecli-interactive
    az feature register --namespace "Microsoft.ContainerService" --name "DefaultDomainPreview"
    ```

    Verify the registration status using the [`az feature show`](/cli/azure/feature#az-feature-show) command:

    ```azurecli-interactive
    az feature show --namespace "Microsoft.ContainerService" --name "DefaultDomainPreview"
    ```

## Enable application routing and the default domain

Set environment variables:

```bash
export CLUSTER=<cluster-name>
export RESOURCE_GROUP=<resource-group-name>
```

Choose the application routing surface you want to attach the default domain to. The `gateway-api` tab uses the [application routing Gateway API implementation][app-routing-gateway-api] (Kubernetes `Gateway` and `HTTPRoute`); the `nginx` tab uses the [managed NGINX ingress controller][app-routing-nginx] (Kubernetes `Ingress`). The page keeps your selection in sync across the rest of the guide.

### Enable during cluster creation

#### [Gateway API](#tab/gateway-api)

Run the following command to create a cluster with the application routing Gateway API implementation and the default domain enabled. The `--app-routing-default-nginx-controller None` flag prevents the default NGINX ingress controller from being provisioned, since you'll attach the default domain to a `Gateway` rather than an `Ingress`:

```azurecli-interactive
az aks create \
    --resource-group ${RESOURCE_GROUP} \
    --name ${CLUSTER} \
    --enable-app-routing-istio \
    --enable-app-routing-default-domain \
    --app-routing-default-nginx-controller None
```

#### [Managed NGINX](#tab/nginx)

Run the following command to create a cluster with the application routing NGINX controller and the default domain enabled:

```azurecli-interactive
az aks create \
    --resource-group ${RESOURCE_GROUP} \
    --name ${CLUSTER} \
    --enable-app-routing \
    --enable-app-routing-default-domain
```

---

### Enable for an existing cluster

#### [Gateway API](#tab/gateway-api)

Run the following command to enable the application routing Gateway API implementation and the default domain on an existing cluster. The `--app-routing-default-nginx-controller None` flag ensures the default NGINX ingress controller is not provisioned:

```azurecli-interactive
az aks update \
    --resource-group ${RESOURCE_GROUP} \
    --name ${CLUSTER} \
    --enable-app-routing-istio \
    --enable-app-routing-default-domain \
    --app-routing-default-nginx-controller None
```

#### [Managed NGINX](#tab/nginx)

Run the following command to enable the application routing NGINX controller and the default domain on an existing cluster:

```azurecli-interactive
az aks update \
    --resource-group ${RESOURCE_GROUP} \
    --name ${CLUSTER} \
    --enable-app-routing \
    --enable-app-routing-default-domain
```

---

### Retrieve the assigned default domain

After the default domain is enabled, AKS generates and assigns a domain name to the cluster. Retrieve it from the managed cluster ingress profile using the [`az aks show`](/cli/azure/aks#az-aks-show) command:

```azurecli-interactive
az aks show \
    --resource-group ${RESOURCE_GROUP} \
    --name ${CLUSTER} \
    --query "ingressProfile.webAppRouting.defaultDomain" \
    -o json
```

```output
{
  "domainName": "<generated-default-domain>",
  "enabled": true
}
```

Save the value for later use:

```bash
export DEFAULT_DOMAIN=$(az aks show \
    --resource-group ${RESOURCE_GROUP} \
    --name ${CLUSTER} \
    --query "ingressProfile.webAppRouting.defaultDomain.domainName" \
    -o tsv)
```

> [!TIP]
> After you create a `DefaultDomainCertificate` (in the next section), you can also read the assigned domain directly from the custom resource's status — for example, `kubectl get defaultdomaincertificate default-domain-cert -n default -o jsonpath='{.status.domain}'`. The status field returns the wildcard form (such as `*.1245.eastus.aksapp.io`), which is convenient when you're working entirely from `kubectl`.

## Configure ingress using the default domain

### Get cluster credentials

```azurecli-interactive
az aks get-credentials --resource-group ${RESOURCE_GROUP} --name ${CLUSTER}
```

### Deploy a sample application

Deploy a sample `hello-world` workload in the `default` namespace:

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - name: hello-world
        image: mcr.microsoft.com/azuredocs/aks-helloworld:v1
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: hello-world
  namespace: default
spec:
  selector:
    app: hello-world
  ports:
  - port: 80
    targetPort: 8080
EOF
```

### Provision a TLS certificate for the default domain

Create a `DefaultDomainCertificate` custom resource. The application routing add-on watches this resource and provisions a TLS certificate for the cluster's default domain, storing it in the Kubernetes secret you specify.

```bash
kubectl apply -f - <<EOF
apiVersion: approuting.kubernetes.azure.com/v1alpha1
kind: DefaultDomainCertificate
metadata:
  name: default-domain-cert
  namespace: default
spec:
  target:
    secret: default-domain-tls
EOF
```

Wait for the certificate to be issued. The add-on populates the secret named in `spec.target.secret` once provisioning completes:

```bash
kubectl wait --for=jsonpath='{.type}'=kubernetes.io/tls \
    secret/default-domain-tls -n default --timeout=5m
```

### Attach the default domain

#### [Gateway API](#tab/gateway-api)

Create a `Gateway` that uses the `approuting-istio` gateway class with a TLS listener bound to the certificate secret, and an `HTTPRoute` that routes traffic for the default domain host name:

```bash
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: hello-world-gateway
  namespace: default
spec:
  gatewayClassName: approuting-istio
  listeners:
  - name: https
    port: 443
    protocol: HTTPS
    hostname: test.${DEFAULT_DOMAIN}
    tls:
      mode: Terminate
      certificateRefs:
      - name: default-domain-tls
    allowedRoutes:
      namespaces:
        from: Same
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: hello-world
  namespace: default
spec:
  parentRefs:
  - name: hello-world-gateway
  hostnames: ["test.${DEFAULT_DOMAIN}"]
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: hello-world
      port: 80
EOF
```

The add-on provisions the Gateway data plane and uses external-dns to publish a DNS record that resolves the host name to the Gateway's public IP.

#### [Managed NGINX](#tab/nginx)

Create an `Ingress` that uses the application routing ingress class, the default domain as the host name, and the certificate secret you provisioned for TLS termination:

```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-world-ingress
  namespace: default
  labels:
    app: hello-world-ingress
spec:
  ingressClassName: webapprouting.kubernetes.azure.com
  tls:
  - hosts:
    - test.${DEFAULT_DOMAIN}
    secretName: default-domain-tls
  rules:
  - host: test.${DEFAULT_DOMAIN}
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hello-world
            port:
              number: 80
EOF
```

The application routing add-on programs the managed NGINX ingress controller to serve traffic for `test.<default-domain>` and uses external-dns to publish a DNS record that resolves the host name to the ingress controller's public IP.

---

### Send a request to the sample application

The add-on's external-dns controller reconciles DNS records on a 3-minute interval, so any time you add a new host name on an `Ingress` or a `Gateway` listener — or change an existing one — it can take up to roughly 3 minutes for the record to be published or updated in Azure DNS. Confirm the record has been published before sending traffic:

```bash
until host test.${DEFAULT_DOMAIN} >/dev/null 2>&1; do
    echo "waiting for DNS record for test.${DEFAULT_DOMAIN}..."
    sleep 15
done
```

Then send an HTTPS request:

```bash
curl -I "https://test.${DEFAULT_DOMAIN}/"
```

You should see an `HTTP 200` response and a TLS certificate issued for the default domain.

## Troubleshooting

If your workload isn't reachable on the default domain, work through the following two checks. Together they cover the two stages the add-on owns: certificate issuance (the `DefaultDomainCertificate`) and DNS publication (the default-domain external-dns controller).

### Check the `DefaultDomainCertificate` custom resource

The `DefaultDomainCertificate` custom resource reports issuance progress through its `.status` and through Kubernetes events. Use it to confirm the add-on accepted the request and successfully populated the target secret.

Inspect the status conditions:

```bash
kubectl get defaultdomaincertificate default-domain-cert -n default \
    -o jsonpath='{.status}' | jq .
```

A healthy resource exposes the assigned `domain`, an `expirationTime`, and a condition with `type=Ready` and `status=True`. A `False` or missing condition usually carries a `reason` and `message` that explain why issuance failed (for example, the cluster's default domain isn't enabled, or the target secret name conflicts with a pre-existing secret the operator doesn't own).

Inspect the events on the resource for the most recent state transitions:

```bash
kubectl describe defaultdomaincertificate default-domain-cert -n default
```

Also confirm that the target secret exists and is of type `kubernetes.io/tls`:

```bash
kubectl get secret default-domain-tls -n default
```

If the secret is missing or has the wrong type, the certificate hasn't been issued yet — work from the condition `message` on the custom resource.

### Check the default-domain external-dns deployment

Once the certificate has been issued and your `Ingress` or `Gateway` is created, the add-on relies on a dedicated external-dns deployment named `default-domain-dns` in the `app-routing-system` namespace to publish the DNS record. If the certificate looks healthy but the host name still doesn't resolve after the 3-minute reconcile interval, inspect that deployment.

Check the deployment status:

```bash
kubectl get deployment default-domain-dns -n app-routing-system
kubectl get pods -n app-routing-system -l app.kubernetes.io/name=external-dns
```

The deployment should report `READY`. If pods are pending, crash-looping, or showing image pull errors, follow the standard pod-diagnosis path (`kubectl describe pod ...`).

Stream the controller logs to see how it's reconciling your record:

```bash
kubectl logs -n app-routing-system deployment/default-domain-dns --tail=200 -f
```

Look for log lines that name your host (`test.<default-domain>`). Common signals:

* `All records are already up to date` — external-dns hasn't seen a new desired record. Confirm that the host name matches the cluster's default domain.
* `Records to be created` followed by an Azure DNS API error — the controller is trying to publish but the underlying call is failing. The error message identifies the cause.
* No mention of your host at all — the resource hasn't been picked up. Verify the namespace and label selectors on your `Ingress` or `Gateway` and confirm the resource is `Programmed` / `Accepted` from the Kubernetes side.

## Disable the default domain

Run the following command to disable the default domain while keeping the application routing add-on enabled:

```azurecli-interactive
az aks update \
    --resource-group ${RESOURCE_GROUP} \
    --name ${CLUSTER} \
    --disable-app-routing-default-domain
```

### Clean up resources

Delete the `DefaultDomainCertificate` and the sample application:

```bash
kubectl delete defaultdomaincertificate default-domain-cert -n default
kubectl delete deployment hello-world -n default
kubectl delete service hello-world -n default
```

Then delete the ingress resource you created.

#### [Gateway API](#tab/gateway-api)

```bash
kubectl delete httproute hello-world -n default
kubectl delete gateway hello-world-gateway -n default
```

#### [Managed NGINX](#tab/nginx)

```bash
kubectl delete ingress hello-world-ingress -n default
```

---

The TLS secret is owned by the `DefaultDomainCertificate` and is removed automatically when the custom resource is deleted.

## Next steps

* [Managed NGINX ingress with the application routing add-on][app-routing-nginx]
* [Configure custom DNS and SSL with the application routing add-on][app-routing-dns-tls]
* [Configure ingress with the Kubernetes Gateway API via the application routing add-on][app-routing-gateway-api]

<!-- LINKS - internal -->
[app-routing-nginx]: app-routing.md
[app-routing-dns-tls]: app-routing-dns-ssl.md
[app-routing-gateway-api]: app-routing-gateway-api.md
[az-extension-add]: /cli/azure/extension#az-extension-add
[az-extension-update]: /cli/azure/extension#az-extension-update
