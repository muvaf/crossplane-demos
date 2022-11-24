# Managed Resources Demo

> This is the demo I did for Cloud Native TR community on 24th November 2022.
> You can find the recording of the demo [here](https://www.youtube.com/watch?v=Tw2R11NfscU).

Managed Resource (MR) is Crossplane’s representation of a resource in an
external system - most commonly a cloud provider. Managed Resources are
opinionated, Crossplane Resource Model (XRM) compliant Kubernetes Custom
Resources that are installed by a Crossplane provider. Read more about
it [here](https://crossplane.io/docs/v1.10/concepts/managed-resources.html)

## Pre-requisites

* `kubectl`: https://kubernetes.io/docs/tasks/tools/
* `kind`: https://kind.sigs.k8s.io/docs/user/quick-start/
* `helm`: https://helm.sh/docs/intro/install/
* A Google Cloud Platform account
  * `gcloud`: https://cloud.google.com/sdk/docs/install

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

Install the GCP provider.
```yaml
# cat <<EOF | kubectl apply -f - (then Shift+Enter, paste the content, Shift+Enter, EOF, Enter)
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-gcp
spec:
  package: xpkg.upbound.io/upbound/provider-gcp:v0.19.0
```

We need to create a GCP service account that can provision a GCP bucket.

Export the name of your GCP project ID.
```bash
# This is project ID which may be different from project name.
export GCP_PROJECT_ID="crossplane-playground"
```

Create a permission file called `permissions.yaml` with the following content.
```yaml
# cat <<EOF > permissions.yaml (then Shift+Enter, paste the content, Shift+Enter, EOF, Enter)
title: "Bucket"
description: "Permissions necessary to create a bucket."
stage: "ALPHA"
includedPermissions:
- storage.buckets.create
- storage.buckets.delete
- storage.buckets.get
- storage.buckets.getIamPolicy
- storage.buckets.setIamPolicy
- storage.buckets.update
- storage.objects.list
- storage.objects.delete
```

Create a role with these permissions.
```bash
gcloud iam roles create cloud_native_tr_bucket --project="${GCP_PROJECT_ID}" \
  --file=permissions.yaml
```

Create a Service Account.
```bash
gcloud iam service-accounts create cloud-native-tr-sa \
  --description="To be used in Cloud Native TR event" \
  --display-name="Cloud Native TR SA"
```

Bind the role to our Service Account.
```bash
gcloud projects add-iam-policy-binding "${GCP_PROJECT_ID}" \
  --member="serviceAccount:cloud-native-tr-sa@${GCP_PROJECT_ID}.iam.gserviceaccount.com" \
  --role="projects/${GCP_PROJECT_ID}/roles/cloud_native_tr_bucket"
```

Get the key for our Service Account.
```bash
gcloud iam service-accounts keys create /tmp/creds.json \
  --iam-account="cloud-native-tr-sa@${GCP_PROJECT_ID}.iam.gserviceaccount.com"
```

Create a Kubernetes `Secret` containing the key.
```bash
kubectl create secret generic gcp-creds \
  --from-file=creds=/tmp/creds.json
```

Finally, create a default `ProviderConfig` for our GCP provider to know about
this `Secret`.
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
      namespace: default
      name: gcp-creds
      key: creds
```

## Create a Bucket

Create a `Bucket` resource.
```yaml
apiVersion: storage.gcp.upbound.io/v1beta1
kind: Bucket
metadata:
  name: cloud-native-tr
spec:
  forProvider:
    location: US
    storageClass: MULTI_REGIONAL
```

Wait for it to become ready.
```bash
kubectl get bucket cloud-native-tr -o yaml -w
```

Go to GCP console to check!

## Cleanup

Delete the `Bucket` resource.
```bash
kubectl delete bucket cloud-native-tr
```

Delete the GCP service account, the role and the key we created in GCP. 
```bash
gcloud iam service-accounts keys delete
gcloud iam service-accounts delete cloud-native-tr-sa
gcloud iam roles delete cloud_native_tr_bucket
```