# nessus-kubernetes-secrets
## Wrapper Helm chart of freddo256/nessus-kubernetes-argocd
This wrapper Helm chart integrates with (External) Secrets to pull credentials from AWS Parameter Store, avoiding hardcoded secrets.

1. (Optionally) Ensure External Secrets Operator is installed and configured with a ClusterSecretStore for AWS Parameter Store.
2. Apply the ExternalSecret / Secret manifest: `secret-nessus-credentials.yaml` (creates Secret in nessus namespace).
3. Install the helm chart:
   - Add this Git repo as a chart repository (URL: https://gitlab.com/timycyip/nessus-kubernetes-secrets.git, branch: main)
   - Install the `nessus-secrets` chart from the repository
   - Set namespace to `nessus` and configure values as needed (secretName defaults to `nessus-credentials`)
4. The chart deploys Nessus with env vars referencing the Secret via `valueFrom.secretKeyRef`.

## Original Helm Deployment (Without Secrets)
helm repo add nessus https://freddo256.github.io/nessus-kubernetes-argocd/helm/charts
helm install my-nessus nessus/nessus --version 0.2.0 --create-namespace --namespace nessus

Source:
https://github.com/freddo256/nessus-kubernetes-argocd/tree/main/helm

## Credentials Secret

This chart expects a Kubernetes Secret containing Nessus credentials. The Secret name is configurable via `values.yaml:secretName` (default: `nessus-credentials`).

The Secret may contain the following optional keys:
- `activation_code`: Nessus activation code (optional)
- `username`: Nessus admin username (optional)
- `password`: Nessus admin password (optional)

If a key is omitted from the Secret, the corresponding environment variable will not be set in the container. Nessus will use its default behavior (e.g., manual activation through the web interface if activation_code is not provided).

### Option 1: Using External Secrets (Recommended)
Apply the ExternalSecret manifest to pull credentials from AWS Parameter Store:
```bash
kubectl apply -f nessus-system/secret-nessus-credentials.yaml
```

### Option 2: Manual Secret Creation
Create a basic Secret with your credentials:
```bash
kubectl -n nessus create secret generic nessus-credentials \
  --from-literal=activation_code=XXXXX \
  --from-literal=username=XXXXX \
  --from-literal=password=XXXXX
```

Or apply the sample manifest (edit values first):
```bash
kubectl apply -f nessus-system/charts/nessus-secrets/examples/secret-nessus-credentials.yaml
```

## Ingress

The chart supports optional Ingress creation for external access. Enable via `values.yaml:ingress.enabled`.

**Important:** This chart only supports end-to-end TLS passthrough modes. Nessus uses self-signed certificates by default, so TLS termination at the ingress controller is not supported. Choose one of the two passthrough options below.

### Option 1: NGINX Ingress with TLS Passthrough

Use NGINX Ingress Controller with SSL passthrough enabled. This allows end-to-end TLS encryption where the backend (Nessus) presents its own certificate.

**Requirements:**
- NGINX Ingress Controller must be installed with `--enable-ssl-passthrough` flag
- The backend service (Nessus) must be configured to serve HTTPS on port 8834

Example configuration:
```yaml
ingress:
  enabled: true
  className: "nginx"  # default
  annotations: {}     # additional annotations (auto-injected passthrough annotations)
  hosts:
    - host: nessus.example.com
      paths:
        - path: /
          pathType: Prefix
  tls: []  # leave empty for passthrough

nginx:
  sslPassthrough: true  # default
```

The chart automatically injects the required NGINX annotations:
- `nginx.ingress.kubernetes.io/ssl-passthrough: "true"`
- `nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"`
- `nginx.ingress.kubernetes.io/ssl-redirect: "true"`

### Option 2: Traefik IngressRouteTCP with TLS Passthrough

Use Traefik's TCP routing with TLS passthrough for true end-to-end TLS encryption.

**Requirements:**
- Traefik must be installed with CRDs enabled (the official Traefik Helm chart installs them)
- The backend service (Nessus) must be configured to serve HTTPS on port 8834

Example configuration:
```yaml
ingress:
  enabled: false  # disable standard ingress

traefikTcp:
  enabled: true
  entryPoint: websecure
  hostSNI: nessus.example.com
  servicePort: 8834
```

**Notes:**
- With TCP passthrough, Traefik does not manage certificates for the host
- The backend presents its own TLS certificate matching the SNI hostname
- Do not enable both `ingress.enabled` and `traefikTcp.enabled` for the same host

Note: Service type can remain `ClusterIP` when using Ingress.

## Server Certificate Override

By default, Nessus generates and uses a self-signed certificate. Optionally, you can provide a custom server certificate that Nessus will import and use instead.

When enabled, the chart mounts a Kubernetes Secret at `/tls` and runs `nessuscli import-certs` during container startup to import the certificate into Nessus. If the Secret is not present or missing keys, the import is skipped and Nessus continues with its self-signed certificate.

### Option 1: Default (Self-Signed Certificate)
No configuration needed. Nessus uses its built-in self-signed certificate.
```yaml
serverCert:
  enabled: false  # default
```

### Option 2: Pre-Existing Secret
Provide a Secret containing `tls.key` and `tls.crt` keys. The chart will mount and import it.
```yaml
serverCert:
  enabled: true
  secretName: my-nessus-cert  # name of your existing Secret
```

Create the Secret manually:
```bash
kubectl -n nessus create secret tls my-nessus-cert \
  --key=server.key \
  --cert=server.crt
```

### Option 3: cert-manager Managed Certificate
Let cert-manager mint and renew the certificate automatically.
```yaml
serverCert:
  enabled: true
  secretName: nessus-cert-managed  # cert-manager will create this Secret
  certManager:
    enabled: true
    clusterIssuer: letsencrypt-prod  # or use issuerRef below
    # issuerRef:
    #   name: my-issuer
    #   kind: Issuer
    commonName: nessus.example.com
    dnsNames:
      - nessus.example.com
```

**Requirements:**
- cert-manager must be installed in the cluster
- A ClusterIssuer or Issuer must exist (e.g., `letsencrypt-prod`)
- DNS must be configured for certificate validation

---
# Nessus-Kubernetes-ArgoCD
These Kubernetes manifests are used to deploy the nessus Docker image onto a Kubernetes server.
The ArgoCD manifest can be used to do the same in an ArgoCD system.

Remember to change the credentials and use an activation code.
Change the service type if you do not use a load balancer.

## Helm
If you want to use Helm that is possible. Use the following repository URL: `https://freddo256.github.io/nessus-kubernetes-argocd/helm/charts`.
1. `helm repo add nessus https://freddo256.github.io/nessus-kubernetes-argocd/helm/charts` to add the repo.
2. `helm repo update` to update the charts in the repository.
3. `helm search repo nessus` to list the repository.
4. `helm install --values helm/values.yaml nessus nessus/nessus` to deploy the nessus chart.
