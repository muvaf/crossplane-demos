# Composition Demo

I did an extensive tutorial on how Crossplane can be used with Backstage, ArgoCD
and GitHub to build your own Heroku on 27th October 2022 at KubeCon NA 2022. 

You can find all the material of that tutorial here: https://github.com/muvaf/cloud-native-heroku

## Installation

Create a cluster.
```bash
kind create cluster --wait 5m
```

Install Crossplane.
```bash
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update

helm install crossplane --namespace crossplane-system --create-namespace --wait crossplane-stable/crossplane
```

We are going to provision cloud infrastructure from Google Cloud. We need to
install a Crossplane provider to do that.

```yaml
# cat <<EOF | kubectl apply -f - (then Shift+Enter, paste the content, Shift+Enter, EOF, Enter)
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-gcp
spec:
  package: xpkg.upbound.io/upbound/provider-gcp:v0.19.0
```

Wait till the provider pod comes up.
```
kubectl get pods -w
```

Next, we need to add our cloud credentials for GCP provider to use. The
following is the minimum permission definition that is required to create an
encrypted bucket. We can create a new GCP Role with that file to be assigned
to a GCP Service Account to be used by Crossplane.

Let's write this to a file named `permissions.yaml`.
```yaml
# cat <<EOF > permissions.yaml (then Shift+Enter, paste the content, Shift+Enter, EOF, Enter)
title: "Encrypted Bucket"
description: "Permissions necessary to create an encrypted bucket."
stage: "ALPHA"
includedPermissions:
- storage.buckets.create
- storage.buckets.createTagBinding
- storage.buckets.delete
- storage.buckets.deleteTagBinding
- storage.buckets.get
- storage.buckets.getIamPolicy
- storage.buckets.setIamPolicy
- storage.buckets.update
- storage.objects.list
- storage.objects.delete
- iam.serviceAccounts.create
- iam.serviceAccounts.delete
- iam.serviceAccounts.get
- iam.serviceAccounts.update
- iam.serviceAccountKeys.create
- iam.serviceAccountKeys.delete
- iam.serviceAccountKeys.get
- cloudkms.keyRings.create
- cloudkms.keyRings.createTagBinding
- cloudkms.keyRings.deleteTagBinding
- cloudkms.keyRings.get
- cloudkms.cryptoKeys.create
- cloudkms.cryptoKeys.get
- cloudkms.cryptoKeys.getIamPolicy
- cloudkms.cryptoKeys.setIamPolicy
- cloudkms.cryptoKeys.update
- cloudkms.cryptoKeyVersions.list
- cloudkms.cryptoKeyVersions.destroy
```

Create a new GCP Role with the above permissions:
```bash
export GCP_PROJECT_ID="my-project-name"
gcloud iam roles create encrypted_bucket --project="${GCP_PROJECT_ID}" \
  --file=permissions.yaml
```

Create a GCP Service Account and assign this new role:
```bash
gcloud iam service-accounts create cloud-native-heroku-sa \
  --description="To be used in Cloud Cative Heroku" \
  --display-name="Cloud Native Heroku SA"
```
```bash
# Assumes GCP_PROJECT_ID is available
gcloud projects add-iam-policy-binding "${GCP_PROJECT_ID}" \
  --member="serviceAccount:cloud-native-heroku-sa@${GCP_PROJECT_ID}.iam.gserviceaccount.com" \
  --role="projects/${GCP_PROJECT_ID}/roles/encrypted_bucket"
```

Our GCP Service Account is ready. Let's create a private key that we can give to
our GCP Provider to use.
```bash
# Assumes GCP_PROJECT_ID is available
gcloud iam service-accounts keys create /tmp/creds.json \
  --iam-account="cloud-native-heroku-sa@${GCP_PROJECT_ID}.iam.gserviceaccount.com"
```

Create a Kubernetes `Secret` with the credentials.
```bash
kubectl -n heroku create secret generic gcp-creds \
  --from-file=creds=/tmp/creds.json
```

Use it in the configuration of the provider.
```yaml
# cat <<EOF | kubectl apply -f - (then Shift+Enter, paste the content, Shift+Enter, EOF, Enter)
apiVersion: gcp.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  projectID: crossplane-playground
  credentials:
    source: Secret
    secretRef:
      namespace: heroku
      name: gcp-creds
      key: creds
```

## Create a Bucket API

Create a `CompositeResourceDefinition` to define a new API.

```yaml
# cat <<EOF | kubectl apply -f - (then Shift+Enter, paste the content, Shift+Enter, EOF, Enter)
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xbuckets.kubecon.org
spec:
  group: kubecon.org
  names:
    kind: XBucket
    plural: xbuckets
  claimNames:
    kind: Bucket
    plural: buckets
  connectionSecretKeys:
    - bucketName
    - googleCredentialsJSON
  versions:
  - name: v1alpha1
    served: true
    referenceable: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              location:
                type: string
                description: |
                  The location for Bucket and the KMS key to be created in.
                  See https://cloudgoogle.com/kms/docs/locations for available locations.
            required:
            - location
          status:
            type: object
            properties:
              serviceAccountEmail:
                type: string
              kmsKeyId:
                type: string
```

## Create an Implementation for Our API

The first implementation, i.e. controller logic, of that API is an encrypted
bucket.

```yaml
# cat <<EOF | kubectl apply -f - (then Shift+Enter, paste the content, Shift+Enter, EOF, Enter)
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: encrypted.xbuckets.kubecon.org
  labels:
    provider: gcp
spec:
  writeConnectionSecretsToNamespace: heroku
  compositeTypeRef:
    apiVersion: kubecon.org/v1alpha1
    kind: XBucket
  resources:
    - name: bucket
      base:
        apiVersion: storage.gcp.upbound.io/v1beta1
        kind: Bucket
        spec:
          forProvider:
            storageClass: MULTI_REGIONAL
            forceDestroy: true
      patches:
        - fromFieldPath: status.kmsKeyId
          toFieldPath: encryption.defaultKmsKeyName
          policy:
            fromFieldPath: Required
        - fromFieldPath: spec.location
          toFieldPath: spec.forProvider.location
      connectionDetails:
        - type: FromFieldPath
          name: bucketName
          fromFieldPath: metadata.annotations[crossplane.io/external-name]
    - name: serviceaccount
      base:
        apiVersion: cloudplatform.gcp.upbound.io/v1beta1
        kind: ServiceAccount
        spec:
          forProvider: {}
      patches:
        - type: CombineFromComposite
          toFieldPath: spec.forProvider.description
          policy:
            fromFieldPath: Required
          combine:
            variables:
              - fromFieldPath: spec.claimRef.namespace
              - fromFieldPath: spec.claimRef.name
            strategy: string
            string:
              fmt: "%s/%s"
        - type: ToCompositeFieldPath
          fromFieldPath: status.atProvider.email
          toFieldPath: status.serviceAccountEmail
          policy:
            fromFieldPath: Required
    - name: serviceaccountkey
      base:
        apiVersion: cloudplatform.gcp.upbound.io/v1beta1
        kind: ServiceAccountKey
        spec:
          forProvider:
            publicKeyType: TYPE_X509_PEM_FILE
            serviceAccountIdSelector:
              matchControllerRef: true
          writeConnectionSecretToRef:
            namespace: heroku
      patches:
        - type: CombineFromComposite
          combine:
            variables:
              - fromFieldPath: metadata.uid
            strategy: string
            string:
              fmt: "%s-serviceaccountkey"
          toFieldPath: spec.writeConnectionSecretToRef.name
          policy:
            fromFieldPath: Required
      connectionDetails:
        - name: googleCredentialsJSON
          fromConnectionSecretKey: private_key
    - name: add-sa-to-bucket
      base:
        apiVersion: storage.gcp.upbound.io/v1beta1
        kind: BucketIAMMember
        spec:
          forProvider:
            bucketSelector:
              matchControllerRef: true
            role: roles/storage.objectAdmin
      patches:
        - type: CombineFromComposite
          combine:
            variables:
              - fromFieldPath: status.serviceAccountEmail
            strategy: string
            string:
              fmt: "serviceAccount:%s"
          toFieldPath: spec.forProvider.member
          policy:
            fromFieldPath: Required
    - name: keyring
      base:
        apiVersion: kms.gcp.upbound.io/v1beta1
        kind: KeyRing
        spec:
          forProvider: {}
      patches:
        - type: FromCompositeFieldPath
          fromFieldPath: spec.location
          toFieldPath: spec.forProvider.location
    - name: cryptokey
      base:
        apiVersion: kms.gcp.upbound.io/v1beta1
        kind: CryptoKey
        spec:
          forProvider:
            destroyScheduledDuration: 86400s
            keyRingSelector:
              matchControllerRef: true
            rotationPeriod: 100000s
      patches:
        - type: ToCompositeFieldPath
          fromFieldPath: status.atProvider.id
          toFieldPath: status.kmsKeyId
          policy:
            fromFieldPath: Required
    - name: add-sa-to-cryptokey
      base:
        apiVersion: kms.gcp.upbound.io/v1beta1
        kind: CryptoKeyIAMMember
        spec:
          forProvider:
            cryptoKeyIdSelector:
              matchControllerRef: true
            role: roles/cloudkms.cryptoKeyEncrypterDecrypter
      patches:
        - type: CombineFromComposite
          combine:
            variables:
              - fromFieldPath: status.serviceAccountEmail
            strategy: string
            string:
              fmt: "serviceAccount:%s"
          toFieldPath: spec.forProvider.member
          policy:
            fromFieldPath: Required
```

## Create a Bucket Using Our Own API

Create an instance of our new API definition.
```yaml
# cat <<EOF | kubectl apply -f - (then Shift+Enter, paste the content, Shift+Enter, EOF, Enter)
apiVersion: kubecon.org/v1alpha1
kind: Bucket
metadata:
  name: hello-bucket
  namespace: default
spec:
  location: us
  writeConnectionSecretToRef:
    name: bucket-creds
```

Check how the resources are getting created.
```bash
kubectl get managed
```

Check the status of the bucket and the connection secret.
```bash
kubectl -n default get buckets --watch
```

## Cleanup

Delete the bucket.
```bash
kubectl delete bucket.kubecon.org hello-bucket
```

Delete the service account, the role and the service account key you created
during the installation.
```bash 
gcloud iam service-accounts keys delete
gcloud iam service-accounts delete cloud-native-tr-sa
gcloud iam roles delete cloud_native_tr_bucket
```