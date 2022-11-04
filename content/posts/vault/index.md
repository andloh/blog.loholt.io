---
title: "Using HashiCorp Vault on Kubernetes"
subtitle: "Manage your secrets using Vault with the bank-vaults"
date: 2022-05-16
tags:
  - Open Source
  - Secret Management
  - Vault
  - Kubernetes
  - MultiCluster
  - K8s Operators
draft: true

featuredImage: "/images/vault-arch.png"
featuredImagePreview: "/images/vault-arch.png"

code:
  copy: true
  maxShownLines: 50

share:
  enable: true

comment:
  enable: true

---

Manage your secrets using **HashiCorp Vault**
<!--more-->

## Bank-vaults

Vault being one of the most known secret management systems out there in the CloudNative world, are still missing an operator for k8s. Right now they just provide a helm chart for installing and configuring. This means we can't configure Vault as a part of our GitOps workflow, at least not without restarting Vault with every change. 

The current feature for injecting secrets in the official chart is [Agent Inject](https://developer.hashicorp.com/vault/docs/platform/k8s/injector). To my understanding it only supports injecting secrets via a shared memory volume that needs to be mounted in your application. 

[**Bank Vaults**](https://github.com/banzaicloud/bank-vaults), a project by [BanzaiCloud](https://banzaicloud.com/) offers these features for integrating with Kubernetes. 

- Operator driven install and configuration
- Inject secrets into environment variables via memory volume
- Auto unseal with multiple options
- OpenSource
- Inject secret/vault data into standard K8s secrets
- Inject secret/vault data into standard K8s ConfigMaps

The two last features can brings a more abstract way to your use of Vault with Kubernetes. E.g shared secrets or configuration across namespaces or clusters.

{{< admonition type=note title="Note" open=true >}}
**BanzaiCloud** is now part of Cisco. Read more here: [Acquisitions](https://www.cisco.com/c/en/us/about/corporate-strategy-office/acquisitions/banzaicloud.html)
{{< /admonition >}}

## Prerequisites

I will be using FluxCDv2 for installing the required components via Helm. Make sure you have a working Kubernetes cluster that's already syncing configuration from a git repository.

You will also need a StorageClass (file or block) and a IngressController. 

## Installation

Official documentation: `https://banzaicloud.com/docs/bank-vaults/overview/`. Covers installing operators with helm or/and `kubectl apply`

### Add Helm Repo

Manifest for adding BanzaiCloud helm repo:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: HelmRepository
metadata:
  name: banzaicloud
  namespace: vault
spec:
  interval: 5m
  url: https://kubernetes-charts.banzaicloud.com
```

### Vault Operator (Flux HelmRelease)

Manifest for Vault Operator:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: vault-operator
  namespace: vault
spec:
  interval: 5m
  chart:
    spec:
      chart: vault-operator
      version: 1.16.0
      sourceRef:
        kind: HelmRepository
        name: banzaicloud
        namespace: vault
  values:
    resources:
      limits:
        memory: 512Mi
```

### Vault Secret Webhook (Flux HelmRelease)

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: vault-secrets-webhook
  namespace: vault
spec:
  interval: 5m
  chart:
    spec:
      chart: vault-secrets-webhook
      version: 1.16.0
      sourceRef:
        kind: HelmRepository
        name: banzaicloud
        namespace: vault
  values:
    resources:
      limits:
        memory: 512Mi
    secretsFailurePolicy: Fail
    configMapMutation: false # enable this if you want to mutate your ConfigMaps
    podsFailurePolicy: Fail
```

### Installing Vault

In this installation manifest we will configure:

- A clusters instance with raft
- Auto unseal
- Some sample roles and mappings
- Create an ingress object
- A sample OIDC integration
- Two secret engines

For all these features to work, you will need to change some values. Like your, ingressclass name, storageclass name, OIDC endpoint and OIDC group name (this configuration will assume you have the group `vault-admins` in your OIDC group scope)

```yaml
apiVersion: vault.banzaicloud.com/v1alpha1
kind: Vault
metadata:
    name: vault
    namespace: vault
    labels:
        app.kubernetes.io/name: vault
        vault_cr: vault
spec:
    size: 3
    image: vault:1.10.5 # Make sure the version of Vault is compatible with the operator
    # Common annotations for all created resources
    annotations:
        common/annotation: "true"
    # Vault Pods , Services and TLS Secret annotations
    vaultAnnotations:
        type/instance: vault
    # Vault Configurer Pods and Services annotations
    vaultConfigurerAnnotations:
        type/instance: vaultconfigurer
    # Vault Pods , Services and TLS Secret labels
    vaultLabels:
        example.com/log-format: json
    # Vault Configurer Pods and Services labels
    vaultConfigurerLabels:
        example.com/log-format: string
    # Support for affinity Rules
    # affinity:
    #   nodeAffinity:
    #     requiredDuringSchedulingIgnoredDuringExecution:
    #       nodeSelectorTerms:
    #       - matchExpressions:
    #         - key : "node-role.kubernetes.io/your_role"
    #           operator: In
    #           values: ["true"]
    # Support for pod nodeSelector rules to control which nodes can be chosen to run
    # the given pods
    #nodeSelector:
    #    node-role.kubernetes.io/infra: ""
    # Support for node tolerations that work together with node taints to control
    # the pods that can like on a node
    # Specify the ServiceAccount where the Vault Pod and the Bank-Vaults configurer/unsealer is running
    serviceAccount: vault
    # Specify the Service's type where the Vault Service is exposed
    # Please note that some Ingress controllers like https://github.com/kubernetes/ingress-gce
    # forces you to expose your Service on a NodePort
    serviceType: ClusterIP
    # Request an Ingress controller with the default configuration
    ingress:
    # Specify Ingress object annotations here, if TLS is enabled (which is by default)
    # the operator will add NGINX, Traefik and HAProxy Ingress compatible annotations
    # to support TLS backends
    annotations:
      nginx.ingress.kubernetes.io/rewrite-target: "/"
      nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
      nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    spec:
      ingressClassName: your-ingress-class
      rules:
      - host: HOSTNAME
        http:
          paths:
          - pathType: Prefix
            path: /
            backend:
              service:
                name: vault
                port:
                  number: 8200
    volumeClaimTemplates:
        - metadata:
            name: vault-raft
          spec:
            storageClassName: my-storage-class
            accessModes:
                - ReadWriteOnce
            volumeMode: Filesystem
            resources:
                requests:
                    storage: 32Gi
    volumeMounts:
        - name: vault-raft
          mountPath: /vault/file
    veleroEnabled: false
    caNamespaces:
        - vault
    unsealConfig:
        options:
            preFlightChecks: true
        kubernetes:
            secretNamespace: vault
    config:
        storage:
            raft:
                path: /vault/file
        listener:
            tcp:
                address: 0.0.0.0:8200
                tls_cert_file: /vault/tls/server.crt
                tls_key_file: /vault/tls/server.key
        api_addr: https://vault.vault:8200
        cluster_addr: https://${.Env.POD_NAME}:8201
        ui: true
    statsdDisabled: true
    serviceRegistrationEnabled: true
    resources:
        vault:
            limits:
                memory: 512Mi
                cpu: 200m
            requests:
                memory: 256Mi
                cpu: 100m
    externalConfig:
        audit:
            - description: STDOUT Audit logging
              options:
                file_path: stdout
              type: file
        policies:
            - name: vault-admins
              rules: path "*" { capabilities = ["create", "read", "update", "delete", "list", "sudo"] }
            - name: blue
              rules: path "blue" { capabilities = ["create", "read", "update", "delete", "list"] }
        auth:
            - type: kubernetes
              config:
                disable_iss_validation: true
              roles:
                - name: blue
                  bound_service_account_names:
                    - default
                    - vault-secrets-webhook
                  bound_service_account_namespaces:
                    - blue-*
                    - vault
                  policies:
                    - blue
                  ttl: 1h
            - type: oidc
              config:
                default_role: k8s-vault
                oidc_client_id: vault
                oidc_client_secret: client_secret
                oidc_discovery_url: my-oidc-url
                options:
                    listing_visibility: unauth
                provider_config: []
              roles:
                - name: vault-admins
                  oidc_discovery_url: my-oidc-url
                  allowed_redirect_uris: https://VAULTURL/ui/vault/auth/oidc/oidc/callback
                  role_type: oidc
                  user_claim: sub
                  groups_claim: groups
                  policies: vault-admins
                  oidc_scopes:
                    - openid
                    - profile
                    - groups
                  ttl: 1h
        groups:
            - name: k8s-vault
              policies:
                - default
              type: external
            - name: vault-admins
              policies:
                - vault-admins
              type: external
        group-aliases:
            - name: k8s-vault
              mountpath: oidc/
              group: k8s-vault
            - name: vault-admins
              mountpath: oidc/
              group: vault-admins
        secrets:
            - path: blue/
              type: kv
              options:
                version: 2
            - path: green/
              type: kv
              options:
                version: 2
    vaultEnvsConfig:
        - name: VAULT_LOG_LEVEL
          value: debug
```
