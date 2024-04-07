# Hands On with Venafi Firefly running in Kubernetes

### Prerequisites
- a healthy Kubernetes cluster with access to pull images from the Internet and that can be managed using `kubectl`
- an OIDC Provider with a public OIDC Discovery endpoint that is capable of issuing JWTs that include custom claims
- a valid Venafi Control Plane (VCP) tenant with Firefly entitlements
- a VCP **Service Account** for Firefly ("Distributed Issuer" scope)
- a **Firefly SubCA Provider** to enable issuance of SubCA certificates for Firefly instances using Venafi Zero Touch PKI, Venafi TLS Protect Datacenter, Microsoft ADCS, or VCP Built-In (test) CA
- at least one **Firefly Policy** that will constrain the certificates _issued by_ Firefly
- a **Firefly Configuration** associated with the Service Account, SubCA Provider, and Policy

### 0. Configure OIDC Provider to issue JWTs with Firefly custom claims
Firefly requires JWTs presented by API clients to include `venafi-firefly.configuration` and either `venaf-firefly.allowedPolicies` or `venafi-firefly.allowAllPolicies` claims.  For evaluation purposes, [jwt-this](https://github.com/tr1ck3r/jwt-this) can be used.

### 1. Create namespace for Firefly
We'll run Firefly in a dedidated namespace.
```sh
kubectl create namespace firefly
```

### 2. Create a secret for the private key of the Firefly service account in Venafi Control Plane
Save the private key associated with the service account you created for Firefly in VCP to a file called `private-key.pem` the run the following command:
```sh
kubectl create secret generic firefly-svc-acct-key -n firefly \
  --from-file=svc-acct.key=./private-key.pem
```

### 3. Create RBAC and CRD for Firefly and cert-manager interoperability
```sh
kubectl apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: k8s-svc-acct
  namespace: firefly
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: firefly
rules:
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create"]
  - apiGroups: ["cert-manager.io"]
    resources: ["certificaterequests"]
    verbs: ["get", "watch", "list"]
  - apiGroups: ["cert-manager.io"]
    resources: ["certificaterequests/status"]
    verbs: ["update"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: firefly
subjects:
- kind: ServiceAccount
  name: k8s-svc-acct
  namespace: firefly
roleRef:
  kind: ClusterRole
  name: firefly
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: firefly:leader-election
  namespace: venafi
rules:
- apiGroups: ["coordination.k8s.io"]
  resources: ["leases"]
  verbs: ["get", "watch", "list", "update", "create"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: firefly:leader-election
  namespace: venafi
subjects:
- kind: ServiceAccount
  name: k8s-svc-acct
  namespace: firefly
roleRef:
  kind: Role
  name: firefly:leader-election
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: clusterissuers.firefly.venafi.com
spec:
  group: firefly.venafi.com
  names:
    kind: ClusterIssuer
    listKind: ClusterIssuerList
    plural: clusterissuers
    singular: clusterissuer
  scope: Cluster
  versions:
  - name: v1alpha1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties: null
EOF
```

### 4. Create ConfigMap that defines Firefly bootstrap configuration
Replace `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` with the Client ID of the service account you created for Firefly in VCP (i.e., the same service account as the `private-key.pem` file you created earlier).
```sh
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: firefly-config
  namespace: firefly
immutable: true
data:
  config.yaml: |
    bootstrap:
      vaas:
        auth:
          privateKeyFile: /etc/firefly/auth/svc-acct.key
          clientID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
        csr:
          instanceNaming: "{HOSTNAME}"
    server:
      grpc:
        port: 8081
        tls:
          dnsNames:
            - "localhost"
      graphql:
        port: 8123
        playground: true
        tls:
          dnsNames:
            - "localhost"
      rest:
        port: 8281
        tls:
          dnsNames:
            - "localhost"
    controller:
      leaderElectionNamespace: venafi
      cert-manager:
        groupName: firefly.venafi.com
        checkApproval: false
EOF
```
>[!NOTE]
> "localhost" should be replaced with the FQDN at which Firefly clients will access its API endpoints.

### 5. Create Deployment to run Firefly in cluster and Service to expose client API endpoints
```sh
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: firefly-deployment
  namespace: firefly
  labels:
    app: firefly
spec:
  replicas: 5
  selector:
    matchLabels:
      app: firefly
  template:
    metadata:
      name: firefly-issuer
      namespace: firefly
      labels:
        app: firefly
    spec:
      serviceAccountName: k8s-svc-acct
      volumes:
        - name: firefly-config-volume
          configMap:
            name: firefly-config
        - name: firefly-secret-volume
          secret:
            secretName: firefly-svc-acct-key
      containers:
        - name: firefly-container
          image: registry.venafi.cloud/public/venafi-images/firefly:latest
          imagePullPolicy: IfNotPresent
          command: ["firefly"]
          args: ["run", "-c", "/etc/firefly/config.yaml"]
          ports:
            - containerPort: 8081
              name: grpc-server
            - containerPort: 8123
              name: graphql-server
            - containerPort: 8281
              name: rest-server
          volumeMounts:
            - name: firefly-config-volume
              mountPath: /etc/firefly
              readOnly: true
            - name: firefly-secret-volume
              mountPath: /etc/firefly/auth
              readOnly: true
          env:
            - name: ACCEPT_TERMS
              value: "Y"
          securityContext:
            runAsNonRoot: true
            runAsUser: 1001
            capabilities:
              add:
                - IPC_LOCK
          readinessProbe:
            httpGet:
              port: 8080
              path: /readyz
---
apiVersion: v1
kind: Service
metadata:
  name: firefly-service
  namespace: firefly
spec:
  type: LoadBalancer
  selector:
    app: firefly
  ports:
    - name: firefly-grpc-port
      port: 8081
      targetPort: 8081
    - name: firefly-graphql-port
      port: 8123
      targetPort: 8123
    - name: firefly-rest-port
      port: 8281
      targetPort: 8281
---
EOF
```

### 6. Create ClusterIssuer for Firefly so it is accessible to cert-manager clients
```sh
kubectl apply -f - <<EOF
apiVersion: firefly.venafi.com/v1alpha1
kind: ClusterIssuer
metadata:
  name: firefly-cluster-issuer
EOF
```
>[!NOTE]
> This is only needed if you plan to use solutions like `csi-driver` or `istio-csr` to request certificate from Firefly (i.e., Workload Identity use cases).

### 7. Request a certificate from Firefly by creating a Kubernetes certificate
```sh
kubectl create namespace demo
```
```sh
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: firefly-test-cert
  namespace: demo
  annotations:
    firefly.venafi.com/policy-name: tls-server-ec
spec:
  isCA: false
  commonName: test.venafi.example
  privateKey:
    algorithm: ECDSA
    rotationPolicy: Always
  secretName: firefly-test-cert-secret
  issuerRef:
    name: firefly
    group: firefly.venafi.com
  dnsNames:
  - test.venafi.example
EOF
```
```sh
kubectl describe certificate firefly-test-cert -n demo
```
>[!TIP]
> "tls-server-ec" needs to be the name of a Firefly Policy defined in the Venafi Control Plane, and the policy needs to be associated with the Firefly Configuration associated with the service account the Firefly instance used to bootstrap.  Firefly will only issue a certificate if the `commonName`, (private key) `algorithm`, and `dnsNames` are compliant with the policy specified by the `firefly.venafi.com/policy-name` annotation.

### 8. Request a certificate from Firefly using REST API
First obtain a JSON Web Token (JWT) from the OIDC provider assigned to the **Firefly Configuration** in VCP.  The token must include the [custom claims](https://developer.venafi.com/tlsprotectcloud/docs/firefly-api-reference-clients#claims) required by Firefly.  Assign the JWT to an environment variable called `TOKEN` and then run the following:
```sh
export PAYLOAD="{\"subject\":{\"commonName\":\"test.venafi.example\"},\"altNames\":{\"dnsNames\":[\"test.venafi.example\"]},\"keyType\":\"EC_P256\",\"validityPeriod\":\"P7D\",\"policyName\":\"tls-server-ec\"}"
```
```sh
curl -s --insecure -H "authorization: Bearer ${TOKEN}" \
    -H "content-type: application/json" -d "${PAYLOAD}" \
    https://localhost:8281/v1/certificaterequest
```
>[!NOTE]
> Tools like [Postman](https://www.postman.com/) can also be good for evaluating Firefly's API interfaces for clients.  Venafi Developer Central includes a [detailed reference](https://developer.venafi.com/tlsprotectcloud/docs/firefly-api-reference-clients) for Firefly's API methods including request and response payloads.

## Appendix

### A. Deploying Firefly using Helm instead of manifests

A similar outcome to steps 3, 4, and 5 above can be accomplished using Helm as documented [here](https://docs.venafi.cloud/firefly/deploy/kubernetes/) and [here](https://developer.venafi.com/tlsprotectcloud/docs/firefly-helm-reference). Start by creating a file called `firely.values.yaml` with the following content:

```yaml
acceptTerms: true

deployment:
  enabled: true
  venafiClientID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
  venafiCredentialsSecretName: firefly-svc-acct-key
  venafiCredentialsSecretEnabled: true

  config:
    server:
      grpc:
        enabled: true
        port: 8081
        dnsNames: ["localhost"]
      graphql:
        enabled: true
        port: 8123
        dnsNames: ["localhost"]
      rest:
        enabled: true
        port: 8281
        dnsNames: ["localhost"]
    controller:
      enabled: true

  replicaCount: 2

  image: registry.venafi.cloud/public/venafi-images/firefly
  imagePullPolicy: IfNotPresent

  securityContext:
    capabilities:
      add: ["IPC_LOCK"]
    readOnlyRootFilesystem: true
    runAsNonRoot: true
    runAsUser: 1001

service:
  type: LoadBalancer

crd:
  enabled: true
  groupName: firefly.venafi.com

  approver:
    enabled: false
```

Then execute the following command:
```sh
helm upgrade prod oci://registry.venafi.cloud/public/venafi-images/helm/firefly \
  --install \
  --namespace firefly \
  --values firefly.values.yaml \
  --version v1.3.3
```

>[!IMPORTANT]
>When deploying Firefly using Helm, it will create the RBAC and the CRD for using Firefly as an _Issuer_ but it does not create the CRD for using Firefly as a _ClusterIssuer_ so you'll either need to do the latter or modify step 6 to create an _Issuer_ rather than a _ClusterIssuer_.