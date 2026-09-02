---
title: Use the managed default domain with the application routing add-on for Azure Kubernetes Service (AKS)
description: Learn how to give an AKS cluster a free managed domain and TLS certificate by using the default domain feature of the application routing add-on.
ms.subservice: aks-networking
ms.author: oliverking
author: kingoliver
ms.service: azure-kubernetes-service
ms.custom: devx-track-azurecli
ms.topic: how-to
ms.date: 09/02/2026
# Customer intent: As a Kubernetes administrator, I want to expose a workload on a free managed domain with a trusted TLS certificate, so that I can serve HTTPS traffic without buying a domain or managing certificates.
---

# Use the managed default domain with the application routing add-on for Azure Kubernetes Service (AKS)

The default domain feature of the [application routing add-on](./app-routing.md) gives your cluster a free, fully managed domain and a trusted TLS certificate. When you enable the feature, Azure Kubernetes Service (AKS) assigns your cluster a unique domain in the form `*.<id>.<region>.aksapp.io`. AKS then issues a publicly trusted wildcard TLS certificate for that domain and publishes the Domain Name System (DNS) records automatically.

Use this feature to serve HTTPS traffic for a workload without buying a domain, creating an Azure DNS zone, or managing a certificate. AKS manages the domain, the certificate, and the DNS records for you. The certificate is renewed automatically before it expires.

[!INCLUDE [preview features callout](~/reusable-content/ce-skilling/azure/includes/aks/includes/preview/preview-callout.md)]

## How the feature works

The default domain feature builds on the application routing add-on. When you enable it, AKS does the following:

- Assigns a read-only domain name to the cluster, in the form `*.<id>.<region>.aksapp.io`. The name doesn't change even if you disable and reenable the feature.
- Issues a publicly trusted wildcard TLS certificate for the domain and reconciles it into a Kubernetes Secret that you name.
- Runs a managed `external-dns` instance that publishes address (A) records for the hostnames you expose.

You bring the workload and a small amount of configuration. AKS brings the domain, the certificate, and the DNS.

## Prerequisites

- The Azure CLI version `2.86.0` or later. Run `az --version` to check your version, and run `az upgrade` to update. If you don't have the Azure CLI, see [Install the Azure CLI](/cli/azure/install-azure-cli).
- The `aks-preview` Azure CLI extension, version `19.0.0b23` or later. This version adds the `--enable-default-domain` and `--disable-default-domain` parameters. Install or update it with the following command:

  ```azurecli-interactive
  az extension add --name aks-preview
  az extension update --name aks-preview
  ```

  Run `az extension show --name aks-preview --query version` to check your installed version.

- An AKS cluster that uses a managed identity. The default domain feature is part of the application routing add-on, which you enable in the next section.
- The [application routing Kubernetes Gateway API implementation][app-routing-gateway-api]. This article exposes the sample application through the Gateway API. Enable it with the `--enable-app-routing-istio` flag, as shown in the next section. It requires the [Managed Gateway API installation][managed-gateway-api].

## Enable the default domain feature

You enable the default domain feature together with the application routing add-on. The `--enable-default-domain` flag turns on the feature.

### Enable during cluster creation

Run the following command to create a cluster with the application routing add-on and the default domain feature:

```azurecli-interactive
# Set environment variables
export RESOURCE_GROUP=<resource-group-name>
export CLUSTER_NAME=<cluster-name>
export LOCATION=<location>

# Create the cluster with the application routing add-on and the default domain feature
az aks create \
    --resource-group ${RESOURCE_GROUP} \
    --name ${CLUSTER_NAME} \
    --location ${LOCATION} \
    --enable-managed-identity \
    --enable-app-routing \
    --enable-app-routing-istio \
    --enable-default-domain \
    --generate-ssh-keys
```

### Enable on an existing cluster

Run the following command to enable the default domain feature on a cluster that already has the application routing add-on:

```azurecli-interactive
az aks approuting update \
    --resource-group ${RESOURCE_GROUP} \
    --name ${CLUSTER_NAME} \
    --enable-default-domain

# Enable the Gateway API implementation if it isn't already enabled
az aks update \
    --resource-group ${RESOURCE_GROUP} \
    --name ${CLUSTER_NAME} \
    --enable-app-routing-istio
```

## Get the assigned domain name

AKS assigns a unique domain to the cluster. Run the following command to read it:

```azurecli-interactive
az aks approuting defaultdomain show \
    --resource-group ${RESOURCE_GROUP} \
    --name ${CLUSTER_NAME}
```

The command returns the domain name in the form `*.<id>.<region>.aksapp.io`, for example `*.6a27935b.westus2.aksapp.io`. You expose your workloads on this domain, or on subdomains of it.

> [!TIP]
> If you prefer a Kubernetes-native workflow, you can also read the assigned domain from the `status.domain` field of the `DefaultDomainCertificate` resource after you create it. This lets you get the domain by using only the Kubernetes API, without the Azure CLI. See [Request the managed certificate](#request-the-managed-certificate).

## Connect to your cluster

Configure `kubectl` to connect to your cluster with the [`az aks get-credentials`][az-aks-get-credentials] command:

```azurecli-interactive
az aks get-credentials --resource-group ${RESOURCE_GROUP} --name ${CLUSTER_NAME}
```

## Request the managed certificate

To get the managed TLS certificate, create a `DefaultDomainCertificate` resource. This resource tells AKS which Kubernetes Secret to store the certificate in. AKS reconciles the certificate into that Secret in the **same namespace** as the `DefaultDomainCertificate` resource. This example uses a dedicated `hello-world` namespace so that the certificate, the `Gateway`, and the workload all live together. A `Gateway` can reference a certificate Secret only from its own namespace, so keep these resources in the same namespace.

1. Create the namespace:

    ```bash
    kubectl create namespace hello-world
    ```

1. Create a file named `default-domain-certificate.yaml` that has the following content. The `metadata.namespace` field places the resource, and therefore the resulting Secret, in the `hello-world` namespace:

    ```yaml
    apiVersion: approuting.kubernetes.azure.com/v1alpha1
    kind: DefaultDomainCertificate
    metadata:
      name: cert
      namespace: hello-world
    spec:
      target:
        secret: cert
    ```

1. Apply the resource with the [`kubectl apply`][kubectl-apply] command:

    ```bash
    kubectl apply -f default-domain-certificate.yaml
    ```

1. Confirm that the certificate is ready. AKS issues the certificate after you create the resource, which usually takes a few minutes. Run the following command and check that the `Available` condition reports `True`:

    ```bash
    kubectl get defaultdomaincertificate cert -n hello-world -o jsonpath='{.status.conditions[?(@.type=="Available")].reason}'
    ```

    Until the certificate is ready, the command returns `CertificateNotReady`. When the certificate is ready, the command returns `CertificateSecretApplied`, and the Secret named `cert` in the `hello-world` namespace holds the certificate. To wait for the certificate instead of checking manually, run the following command:

    ```bash
    kubectl wait --for=condition=Available defaultdomaincertificate/cert -n hello-world --timeout=5m
    ```

    Confirm the Secret exists:

    ```bash
    kubectl get secret cert -n hello-world
    ```

1. Read the assigned domain from the resource status. The `DefaultDomainCertificate` resource reports the domain in its `status.domain` field, so you can get it without the Azure CLI:

    ```bash
    kubectl get defaultdomaincertificate cert -n hello-world -o jsonpath='{.status.domain}'
    ```

    The command returns the assigned domain, for example `*.6a27935b.westus2.aksapp.io`.

## Deploy a sample application

The following steps deploy a sample application into the `hello-world` namespace and expose it on the default domain.

1. Create a file named `hello-world.yaml` that has the following content:

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: hello-world
      namespace: hello-world
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
            - containerPort: 80
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: hello-world
      namespace: hello-world
    spec:
      selector:
        app: hello-world
      ports:
      - port: 80
        targetPort: 80
    ```

1. Apply the manifest:

    ```bash
    kubectl apply -f hello-world.yaml
    ```

## Expose the application on the default domain

Create a `Gateway` and an `HTTPRoute` that use the `approuting-istio` GatewayClass and the managed certificate. Because the managed certificate is a wildcard certificate for `*.<id>.<region>.aksapp.io`, you can use the following as the hostname:

- A subdomain, such as `hello.<id>.<region>.aksapp.io`. This example uses a subdomain.
- The apex domain itself, `<id>.<region>.aksapp.io`.

You can also route different paths on the same hostname to different services with `HTTPRoute` rules. The `Gateway`, `HTTPRoute`, and workload run in the `hello-world` namespace, alongside the certificate Secret.

1. Set a hostname under your assigned domain. Replace `<id>` and `<region>` with the values from your assigned domain:

    ```bash
    export HOST=hello.<id>.<region>.aksapp.io
    ```

1. Create the `Gateway`. The listener terminates TLS by using the managed certificate in the `cert` Secret in the same namespace:

    ```bash
    cat <<EOF | kubectl apply -f -
    apiVersion: gateway.networking.k8s.io/v1
    kind: Gateway
    metadata:
      name: hello-world
      namespace: hello-world
    spec:
      gatewayClassName: approuting-istio
      listeners:
      - name: https
        hostname: "${HOST}"
        port: 443
        protocol: HTTPS
        tls:
          mode: Terminate
          certificateRefs:
          - name: cert
        allowedRoutes:
          namespaces:
            from: Same
    EOF
    ```

1. Create the `HTTPRoute` that routes traffic to the sample service:

    ```bash
    cat <<EOF | kubectl apply -f -
    apiVersion: gateway.networking.k8s.io/v1
    kind: HTTPRoute
    metadata:
      name: hello-world
      namespace: hello-world
    spec:
      parentRefs:
      - name: hello-world
      hostnames:
      - "${HOST}"
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

When you apply the `Gateway`, the application routing add-on publishes an A record for the listener hostname to the managed DNS zone and terminates TLS by using the managed certificate.

## Verify the application

DNS propagation takes a short time after you create the `Gateway`. Wait a minute, then verify that the domain resolves and serves the trusted certificate.

1. Wait for the `Gateway` to be programmed, then get its public address:

    ```bash
    kubectl wait --for=condition=programmed gateways.gateway.networking.k8s.io hello-world -n hello-world
    export INGRESS_HOST=$(kubectl get gateways.gateway.networking.k8s.io hello-world -n hello-world -o jsonpath='{.status.addresses[0].value}')
    ```

1. Confirm that the hostname resolves to the gateway address:

    ```bash
    nslookup ${HOST}
    ```

    The command returns the public IP address of the gateway. If it returns no answer, wait a minute for DNS to propagate, then try again.

1. Send an HTTPS request and check the certificate:

    ```bash
    curl -sS -o /dev/null -w "HTTP %{http_code} verify=%{ssl_verify_result}\n" "https://${HOST}/"
    ```

    A result of `HTTP 200 verify=0` means the request succeeded and the certificate validated against a trusted certificate authority. You don't need the `curl -k` flag, because the certificate is publicly trusted.

You can also open `https://<HOST>` in a browser and confirm that the browser shows a valid, trusted certificate.

## Supported ingress types

The default domain feature works with the ingress options that the application routing add-on manages:

- The [application routing Kubernetes Gateway API implementation](./app-routing-gateway-api.md), as shown in this article. Use the `approuting-istio` GatewayClass, and reference the managed certificate Secret in the `Gateway` listener `certificateRefs` field.
- The managed NGINX ingress controller. Use the `webapprouting.kubernetes.azure.com` ingress class, and reference the managed certificate Secret in the `Ingress` `tls.secretName` field.

Self-managed ingress controllers and self-managed Gateway API resources aren't supported with the default domain feature.

## Disable the default domain feature

Run the following command to disable the feature:

```azurecli-interactive
az aks approuting update \
    --resource-group ${RESOURCE_GROUP} \
    --name ${CLUSTER_NAME} \
    --disable-default-domain
```

The assigned domain name is retained. If you reenable the feature later, the cluster keeps the same domain.

## Troubleshoot

If the certificate doesn't become ready, or the domain doesn't resolve, see [Troubleshoot the application routing default domain feature](/troubleshoot/azure/azure-kubernetes/extensions/application-routing-default-domain).

## Next steps

- Learn more about the [application routing add-on](./app-routing.md).
- Configure ingress with the [Kubernetes Gateway API implementation](./app-routing-gateway-api.md).
- Use a [custom domain name and certificate](./app-routing-dns-ssl.md) instead of the managed default domain.

<!-- LINKS - internal -->
[az-aks-get-credentials]: /cli/azure/aks#az-aks-get-credentials
[kubectl-apply]: https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#apply
[app-routing-gateway-api]: ./app-routing-gateway-api.md
[managed-gateway-api]: ./managed-gateway-api.md
