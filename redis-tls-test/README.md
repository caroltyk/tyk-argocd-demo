## Install Redis Cluster with TLS enabled and use cert-manager to manage certificates

This document is for setting up an environment on K8s to test secure SSL connection between Gateway and Redis Cluster.

**Note currently Tyk Gateway cannot connect to Redis Cluster with TLS connection if sslInsecureSkipVerify is false. Need to investigate what's wrong with the certificate generated. **

There are 4 high level steps:
1. Install cert-manager and trust-manager and setup Cluster Issuers
2. Create a trusted TLS certificate for Redis
3. Install Redis Cluster with TLS enabled
4. Configure Tyk Gateway to connect with Redis Cluster using secure SSL connection

## Step 1: Install cert-manager and trust-manager and setup Cluster Issuers
Cert-manager can be used to generate CA certificates or TLS certificates in Kubernetes. Trust-manager is a tool to help you 
distribute CA certs to different namespaces.

1. Install [cert-manager](https://cert-manager.io/docs/installation/helm/) and
[trust-manager](https://cert-manager.io/docs/trust/trust-manager/installation/)

```
helm repo add jetstack https://charts.jetstack.io --force-update

helm upgrade cert-manager jetstack/cert-manager \
  --install \
  --create-namespace \
  --namespace cert-manager \
  --set installCRDs=true

helm upgrade trust-manager jetstack/trust-manager \
  --install \
  --namespace cert-manager \
  --wait
```

2. Create Issuer

The [CA issuer](https://cert-manager.io/docs/configuration/ca/) represents a Certificate Authority whose certificate and private 
key are stored inside the cluster as a Kubernetes Secret. Use trust-manager to distribute your CA certificate safely across your cluster.

CA Issuers must be configured with a certificate and private key stored in a Kubernetes secret. You can create this externally if you wish, 
or you could bootstrap a root certificate using a SelfSigned issuer.

First create a self-signed issuer to bootstrap the root certificate.

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned
  annotations:
    argocd.argoproj.io/sync-wave: "25"
spec:
  selfSigned: {}
```

Then, create a Certificate Request, using selfsigned issuer, to generate a secret called
`ca-key-pair`. 

See [Certificate resource](https://cert-manager.io/docs/usage/certificate/) for all 
configurable fields for requesting certificate.

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: ca-cert
  namespace: cert-manager
  annotations:
    argocd.argoproj.io/sync-wave: "26"
spec:
  # Secret names are always required.
  secretName: ca-key-pair

  # secretTemplate is optional. If set, these annotations and labels will be
  # copied to the Secret named example-com-tls. These labels and annotations will
  # be re-reconciled if the Certificate's secretTemplate changes. secretTemplate
  # is also enforced, so relevant label and annotation changes on the Secret by a
  # third party will be overwriten by cert-manager to match the secretTemplate.
#  secretTemplate:
#    annotations:
#      my-secret-annotation-1: "foo"
#      my-secret-annotation-2: "bar"
#    labels:
#      my-secret-label: foo

  duration: 8760h0m0s # 90d
  renewBefore: 360h0m0s # 15d
  subject:
    organizations: # Organizations to be used on the Certificate
      - Tyk
  # The use of the common name field has been deprecated since 2000 and is
  # discouraged from being used.
  # CN - commonName
  # L - localityName
  # ST - stateOrProvinceName
  # O - organizationName
  # OU - organizationUnitName
  # C - countryName
  # STREET - streetAddress
  # DC - domainComponent
  # UID - userId
  commonName: "*.svc.cluster.local"
  isCA: true
  privateKey:
    algorithm: RSA
    encoding: PKCS1
    size: 2048
  usages:
    - digital signature
    - key encipherment
    - server auth
    - client auth
  # At least one of a DNS Name, URI, IP address or otherName is required.
  dnsNames:
    - "*.svc"
    - "*.svc.cluster.local"
    - "*.pod.cluster.local"
#  uris:
#    - spiffe://cluster.local/ns/sandbox/sa/example
#  ipAddresses:
#    - 10.131.132.140
#    - 10.131.132.141
#    - 10.131.97.63
  # Needs cert-manager 1.14+ and "OtherNames" feature flag
#  otherNames:
    # Should only supply oid of ut8 valued types
#    - oid: 1.3.6.1.4.1.311.20.2.3 # User Principal Name "OID"
#      utf8Value: upn@example.local
  # Issuer references are always required.
  issuerRef:
    name: selfsigned
    # We can reference ClusterIssuers by changing the kind here.
    # The default value is Issuer (i.e. a locally namespaced Issuer)
    ###kind: Issuer
    kind: ClusterIssuer
    # This is optional since cert-manager will default to this value however
    # if you are using an external issuer, change this to that issuer group.
    group: cert-manager.io

  # keystores allows adding additional output formats. This is an example for reference only.
#  keystores:
#    pkcs12:
#      create: true
#      passwordSecretRef:
#        name: example-com-tls-keystore
#        key: password
#      profile: Modern2023
```

Finally, create a Cluster Issuer `cluster-ca` which use the generated tls secret `ca-key-pair` to
sign and issue certificates for services in the Kubernetes cluster.

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: cluster-ca
  annotations:
    argocd.argoproj.io/sync-wave: "26"
spec:
  ca:
    secretName: ca-key-pair
```

3. Create Bundle resource

By creating this `Bundle` resource. `trust-manager` will create a secret named `my-bundle` in all namespaces which include
the `ca.crt` certificate from `ca-key-pair` secret, which is the root certificate from ClusterIssuer `cluster-ca`.

```yaml
apiVersion: trust.cert-manager.io/v1alpha1
kind: Bundle
metadata:
  name: my-bundle  # The bundle name will also be used for the target
spec:
  sources:
  # Include a bundle of publicly trusted certificates which can be
  # used to validate most TLS certificates on the internet, such as
  # those issued by Let's Encrypt, Google, Amazon and others.
  - useDefaultCAs: false
  - secret:
      name: "ca-key-pair"
      key: "ca.crt"

  target:
    # Sync the bundle to a ConfigMap called `my-bundle` in every namespace which
    # has the label "my-bundle/inject=enabled"
    # All ConfigMaps will include a PEM-formatted bundle, here named "root-certs.pem"
    secret:
      key: "trust-bundle.pem"
```

## Step 2: Create a trusted TLS certificate for Redis

Create this `Certificate` resource to request TLS certificate and store it in secret
`redis-cluster-tls` in `redis-cluster` namespace, using the `cluster-ca` issuer created in
step 1.

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: tyk-redis-cluster-cert
  namespace: redis-cluster
  annotations:
    argocd.argoproj.io/sync-wave: "40"
spec:
  # Secret names are always required.
  secretName: redis-cluster-tls

  # secretTemplate is optional. If set, these annotations and labels will be
  # copied to the Secret named example-com-tls. These labels and annotations will
  # be re-reconciled if the Certificate's secretTemplate changes. secretTemplate
  # is also enforced, so relevant label and annotation changes on the Secret by a
  # third party will be overwriten by cert-manager to match the secretTemplate.
#  secretTemplate:
#    annotations:
#      my-secret-annotation-1: "foo"
#      my-secret-annotation-2: "bar"
#    labels:
#      my-secret-label: foo

  duration: 2160h0m0s # 90d
  renewBefore: 360h0m0s # 15d
  #subject:
  #  organizations: # Organizations to be used on the Certificate
  #    - Tyk
  # The use of the common name field has been deprecated since 2000 and is
  # discouraged from being used.
  # CN - commonName
  # L - localityName
  # ST - stateOrProvinceName
  # O - organizationName
  # OU - organizationUnitName
  # C - countryName
  # STREET - streetAddress
  # DC - domainComponent
  # UID - userId
  commonName: "tyk-redis-cluster.redis-cluster.svc"
  # isCA: false
  privateKey:
    algorithm: ECDSA
    size: 256
  usages:
    - digital signature
    - key encipherment
    - server auth
    - client auth
  # At least one of a DNS Name, URI, IP address or otherName is required.
  dnsNames:
    - "*.redis-cluster.svc"
    - "*.redis-cluster.svc.cluster.local"
#  uris:
#    - spiffe://cluster.local/ns/sandbox/sa/example
#  ipAddresses:
#    - "10.244.0.173"
#    - "10.244.1.114"
#    - "10.244.0.56"
#    - "10.244.0.9"
#    - "10.244.1.6"
#    - "10.244.0.192"
# Needs cert-manager 1.14+ and "OtherNames" feature flag
#  otherNames:
    # Should only supply oid of ut8 valued types
#    - oid: 1.3.6.1.4.1.311.20.2.3 # User Principal Name "OID"
#      utf8Value: upn@example.local
  # Issuer references are always required.
  issuerRef:
    name: cluster-ca
    #name: selfsigned
    # We can reference ClusterIssuers by changing the kind here.
    # The default value is Issuer (i.e. a locally namespaced Issuer)
    ###kind: Issuer
    kind: ClusterIssuer
    # This is optional since cert-manager will default to this value however
    # if you are using an external issuer, change this to that issuer group.
    group: cert-manager.io

  # keystores allows adding additional output formats. This is an example for reference only.
#  keystores:
#    pkcs12:
#      create: true
#      passwordSecretRef:
#        name: example-com-tls-keystore
#        key: password
#      profile: Modern2023
```

## Step 3: Install Redis Cluster with TLS enabled

Use [redis-cluster](https://artifacthub.io/packages/helm/bitnami/redis-cluster) chart from 
bitnami to install Redis Cluster.

```bash
kubectl create secret generic redis-password-secret --namespace redis-cluster \
  --from-literal=redis-password=$REDIS_PASSWORD
```

```bash
helm install tyk-redis-cluster oci://registry-1.docker.io/bitnamicharts/redis-cluster \
  --version=10.0.0 \
  --set volumePermissions.enabled='true' \
  --set existingSecret=redis-password-secret \
  --set existingSecretPasswordKey=redis-password \
  --set tls.enabled='true' \
  --set tls.authClients='false' \
  --set tls.existingSecret='redis-cluster-tls' \
  --set tls.certFilename='tls.crt' \
  --set tls.certKeyFilename='tls.key' \
  --set tls.certCAFilename='ca.crt'
```

## Step 4: Configure Tyk Gateway to connect with Redis Cluster using secure SSL connection

```
kubectl create secret generic redis-password-secret --namespace tyk-oss \
  --from-literal=redis-password=$REDIS_PASSWORD
```

```bash
helm install tyk-oss https://helm.tyk.io/public/helm/charts/tyk-oss \
    --namespace tyk-oss \
    --version=1.3.0 \
    --set 'global.secrets.APISecret=foo' \
    --set 'global.redis.addrs[0]=tyk-redis-cluster.redis-cluster.svc:6379' \
    --set 'global.redis.enableCluster="true"' \
    --set 'global.redis.useSSL="true"' \
    --set 'global.redis.sslInsecureSkipVerify="false"' \
    --set 'global.redis.tlsMinVersion=1.2' \
    --set 'global.redis.tlsMaxVersion=1.3' \
    --set 'global.redis.sslCAFile="/etc/ssl/certs/trust-bundle.pem"' \
    --set 'global.redis.certificatesMountPath="/etc/ssl/certs"' \
    --set 'global.redis.secretName="my-bundle"' \
    --set 'global.redis.volumeName="ca-cert"' \
    --set 'global.redis.passSecret.name=redis-password-secret' \
    --set 'global.redis.passSecret.keyName=redis-password'
```
