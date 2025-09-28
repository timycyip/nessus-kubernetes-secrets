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

The Secret must contain the following keys:
- `activation_code`: Nessus activation code
- `username`: Nessus admin username
- `password`: Nessus admin password

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

### NGINX Ingress with TLS Passthrough (Default)

By default, the chart is configured for NGINX Ingress with TLS passthrough enabled. This allows end-to-end TLS encryption where the backend (Nessus) presents its own certificate.

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

### Traefik Ingress with Upstream HTTPS

For Traefik, configure HTTPS termination at Traefik with re-encryption to the backend:

```yaml
ingress:
  enabled: true
  className: "traefik"
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    traefik.ingress.kubernetes.io/router.tls: "true"
    traefik.ingress.kubernetes.io/service.port: "8834"
    traefik.ingress.kubernetes.io/service.serversscheme: https
    traefik.ingress.kubernetes.io/service.serverstransport: "nessus/nessus-transport"
  hosts:
    - host: nessus.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: nessus-tls
      hosts:
        - nessus.example.com

traefik:
  serversTransport:
    create: true
    insecureSkipVerify: true
```

### Traefik TCP Passthrough (End-to-End TLS)

For true end-to-end TLS with Traefik doing only SNI-based TCP routing:

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
- DNS must be configured separately (external-dns may not watch Traefik CRDs by default)
- Do not enable both `ingress.enabled` and `traefikTcp.enabled` for the same host

Note: Service type can remain `ClusterIP` when using Ingress.

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
