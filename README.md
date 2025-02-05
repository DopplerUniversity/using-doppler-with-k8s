# Using Doppler with Kubernetes

This is just a quick guide walking through how to quickly setup the Doppler Kubernetes Operator or External Secrets Operator with the Doppler provider. There are CRD examples demonstrating how to take advantage of a variety of features in both operators.

You can find the full documentation for these operators here:

- [Doppler Kubernetes Operator](https://docs.doppler.com/docs/kubernetes-operator)
- [External Secrets Operator](https://external-secrets.io/latest/guides/introduction/)

## Pre-requisites

### Note for Production

Make sure [encryption at rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/) is enabled in your Kubernetes cluster before using any secret operators.

### Create Doppler test project

Create a project in Doppler to import the sample secrets in `secrets.json` into. Name it whatever you like and then import the secrets by running:

```shell
doppler secrets upload -p YOUR_PROJECT -c dev secrets.json
```

### Install Docker

https://docs.docker.com/get-started/get-docker/

### Install Kind

https://kind.sigs.k8s.io/docs/user/quick-start/

```shell
brew install kind
```

### Install Helm

https://helm.sh/docs/intro/install/

```shell
brew install helm
```

### Install Kubectl

https://kubernetes.io/docs/tasks/tools/install-kubectl-macos/

```shell
brew install kubectl

# or

brew install kubernetes-cli
```

### Install k9s (optional)

https://k9scli.io/

```shell
brew install derailed/k9s/k9s
```

## Doppler Kubernetes Operator

### Install Operator

#### Create k8s cluster

```shell
kind create cluster
```

#### Add helm repo

```shell
helm repo add doppler https://helm.doppler.com
```

If you'd already added the helm repo previously, make sure you update it:

```shell
helm repo update
```

#### Install helm chart

```shell
helm install --generate-name doppler/doppler-kubernetes-operator
```

If you want to specify the name rather than generating it, use:

```shell
helm install YOUR_NAME_HERE doppler/doppler-kubernetes-operator
```

### Setup secret syncing

#### Create k8s secret for Doppler token

The operator itself doesn't require a Doppler token to install, but when you add a `DopplerSecret` CRD, you have to specify a token to use. For demo purposes, we're just going to use the local CLI token:

```shell
kubectl create secret generic doppler-token-secret \
  --namespace doppler-operator-system \
  --from-literal=serviceToken=$(doppler configure get token --plain)
```

**Note that this command is using your local `doppler` CLI token. This is fine for local testing purposes, but anywhere else you would likely want to use a [service account token](https://docs.doppler.com/docs/service-accounts).**

#### Apply DopplerSecret CRD

```yaml
apiVersion: secrets.doppler.com/v1alpha1
kind: DopplerSecret
metadata:
  name: dopplersecret-test
  namespace: doppler-operator-system
spec:
  tokenSecret:
    name: doppler-token-secret
    namespace: doppler-operator-system
  managedSecret:
    name: dopplersecret-test
    namespace: default
    type: Opaque
  project: k8s-demo
  config: dev
```

```shell
kubectl apply -f examples/doppler-secret.yml
```

##### Permissions

A `DopplerSecret` CRD created in the operator namespace can create and reference secrets outside of that namespace. However, any CRD created in another namespace is restricted to referencing secrets within that namespace. This allows teams to create a k8s secret containing their own Doppler service account token and then `DopplerSecret` CRDs within their own namespace without needing outside assistance. They're restricted from using tokens or writing to secrets in other team's namespaces.

##### Adjusting Sync Interval

https://docs.doppler.com/docs/kubernetes-operator#adjusting-sync-interval

You can control the polling interval that the operator uses when syncing by specifying `resyncSeconds`. We recommend leaving this at the default, but there are scenarios where adjusting this may be necessary or desired.

```yaml
apiVersion: secrets.doppler.com/v1alpha1
kind: DopplerSecret
metadata:
  name: dopplersecret-test
  namespace: doppler-operator-system
spec:
  tokenSecret:
    name: doppler-token-secret
    namespace: doppler-operator-system
  managedSecret:
    name: dopplersecret-test
    namespace: default
    type: Opaque
  project: k8s-demo
  config: dev
  resyncSeconds: 120
```

##### Name Transformers

https://docs.doppler.com/docs/kubernetes-operator#name-transformers

This can be used to transform the secret names in the config you're syncing to other formats.

```yaml
apiVersion: secrets.doppler.com/v1alpha1
kind: DopplerSecret
metadata:
  name: dopplersecret-test
  namespace: doppler-operator-system
spec:
  tokenSecret:
    name: doppler-token-secret
    namespace: doppler-operator-system
  managedSecret:
    name: dopplersecret-test
    namespace: default
    type: Opaque
  project: k8s-demo
  config: dev
  nameTransformer: dotnet-env
```

##### Processors

https://docs.doppler.com/docs/kubernetes-operator#processors

Processors can be used to rename or base64 decode secrets.

```yaml
apiVersion: secrets.doppler.com/v1alpha1
kind: DopplerSecret
metadata:
  name: dopplersecret-test
  namespace: doppler-operator-system
spec:
  tokenSecret:
    name: doppler-token-secret
    namespace: doppler-operator-system
  managedSecret:
    name: doppler-test-secret
    namespace: default
    type: Opaque
  project: k8s-demo
  config: dev
  processors:
    TLS_CRT:
      type: plain
      asName: tls.crt
    TLS_KEY:
      type: plain
      asName: tls.key
```

##### Secret Subsets

https://docs.doppler.com/docs/kubernetes-operator#secret-subsets

You can have the operator only sync a specific list of secrets. Note that the `DOPPLER_*` secrets are currently still added automatically.

```yaml
apiVersion: secrets.doppler.com/v1alpha1
kind: DopplerSecret
metadata:
  name: dopplersecret-test
  namespace: doppler-operator-system
spec:
  tokenSecret:
    name: doppler-token-secret
    namespace: doppler-operator-system
  managedSecret:
    name: doppler-test-secret
    namespace: default
    type: Opaque
  project: k8s-demo
  config: dev
  secrets:
    - TLS_CRT
    - TLS_KEY
```

##### Download Format

https://docs.doppler.com/docs/kubernetes-operator#download-formats

You can have secrets synced into a single `DOPPLER_SECRETS_FILE` key in the specified secret. The contents of the key is a string with your secrets in the specified download format (e.g., JSON, .env, YAML).

```yaml
apiVersion: secrets.doppler.com/v1alpha1
kind: DopplerSecret
metadata:
  name: dotnet-webapp-appsettings
  namespace: doppler-operator-system
spec:
  tokenSecret:
    name: doppler-token-dotnet-webapp
    namespace: doppler-operator-system
  managedSecret:
    name: dotnet-webapp-appsettings
    namespace: default
  format: dotnet-json
```

You can then mount this as a file in a Deployment if you so desire:

```yaml
---
spec:
  containers:
    - name: dotnet-webapp
      volumeMounts:
        - name: doppler
          mountPath: /usr/src/app/secrets
          readOnly: true
  volumes:
    - name: doppler
      secret:
        secretName: dotnet-webapp-appsettings # Managed secret name
        optional: false
        items:
          - key: DOPPLER_SECRETS_FILE # Hard-coded by Operator when format specified
            path: appsettings.json # Name or path to file name appended to container mountPath
```

### Use secrets in Deployment

#### Create deployment

Uses `envFrom` to inject the Doppler secrets into the container as environment variables. You can see examples of injecting via volume or injecting individual secrets [here](https://docs.doppler.com/docs/kubernetes-operator#examples).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: doppler-test-deployment-envfrom
  annotations:
    secrets.doppler.com/reload: "true"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: doppler-test
  template:
    metadata:
      labels:
        app: doppler-test
    spec:
      containers:
        - name: doppler-test
          image: alpine
          command:
            - /bin/sh
            - -c
            # Print all mongo environment variables
            - apk add --no-cache tini > /dev/null 2>&1 &&
              echo "### This is a simple deployment running with these MONGO_* variables:" &&
              printenv | grep MONGO_ &&
              tini -s tail -f /dev/null
          imagePullPolicy: Always
          envFrom:
            - secretRef:
                name: dopplersecret-test # Kubernetes secret name
          resources:
            requests:
              memory: "250Mi"
              cpu: "250m"
            limits:
              memory: "500Mi"
              cpu: "500m"
```

```shell
kubectl apply -f examples/deployment.yml
```

#### Make Deployment auto-restart on secret change

The `secrets.doppler.com/reload` annotation on the deployment causes it to auto-restart when the Doppler operator detects a secret change (this check is done based on polling that's done every 60 seconds):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: doppler-test-deployment-envfrom
  annotations:
    secrets.doppler.com/reload: "true"
```

Note that this only works with `Deployment` resources. To auto-restart other resource types, consider using [Reloader](https://github.com/stakater/Reloader).

## External Secrets Operator

#### Create k8s cluster

```shell
kind create cluster
```

#### Add helm repo

```shell
helm repo add external-secrets https://charts.external-secrets.io
```

If you'd already added the helm repo previously, make sure you update it:

```shell
helm repo update
```

#### Install helm chart

```shell
helm install external-secrets external-secrets/external-secrets -n external-secrets --create-namespace
```

### Setup secret syncing

#### Create k8s secret for Doppler token

The operator doesn't require a Doppler token to install, but when you add a `SecretStore` CRD, you have to specify a Doppler token to use. For demo purposes, we're just going to use the local CLI token:

```shell
kubectl create secret generic doppler-token-auth-api --from-literal=dopplerToken=$(doppler configure get token --plain) -n external-secrets
```

**Note that this command is using your local `doppler` CLI token. This is fine for local testing purposes, but anywhere else you would likely want to use a [service account token](https://docs.doppler.com/docs/service-accounts).**

#### Apply SecretStore CRD

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: doppler-auth-api
  namespace: external-secrets
spec:
  provider:
    doppler:
      auth:
        secretRef:
          dopplerToken:
            name: doppler-token-auth-api
            key: dopplerToken
      project: k8s-demo
      config: dev
```

```shell
kubectl apply -f examples/eso-secretstore.yml
```

#### Apply ExternalSecret CRD

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: auth-api-single
  namespace: external-secrets
spec:
  secretStoreRef:
    kind: SecretStore
    name: doppler-auth-api

  target:
    name: auth-api-single

  data:
    # sync just the "MONGO_URL" secret and use the key "mongo-url" in k8s
    - secretKey: mongo-url
      remoteRef:
        key: MONGO_URL
```

```shell
kubectl apply -f examples/eso-externalsecret-single.yml
```

##### Rewriting a secret name

https://external-secrets.io/latest/guides/datafrom-rewrite/

You can use rewrites to remove prefixes or perform other operations.

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: auth-api-rewrite
  namespace: external-secrets
spec:
  secretStoreRef:
    kind: SecretStore
    name: doppler-auth-api

  target:
    name: auth-api-rewrite

  dataFrom:
    # limit to just secrets starting with "DEVOPS_"
    - find:
        path: DEVOPS_
      # rewrite the secret, removing the "DEVOPS_" prefix
      rewrite:
        - regexp:
            source: "DEVOPS_(.*)"
            target: "$1"
```

```shell
kubectl apply -f examples/eso-externalsecret-rewrite.yml
```

##### Handling PKCS12 Certificates

You can handle PKCS12 certificates using some specialized functions.

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: auth-api-pkcs12
  namespace: external-secrets
spec:
  secretStoreRef:
    kind: SecretStore
    name: doppler-auth-api

  target:
    name: auth-api-pkcs12
    template:
      type: kubernetes.io/tls
      data:
        tls.crt: "{{ .cert | pkcs12certPass .pass }}"
        tls.key: "{{ .cert | pkcs12keyPass .pass }}"

  data:
    # this is a pkcs12 archive that contains a cert and a private key
    - secretKey: cert
      remoteRef:
        key: PKCS12_CRT
        decodingStrategy: Base64
    - secretKey: pass
      remoteRef:
        key: PKCS12_PASS
```

```shell
kubectl apply -f examples/eso-externalsecret-pkcs12.yml
```
