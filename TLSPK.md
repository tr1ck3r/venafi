# Hands On with cert-manager and Venafi TLS Protect for Kubernetes

### Prerequisites
- A healthy Kubernetes cluster ([EKS](https://aws.amazon.com/eks/), [AKS](https://azure.microsoft.com/en-us/products/kubernetes-service), [GKE](https://cloud.google.com/kubernetes-engine), [OpenShift](https://www.redhat.com/en/technologies/cloud-computing/openshift), [Kind](https://kind.sigs.k8s.io/), [K3s](https://k3s.io/), etc.) with access to pull images from registries on the Internet
- `kubectl` [installed](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/) with the ability to fully manage the cluster
- `helm` [installed](https://helm.sh/docs/intro/install/)
- `helm diff` plugin [installed](https://github.com/databus23/helm-diff?tab=readme-ov-file#install)

### 1. Install cert-manager open source using Helm
```sh
helm repo add jetstack https://charts.jetstack.io --force-update
```
```sh
helm repo update
```
```sh
helm install cert-manager \
    jetstack/cert-manager \
    --namespace venafi \
    --create-namespace \
    --version v1.14.4 \
    --set installCRDs=true
```

> [!NOTE]
> The "venafi" namespace is the only difference to what is documented on https://cert-manager.io/docs/installation/helm/#installing-with-helm.

### 2. Create a self-signed issuer for cert-manager
```sh
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: self-signed-issuer
spec:
  selfSigned: {}
EOF
```
```sh
kubectl get clusterissuers
```

### 3. Create sample ingresses that use self-signed issuer for their certificates
Running the following will create a namespace called "self-signed" and within it create a deployment, service, and 6 ingresses named after earth/water zodiac signs.
```sh
kubectl apply -f https://github.com/tr1ck3r/venafi/releases/latest/download/sample-ingresses-self-signed.yaml
```
> [!IMPORTANT]  
> These samples expect the certificate issuer to be a ClusterIssuer called "self-signed-issuer".
```sh
kubectl get ingresses -n self-signed
```

### 4. Install Venafi Kubernetes agent to make certificates visible in the Venafi Control Plane
:cloud: This task is best experienced using the Venafi Control Plane web interface (navigate to *Installations > Kubernetes Clusters* and click "Connect").  It will dynamically generate the `venctl` command with the cluster name you enter and the API key for the logged in user.
```sh
curl -sSfL https://dl.venafi.cloud/venctl/latest/installer.sh | bash
```
```sh
venctl installation cluster connect --name "hands-on" --api-key xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```
Shortly after the agent has been installed, certificates in the cluster (including the 6 for the ingresses) will start to appear in the Venafi Control Plane inventory (navigate to *Inventory > Certificates*).

### 5. Install additional Venafi TLS Protect for Kubernetes components
:cloud: Use the Venafi Control Plane web interface to create a service account for accessing the Venafi Registry (navigate to *Settings > Service Accounts*, click "New" and select "Venafi Registry").  Make sure to add all 3 of the available scopes.  Choose the "[Docker Config Format](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/#registry-secret-existing-credentials)" and, as instructed, add the content as to a file called `venafi_registry_docker_config.json`.  The run the following commands (using the Client ID of the just created service account for the `VENAFI_KUBERNETES_AGENT_CLIENT_ID`):
```sh
kubectl create secret generic venafi-image-pull-secret -n venafi \
    --type=kubernetes.io/dockerconfigjson \
    --from-file=.dockerconfigjson=./venafi_registry_docker_config.json
```
```sh
kubectl patch serviceaccount  default -n venafi \
    -p '{"imagePullSecrets": [{"name": "venafi-image-pull-secret"}]}'
```
```sh
export VENAFI_KUBERNETES_AGENT_CLIENT_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```
```sh
venctl c k apply --cert-manager \
    --venafi-kubernetes-agent \
    --venafi-enhanced-issuer \
    --approver-policy-enterprise \
    --log-level debug
```
> [!CAUTION]
> The current version of `venctl` will remove any already installed component if you don't include it on the command line, so don't omit `--venafi-kubernetes-agent`.  Including `--cert-manager` will upgrade it from open source to enterprise version.

> [!TIP]
> The above command will likely return an error but the Venafi Kubernetes Agent should be functional even if the Helm deployment failed.
```sh
helm list -n venafi && kubectl get pods -n venafi
```

### 6. Create Issuing Template and Application in Venafi Control Plane
:cloud: Use the Venafi Control Plane web interface to create an Issuing Template (navigate to *Policies > Issuing Templates* and click "New").  We'll see how the constraints you configure for the Issuing Template can be applied in 2 different ways (i.e., _issuance_ and _policy enforcement_).  Configure the Issuing Template to allow "EC P256" keys, and CNs and DNS SANs that include "fire" or "water"using the following regex:
```regex
(.*\.)?(fire|water)\.zodiac\.example
```

Create an Application (navigate to *Applications* and click "New") and assign to it the Issuing Template you just created.  You can use the "API Alias" field to specify a shorthand.

### 7. Create a TLS Protect Cloud issuer for the Venafi Enhanced Issuer
Substitute a valid API key for the placeholder and replace "your-app-name" and "issuing-tmpl-alias" with the actual name of your Application its alias for the Issuing Template to use for _issuance_, respectively.
```sh
kubectl create secret generic venafi-api-key -n venafi \
    --from-literal=api-key=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```
```sh
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: get-tlspc-api-key-role
  namespace: venafi
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]
    resourceNames: ["venafi-api-key"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: get-tlspc-api-key-rolebinding
  namespace: venafi
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: get-tlspc-api-key-role
subjects:
- kind: ServiceAccount
  name: venafi-connection
  namespace: venafi
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cert-manager-controller-approve:venafi-enhanced-issuer
rules:
  - apiGroups: ["cert-manager.io"]
    resources: ["signers"]
    verbs: ["approve"]
    resourceNames: ["venafiissuers.jetstack.io/*", "venaficlusterissuers.jetstack.io/*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cert-manager-controller-approve:venafi-enhanced-issuer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cert-manager-controller-approve:venafi-enhanced-issuer
subjects:
  - name: cert-manager
    namespace: venafi
    kind: ServiceAccount
---
apiVersion: jetstack.io/v1alpha1
kind: VenafiConnection
metadata:
  name: venafi-connection
  namespace: venafi
spec:
  vaas:
    url: https://api.venafi.cloud/
    apiKey:
      - secret:
          name: venafi-api-key
          fields: [ "api-key" ] 
---
apiVersion: jetstack.io/v1alpha1
kind: VenafiClusterIssuer
metadata:
  name: venafi-tlspc-issuer
spec:
  venafiConnectionName: venafi-connection
  zone: "your-app-name\\\\issuing-tmpl-alias"
EOF
```
```sh
kubectl get VenafiClusterIssuer
```

### 8. Add an Approver Policy for certificate requests
Replace "your-app-name" and "issuing-tmpl-alias" with the actual name of your Application and its alias for the Issuing Template to user for _policy enforcement_, respectively.
```sh
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cert-manager-policy:my-venafi-policy
rules:
  - apiGroups: ["policy.cert-manager.io"]
    resources: ["certificaterequestpolicies"]
    verbs: ["use"]
    resourceNames: ["my-venafi-policy"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cert-manager-policy:my-venafi-policy
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cert-manager-policy:my-venafi-policy
subjects:
- kind: Group
  name: system:authenticated
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: policy.cert-manager.io/v1alpha1
kind: CertificateRequestPolicy
metadata:
  name: my-venafi-policy
spec:
  allowed:
    commonName:
      value: "*.zodiac.example"
    dnsNames:
      values: ["*.zodiac.example"]
    usages:
      - "digital signature"
      - "key encipherment"
  plugins:
    venafi:
      values:
        venafiConnectionName: venafi-connection
        zone: "your-app-name\\\\issuing-tmpl-alias"
  selector:
    issuerRef: {}
EOF
```

### 9. Create sample ingresses that use Venafi TLS Protect Cloud issuer for their certificates
Running the following will create a namespace called "tls-protect-cloud" and within it create a deployment, service, and 6 ingresses named after air/fire zodiac signs.
```sh
kubectl apply -f https://github.com/tr1ck3r/venafi/releases/latest/download/sample-ingresses-tls-protect-cloud.yaml
```
> [!IMPORTANT]  
> These samples expect the certificate issuer to be a VenafiClusterIssuer called "venafi-tlspc-issuer".
```sh
kubectl get ingresses -n tls-protect-cloud
```
In this case, because the **Venafi Enhanced Issuer** is being used to fulfill certificate requests instead of the self-signed issuer, the ingress certificates appear in the Venafi Control Plane inventory as a result of issuance rather than having to be discovered by the Venafi Kubernetes Agent.

### 10. Review the ingress certificates
```sh
kubectl get certificates -A
```
Notice that certificates were issued for the "fire" ingresses but not for the "air" ones.  Let's see what happens now if we recreate the "water" and "earth" ingresses that are using the self-signed issuer...
```sh
kubectl delete ingress,secrets --all -n self-signed
```
```sh
kubectl apply -f https://github.com/tr1ck3r/tlspk-demo/releases/latest/download/sample-ingresses-self-signed.yaml
```
```sh
kubectl get certificates -A
```
Notice that the Approver Policy is governing the non-Venafi self-signed issuer, too.  Certificates were only issued for the "water" ingresses, not the "earth".