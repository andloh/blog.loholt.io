---
title: "Encrypt your Kubernetes secrets with Sops, FluxCD and age"
subtitle: "Manage your secrets using GitOps and Sops"
date: 2022-05-15
tags:
  - Open Source
  - Secret Management
  - GitOps
  - Kubernetes
draft: false

featuredImage: "/images/sops-arch.png"
featuredImagePreview: "/images/sops-arch.png"

code:
  copy: true
  maxShownLines: 50

share:
  enable: true

comment:
  enable: true

---

Manage your secrets using **GitOps** and **Sops**
<!--more-->

## The problem

With everything moving to a Gitops based workflow, which is great btw, we need to have a place to _store_ our secrets together with our declarative infrastructure and application definitions. 

While k8s's default secret object is the preferred way to give containers our secrets, the object faces some challenges if you want to _gitops_ your secrets.

The most obvious one is that your value you give the secret key is not encrypted in any way, just encoded with base64. The main purpose with this is that your value should not be readable to the naked eye. If we where to push your encoded secret to e.g github, your secret would then be exposed.

To solve this issue, we need to have a way of encrypting our secrets before we push it to our git and a way to make our continuous delivery too decrypt and deploy av secret to our Kubernetes clusters. 

## Why sops? 

Today we have multiple options for handling our secrets a secure way when they are consumed by our applications. To list a few:

- [Mozilla Sops](https://github.com/mozilla/sops)
- [SealedSecrets](https://github.com/bitnami-labs/sealed-secrets)
- HashCorp Vault
- Cloud Providers's Vault/KMS with [CSI drivers](https://secrets-store-csi-driver.sigs.k8s.io/)

The first two options is solely to solve the GitOps problem

Myself have worked with all the providers above and `sops` comes out to be the most simples one. Due to the fact that it is so easy to use, configure and integrates very well with FluxCD.

Sealed Secrets from Bitnami is also a great tool, but it's not as practical to work with if you compare it to `sops`.

I will not go into detail with the Vault's in this post, stay tuned for a separate posts about working with Vault's in Kubernetes ;)

{{< admonition type=note title="Note" open=true >}}
`Sops` is actually not a tool made for Kubernetes. It's an editor of encrypted files that supports YAML, JSON, ENV, INI and BINARY formats. It also has many integrations that it can encrypt with.
{{< /admonition >}}

## Prerequisites

You need access to a Kubernetes cluster with FluxCD thats already syncing configuration from a git repository.

{{< admonition type=note title="Note" open=true >}}
ArgoCD does not have `sops` integration, yet. Hope they can come with trough with this soon as it would be nice to have the option to use it with Argo too. It's possible to use it with Argo, but it will require you to add all the dependencies to ArgoCD's container image. Which is a lot of hassle to maintain, due to the rapid development around this tool. 
{{< /admonition >}}

## Installation
Before we get all secure we need some tooling first. 

Some distributions and OS's have these binaries in their package manager so check that out before you begin to download the binaries.

If you are running a Linux distribution you can use this short script:

{{< admonition type=tip title="Tip" open=true >}}
Make sure you are using the latest version and that you have `/usr/local/bin` in your path. Otherwise add it or change the destination of the binaries
{{< /admonition >}}

{{< highlight Bash "linenos=table" >}}
#!/bin/bash

# Install age and age-keygen
AGE_VERSION=v1.0.0
wget https://github.com/FiloSottile/age/releases/download/$AGE_VERSION/age-$AGE_VERSION-linux-amd64.tar.gz -O - | tar -xz
chmod +x age/age* && sudo mv age/age* /usr/bin/.
rm -rf age

# Install sops
SOPS_VERSION=v3.7.3
wget https://github.com/mozilla/sops/releases/download/$SOPS_VERSION/sops-$SOPS_VERSION.linux.amd64 && chmod +x sops-$SOPS_VERSION.linux.amd64 && sudo mv sops-$SOPS_VERSION.linux.amd64 /usr/bin/sops
{{< / highlight >}}

## Why Age

FluxCD does support the following encryption tools
- GPG (GNU Privacy Guard)
- OpenPGP
- Age

It also supports encryption with various cloud providers vault's and KMS's.

We will choose `age` for this scenario, because its simple, yet secure and modern.


## Generating the private key

Most of this is taken from the official [flux documentation](https://fluxcd.io/docs/guides/mozilla-sops/)

```bash
age-keygen -o age.agekey
Public key: age1qg2av8nvrnfywxz5j0hxqnvl6a4df73lcq4f48xdagj9qvsutvhq47awhv
```

`age-keygen` will automatically output the public key for us, we need this for later.

```bash
cat age.agekey |
kubectl create secret generic sops-age \
--namespace=flux-system \
--from-file=age.agekey=/dev/stdin
```
This will generate our first keypair and it will create a Kubernetes secret with our private key so that flux can access it.

{{< admonition type=warning title="Warning" open=true >}}
Create a secret with the age private key, the key name must end with .agekey to be detected as an age key
{{< /admonition >}}

{{< admonition type=Abstract title="Important" open=true >}}
Remember to backup the `age.agekey` file to a secure secret store. If you need to recreate your clusters you need to have the private key available to be able to decrypt your secrets again.
{{< /admonition >}}

## Encrypt your secret object

Create your secret object:
```bash
k create secret generic mysecret -n myapp --from-literal=dbpassword=supersecret123 --dry-run=client -o yaml > mysecret.yaml
```
Now let's take a look at our secret:

`cat secret.yaml`
```yaml
apiVersion: v1
data:
  dbpassword: c3VwZXJzZWNyZXQxMjM=
kind: Secret
metadata:
  creationTimestamp: null
  name: mysecret
  namespace: myapp
```
Looks good. 

Now we get to the exciting part, actually encrypting our value:

```bash
sops --age=age1qg2av8nvrnfywxz5j0hxqnvl6a4df73lcq4f48xdagj9qvsutvhq47awhv --encrypt --encrypted-regex '^(data|dbpassword)$' --in-place mysecret.yaml
```
{{< admonition type=tip title="Tip" open=true >}}
Pay attention to the `--encrypted-regex` parameter. Flux does not support encrypting the whole file, because then flux can not determine the object name, apiVersion etc... Therefore we can only encrypt the value of your key, which is a perfect way to do it, because it will make the file more readable.
{{< /admonition >}}


Now lets look at our secret object:
```yaml
cat mysecret.yaml
apiVersion: v1
data:
    dbpassword: ENC[AES256_GCM,data:oNKIqI0jA4JjE24Y82JebXSuvHY=,iv:PguL7OGquRCGtnLvCK5TMGchCxWPPiIRI3kcJqY623g=,tag:demSCh7SqO/Y67mKifjKXA==,type:str]
kind: Secret
metadata:
    creationTimestamp: null
    name: mysecret
    namespace: myapp
sops:
    kms: []
    gcp_kms: []
    azure_kv: []
    hc_vault: []
    age:
        - recipient: age1qg2av8nvrnfywxz5j0hxqnvl6a4df73lcq4f48xdagj9qvsutvhq47awhv
          enc: |
            -----BEGIN AGE ENCRYPTED FILE-----
            YWdlLWVuY3J5cHRpb24ub3JnL3YxCi0+IFgyNTUxOSBmNjB3ejVLUVNMLzErSm14
            YnhhamtDWGJMcDUrQTZOSVM2SzIvRUZiRmdBCjhDNHlKL3BiMEZZanNFZGVzckdx
            SUpUeU9SZzdxQXBnWGZtYmFYS1lUb1kKLS0tIHgyOXM2NGxLc1ZtUEhFRzdRN2VT
            K3E3dmpzazc2bFF3ekR6VGJsNHZVQ0UK4+MFB2HU1iYFpo15Zgid55HW9k9Wueko
            Y8D32VBFD8Tevx+ovXaQoHd6+ddFYLPSKTAdv6LMFt5ZV5z6VLAOjw==
            -----END AGE ENCRYPTED FILE-----
    mac: ENC[AES256_GCM,data:zO7B0Ui8HKaLXXVU3EiL+K1a2AyXqAnpWiZVL91x7GWWHd/H5Ce/I0aF4pPkaxfoqLJecjhEp2SD57HyoyXRwKcyAWWdomEOT150yk5tx2YikKPGz/ed6QiZx+YzmJjzsHuKDicyqKP/ZgFzAQXxUpa58CJTQN+9E/NBuf1iePQ=,iv:dmUV4tN8y1zgvOchkfiyNNJ9ERF+N7XhX3Lj+KJkiC8=,tag:gFKiOfdqsxmE1eOz3OohJw==,type:str]
    pgp: []
    encrypted_regex: ^(data|dbpassword)$
    version: 3.7.2
```

Looks scary right? Do not worry, this is how it should look. This file is now safe to push to git and share with others.

## Deploy your secret

To deploy our secret we need to go trough flux, thats also means that we will need to push it to git. You also need to specify in the flux `Kustomization` which encryption method flux shall use.

Update or create your `Kustomization` with the `spec.decryption` parameter. 
Here we specify our decryption provider and the secret with our private `age` key.

```yaml 
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: myapp
  namespace: flux-system
spec:
  interval: 1m0s
  path: ./test/myapp
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  decryption:
    provider: sops
    secretRef:
      name: sops-age
```

With this configuration in place, flux will decrypt and deploy your secret to your targeted Kubernetes cluster.
To verify, make sure your `Kustomization` has synced and take a look at your secret inside Kubernetes

## Decrypt and edit your secret on the fly for debugging

This is one of the main reasons i like `sops` better than `Sealed Secrets`. You can decrypt yor secret locally with just doing: 

```bash
sops mysecret.yaml
```

This helps a lot with debugging etc. Anyone with access to the private key can decrypt the secret object you have stored in git.

## Encrypt a secret object with multiple keys

The easiest way to encrypt a secret object with multiple keys is just to add another `--encrypted-regex` parameter when you're encrypting.

## Age and Sops configuration

### Age
Your private keys must be stored in `~/.config/sops/age/keys.txt` when working with `sops`, you can also specify your own path with `export SOPS_AGE_KEY_FILE=/my/secret/path`. You can have multiple keypairs stored in that single file. `Sops` will know which private key to use when you run `sops` against a encrypted file, because the public key is annotated in all encrypted secret object's. 

### Sops

When decrypting you really don't want to find which public key you want to decrypt with every time, if you manage multiple pairs, which is recommended. To make this easier, create a `sops` config file: 

This file should be created in the root of you git directory.

```yml
creation_rules:
    - path_regex: ^test\/.*\.yaml$
      age: publickey
    - path_regex: ^qa\/.*\.yaml$
      age: publickey
    - path_regex: ^prod\/.*\.yaml$
      age: publickey
```
This example will select the targeted keypair based on which folder you are standing in when running `sops --encrypt`

## Useful commands and tips

- `cat` your secret
  ```bash
  sops -d mysecret.yaml
  ```
- If you want do decrypt your secret _and_ test/verify it against the cluster at the same time:
  ```bash
  sops -d mysecret.yaml | k diff -f -
  ```
- You can edit your secret after it has been encrypted, at least the _non encrypted_ part of it. To do this you need to open the file with `sops` not a normal text editor e.g `vim`.
  ```bash
  sops mysecret.yaml
  # Edit your file and save
  ```
- Send encrypted file to new file
  ```bash
  sops -e test.yaml > encryptedTest.yaml
  ```

## Verdict

All in all, a very good, secure and simple to use tool for _storing_ your secrets in git.

### The good
- Simple to use and install
- Secure
- Very easy to work with and maintain
- Good Flux integration
- Perfect for if you need to recover your secrets when you are recreating your clusters, which is a common disaster recovery strategy when working with Kubernetes and stateless platforms
- Supports the GitOps model

### The _not_ so good
- The private keys are still stored as base64 encoded Kubernetes secrets. You will need to lock this down with RBAC
  - An alternative integration with a Vault will solve this to some extend.
- No support _yet_ for usage with ArgoCD
