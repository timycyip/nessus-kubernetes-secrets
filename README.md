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

Example configuration for Traefik:
```yaml
ingress:
  enabled: true
  className: "traefik"
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
  hosts:
    - host: nessus.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: nessus-tls
      hosts:
        - nessus.example.com
```

For NGINX:
```yaml
ingress:
  enabled: true
  className: "nginx"
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
  hosts:
    - host: nessus.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: nessus-tls
      hosts:
        - nessus.example.com
```

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