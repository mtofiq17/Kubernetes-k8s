# Creating a Kubernetes User with Specific Permissions

This guide outlines the steps to create a new Kubernetes user with specific permissions. In this example, we'll create a user named `tofiq`.

## Step 1: Create the CSR for the User

Generate the private key and CSR:

```bash
openssl genrsa -out tofiq.key 2048
openssl req -new -key tofiq.key -out tofiq.csr -subj "/CN=tofiq/O=group1"


```

## Step 2: Base64 Encode the CSR

Encode the CSR in base64:
```bash
cat tofiq.csr | base64 | tr -d '\n'
```

## Step 3: Create the CertificateSigningRequest (CSR) YAML
Create a file named tofiq-csr.yaml with the following content. Replace BASE64_CSR with the base64 encoded string from the previous step.

```bash
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: tofiq
spec:
  request: BASE64_CSR
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
```

## Step 4: Apply the CSR and Approve it
Apply the CSR YAML file:

```bash
kubectl apply -f tofiq-csr.yaml
```

Approve the CSR:

```bash
kubectl certificate approve tofiq
```

## Step 5: Retrieve the Signed Certificate
Get the signed certificate and save it as tofiq.crt:

```bash
kubectl get csr tofiq -o jsonpath='{.status.certificate}' | base64 --decode > tofiq.crt
```

## Step 6: Create the Role and RoleBinding
Create a file named tofiq-role.yaml with the following content:

```bash
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: tofiq
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

## Apply the Role and RoleBinding:

```bash
kubectl apply -f tofiq-role.yaml
```

## Step 7: Configure kubectl for the New User
Configure the Kubernetes context for the new user tofiq:

```bash
kubectl config set-credentials tofiq --client-certificate=tofiq.crt --client-key=tofiq.key
kubectl config set-context tofiq-context --cluster=kubernetes --namespace=default --user=tofiq
kubectl config use-context tofiq-context
```

## Step 8: Verify the Setup

```bash
kubectl config get-contexts
kubectl get pods
```

## Step 9: Create a Deployment
Create a deployment JSON file:

```bash
kubectl create deployment nginx --image=nginx --dry-run=client -o json > deploy.json
kubectl run nginx --image=nginx --dry-run=client -o json
```

## Step 10: Service Account Creation and API Calls
Create a service account and bind it to the cluster admin role:

```bash
kubectl create serviceaccount sam --namespace default
kubectl create clusterrolebinding sam-clusteradmin-binding --clusterrole=cluster-admin --serviceaccount=default:sam
kubectl create token sam
```

Store the token and API server address in variables:

```bash
TOKEN=$(kubectl create token sam)
APISERVER=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')
```

List deployments using the service account token:

```bash
curl -X GET $APISERVER/apis/apps/v1/namespaces/default/deployments -H "Authorization: Bearer $TOKEN" -k
```

List pods using the service account token:

```bash
curl -X GET $APISERVER/api/v1/namespaces/default/pods -H "Authorization: Bearer $TOKEN" -k
```

