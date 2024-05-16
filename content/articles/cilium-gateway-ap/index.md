---
title: "Gateway API with Cilium and Cert-manager"
date: 2024-05-15
draft: false
description: "a description"
tags: ["api", "tag", "http"]
views: 52
---

The [Gateway API SIG](https://google.com) (Special Interest Group) recently [released v1.0]() which spurred my interest in the project.
In their own words,
> If you’re familiar with the older Ingress API, you can think of the Gateway API as analogous to a more-expressive next-generation version of that API.

In this article we’ll quickly review the role-oriented architecture of the Gateway API before we implement it using Cilium and Cert-manager. Other Gateway API implementations are listed on the Gateway SIG site.

We’ll mainly take a look at replacing `Ingress` resources for traffic from clients outside the cluster to services inside the cluster ( north/south traffic). Although the Gateway API also supports so-called east/west traffic between workloads within a cluster (through the GAMMA-initiative), this is outside the scope of this article.

Before reading this article you might want to try a hands-on lab on Cilium Gateway API by Isovalent, the company behind Cilium.

{{< alert >}}
Eating my own dog food this blog is now delivered through the Gateway configuration explained here.
{{< /alert >}}

## Overview

In the role-oriented design of the Gateway API, the infrastructure provider provisions a GatewayClass which the cluster operators can use to create different Gateway resources for consumption by the application developers using HTTPRoutes connecting to plain old Services.

Comparing this with the Ingress API we see that the Ingress resource has been split into the Gateway and HTTPRoute objects with different responsibilities.

## Gateway API

Kubernetes 1.28 doesn’t ship with the Gateway API CRDs (Custom Resource Definitions), we therefore need to manually apply them ourselves.

The latest Gateway API release can be found in their GitHub repository under releases. We’re interested in trying out some of the experimental features (the GatewayInfrastructure), so we’ll apply the experimental installation by running

```
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/experimental-install.yaml
```

This effectively enables the Gateway API, and we can start using it.

## Cilium

Cilium already has a page on Migrating from Ingress to Gateway in their documentation, gearing up to fully support Gateway API v1.0.0 in Cilium v1.15.0.

This article is mostly written using Cilium v1.14.5 which passes conformance tests for Gateway API v0.7.1, though I did not run in to any issues using core functionality for Gateway API v1.0.0.

Following the documentation of Cilium we can enable Gateway support in one of two ways, either with the Cilium-CLI ( ≥ v0.15)

cilium install --version 1.14.5 \
    --set kubeProxyReplacement=true --set gatewayAPI.enabled=true
or using the Helm Chart as described in the summary section.

## Cert-manager

Gateway API support is an experimental feature in the latest stable release of Cert-manager at the time of writing (v1.13.3). To enable this support we add the --feature-gates=ExperimentalGatewayAPISupport=true flag on startup of Cert-manager. This is done by setting it as an extra argument when installing Cert-manager using its Helm Chart

helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager --version v1.13.3 \
    --namespace cert-manager --set installCRDs=true --create-namespace \
    --set "extraArgs={--feature-gates=ExperimentalGatewayAPISupport=true}"
## Configuration

Once we have the Gateway API CRDs available and enabled support for it in Cilium and Cert-manager we can finally start to create the resources we need to utilise it.

### Infrastructure provider

If a cluster wide GatewayClass resource referencing Cilium is not already present (kubectl get gatewayclasses) we need to create one ourselves1

```
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: cilium
spec:
  controllerName: io.cilium/gateway-controller
```

Take note of the GatewayClass name (line 4) and make sure of the controllerName on line 6.

If the GatewayClass is created successfully you should be able to view the supported features by running

kubectl describe gatewayclass cilium
Cluster operator
#
For convenience, we’ll group the cluster operator related resources in the gateway namespace. This allows us an easy overview of our gateways and connected resources as cluster operators.

kubectl create ns gateway
TLS certificates (Cloudflare)
#
To automatically provision TLS certificates attached to our Gateway we can create a Cert-manager Issuer resource. This section is optional if you don’t need certificates, though I highly recommend it!

For details on how to automatically provision wildcard certificates using Cert-manager and Let’s Encrypt I’ve summarised the process in a previous article on Traefik Wildcard Certificates, so I’ll allow myself to be brief here.

Obtain a Cloudflare API token (or from you supported DNS provider of choice) as mentioned in the above article and create a Secret for it

1
2
3
4
5
6
7
8
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-token
  namespace: gateway
type: Opaque
stringData:
  api-token: "<--CLOUDFLARE API TOKEN-->"
We can then reference this secret in an Issuer resource (line 16) which enables us to complete a DNS-01 challenge that allows us to issue wildcard certificates for the proven domain. Remember to provide the domain owner e-mail on line 9.

 1
 2
 3
 4
 5
 6
 7
 8
 9
10
11
12
13
14
15
16
17
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: cloudflare-issuer
  namespace: gateway
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: "<--YOUR EMAIL-->"
    privateKeySecretRef:
      name: cloudflare-key
    solvers:
      - dns01:
          cloudflare:
            apiTokenSecretRef:
              name: cloudflare-api-token
              key: api-token
Gateway
#
Next we create a Gateway resources that references the cilium GatewayClass (line 9). The annotation on line 7 is picked up by Cert-manager to automatically issue certificates from the cloudflare-issuer Issuer and create a TLS-Secret with the name with provided on line 18 when all goes well.

 1
 2
 3
 4
 5
 6
 7
 8
 9
10
11
12
13
14
15
16
17
18
19
20
21
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: cilium-gateway
  namespace: gateway
  annotations:
    cert-manager.io/issuer: cloudflare-issuer
spec:
  gatewayClassName: cilium
  listeners:
    - protocol: HTTPS
      port: 443
      name: https-gateway
      hostname: "*.<--YOUR DOMAIN-->"
      tls:
        certificateRefs:
          - kind: Secret
            name: cloudflare-cert
      allowedRoutes:
        namespaces:
          from: All
We need to create at least one listener per Gateway that listens for e.g. HTTPRoute resources that matches. In our case we’ve created an HTTPS-listener on port 443 that matches all subdomains. The tls-field is picked up by Cert-manager which will create a TLS-secret with the given name when an HTTPRoute is attached. We allow eligible HTTPRoutes from all namespaces to connect through this Gateway, but we can also create a selector for more fine-grained control.

Gateway Service
#
Cilium 1.15.0 released on 2024-01-31 and fully supports the Gateway infrastructure annotation discussed below.
A LoadBalancer Service is created when the Gateway is picked up by the Cilium controller. As of Cilium 1.14.5 there appears to be no methods to directly manipulate the spawned Service from the Gateway to e.g. annotate it for requesting a specific IP address.

I strive for an idempotent configuration for my homelab and thus prefer static IPs for my LoadBalancer Services. Running Cilium LB-IPAM a way to do this is by annotating the Service with io.cilium/lb-ipam-ips: "<--IP-->".

At first, I hoped that annotating the Gateway may propagate to the Service. I was apparently not the first to think this might work according to a closed GitHub issue which states Gateway annotation propagation will not be implemented.

In a thread on Reddit u/nuskovg points to the GatewayAddress field used by the GatewaySpec as a possible solution. The support for this field is extended, and u/TheGarbInC mentions an open GitHub issue for Cilium support which is yet undecided.

Digging deeper, u/h_hover can inform of an experimental GatewayInfrastructure field and a closed GitHub issue cautiously promising support in Cilium 1.15.

This then led me to optimistically try Cilium 1.15.0-rc.0 and add

infrastructure:
  annotations:
    io.cilium/lb-ipam-ips: "<--IP-->"
in my Gateway spec… And it works! The Gateway Service now has the requested annotation, and we have a deterministic configuration!

DNS
#
Now that you’ve got your Gateway and attached LoadBalancer Service set up you want to point you DNS to the Service IP. This IP address should be the same as what you set in the Gateway infrastructure annotation field (io.cilium/lb-ipam-ips), but to make sure you can run

kubectl get svc -A | grep LoadBalancer
and find the External IP of the Service named <GatewayClass.name>-gateway-<Gateway.name>, in our case cilium-gateway-cilium-gateway.2

Open up port 443 in your router or firewall to the Service IP and point you DNS to your public IP. If you don’t have your public IP at hand you can find it by running

dig +short myip.opendns.com @resolver1.opendns.com
If everything is set up correctly, an external web request should roughly take the following path:

Web
DNS
Router
Service
Gateway
First the hostname is looked up in a DNS. The DNS should respond with the IP you set up, and the request is relayed to your router. Next the router port forwards the request to the Service IP connected to the Gateway. The Gateway then presents the attached certificate and the journey continues.

Gateway
HTTPRoute ɑ
Service
Pod/Application
HTTPRoute β
HTTPRoute γ
From the Gateway the request is channeled to the correct HTTPRoute – route ɑ in this case, based on the rules you’ve set up, e.g. hostname or header-matching. Next, the HTTPRoute directs the request to its attached Service, which then finally delivers the request to its destination. Hopefully the application responds with something nice.

If you don’t want to expose your public IP, you can instead use a service like cloudflared to tunnel traffic directly to your cluster. If you want to go this route you can find an example configuration here.

Application developer
#
Now that both the infrastructure provider and cluster operator have done their job (kudos to you!), we can let the application developers (also you) take the centre stage.

Given a Service named my-service we can easily create a simpleHTTPRoute referencing our Gateway (line 7) and the Service (line 17) to expose the Service

 1
 2
 3
 4
 5
 6
 7
 8
 9
10
11
12
13
14
15
16
17
18
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-http-route
spec:
  parentRefs:
    - name: cilium-gateway
      namespace: gateway
  hostnames:
    - "gateway.<--YOUR DOMAIN-->"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: my-service
          port: 80
HTTPRoutes also allows developers to easily do header based routing for canary deployments, or traffic splitting for blue-green testing.

I strongly encourage you to take a look all the capabilities on the Gateway API user guide for more ideas.

Summary
#
I’m running Argo CD with Kustomize + Helm in an attempt to follow GitOps best-practices. This summary assumes a similar setup together with Sealed Secrets. My full homelab configuration as of the writing of this article can be found on GitHub as a reference.

Gateway API
#
We gather all resources related to the Gateway in one namespace. This includes the Cert-manager Issuer.

#gateway/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/experimental-install.yaml
  - gateway-class.yaml
  - ns.yaml
  - cloudflare-api-token.yaml
  - cloudflare-issuer.yaml
  - gateway.yaml
#gateway/gateway-class.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: cilium
spec:
  controllerName: io.cilium/gateway-controller
#gateway/ns.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: gateway
#gateway/cloudflare-api-token.yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: cloudflare-api-token
  namespace: gateway
spec:
  encryptedData:
    api-token: <--Sealed Cloudflare API Token-->
  template:
    metadata:
      name: cloudflare-api-token
      namespace: gateway
    type: Opaque
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: cloudflare-issuer
  namespace: gateway
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: "<--YOUR EMAIL-->"
    privateKeySecretRef:
      name: cloudflare-key
    solvers:
      - dns01:
          cloudflare:
            apiTokenSecretRef:
              name: cloudflare-api-token
              key: api-token
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: cilium-gateway
  namespace: gateway
  annotations:
    cert-manager.io/issuer: cloudflare-issuer
spec:
  gatewayClassName: cilium
  infrastructure:
    annotations:
      io.cilium/lb-ipam-ips: "<--IP-->"
  listeners:
    - protocol: HTTPS
      port: 443
      name: https-subdomains-gateway
      hostname: "*.<--YOUR DOMAIN-->"
      tls:
        certificateRefs:
          - kind: Secret
            name: cloudflare-cert
      allowedRoutes:
        namespaces:
          from: All
    - protocol: HTTPS
      port: 443
      name: https-domain-gateway
      hostname: "<--YOUR DOMAIN-->"
      tls:
        certificateRefs:
          - kind: Secret
            name: cloudflare-domain-cert
      allowedRoutes:
        namespaces:
          from: All
Cilium
#
Cilium is configured to use v1.15.0 to support Gateway Service annotation which works in conjunction with Cilium LB-IPAM and L2 announcements.

#cilium/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ip-pool.yaml
  - announce.yaml

helmCharts:
  - name: cilium
    repo: https://helm.cilium.io
    version: 1.15.0
    releaseName: "cilium"
    includeCRDs: true
    namespace: kube-system
    valuesFile: values.yaml
#cilium/values.yaml
kubeProxyReplacement: true

gatewayAPI:
  enabled: true

# Roll out cilium agent and operator pods automatically when ConfigMap is updated.
rollOutCiliumPods: true

operator:
  rollOutPods: true

# Increase rate limit when doing L2 announcements
k8sClientRateLimit:
  qps: 100
  burst: 200

l2announcements:
  enabled: true
apiVersion: cilium.io/v2alpha1
kind: CiliumLoadBalancerIPPool
metadata:
  name: default-pool
  namespace: kube-system
spec:
  cidrs:
    - cidr: "<--Valid CIDR range-->"
apiVersion: cilium.io/v2alpha1
kind: CiliumL2AnnouncementPolicy
metadata:
  name: default-l2-announcement-policy
  namespace: kube-system
spec:
  interfaces:
    ["<--eth interfaces to announce on-->"]
  loadBalancerIPs: true
Cert-manager
#
#cert-manager/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ns.yaml

helmCharts:
  - name: cert-manager
    repo: https://charts.jetstack.io
    version: 1.13.3
    releaseName: cert-manager
    namespace: cert-manager
    valuesInline:
      installCRDs: true
      extraArgs:
        - "--feature-gates=ExperimentalGatewayAPISupport=true"
#cert-manager/ns.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: cert-manager
Cloudflared
#
For completeness’s sake this is the relevant cloudflared config I’m currently running.

ingress:
  - hostname: blog.stonegarden.dev
    service: https://cilium-gateway-cilium-gateway.gateway.svc.cluster.local:443
    originRequest:
      originServerName: blog.stonegarden.dev
  - hostname: remark42.stonegarden.dev
    service: https://cilium-gateway-cilium-gateway.gateway.svc.cluster.local:443
    originRequest:
      originServerName: remark42.stonegarden.dev
  - hostname: gateway.stonegarden.dev
    service: https://cilium-gateway-cilium-gateway.gateway.svc.cluster.local:443
    originRequest:
      originServerName: gateway.stonegarden.dev
  - hostname: stonegarden.dev
    service: https://cilium-gateway-cilium-gateway.gateway.svc.cluster.local:443
    originRequest:
      originServerName: stonegarden.dev
  - hostname: "*.stonegarden.dev"
    service: https://traefik.traefik.svc.cluster.local:443
    originRequest:
      originServerName: "*.stonegarden.dev"
  - service: http_status:404
In the Isovalent Cilium Gateway API lab this GatewayClass is already created for you. ↩︎

If I knew about this name scheme I would’ve picked better names for my Gateway and GatewayClass resources. ↩︎

This is a paragraph with **bold** and *italic* text.
Check more at [Blowfish documentation](https://blowfish.page/)
undefined
