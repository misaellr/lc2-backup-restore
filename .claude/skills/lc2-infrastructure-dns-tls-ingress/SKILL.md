---
name: lc2-dns-tls-ingress
description: Configure cert-manager and ingress the same way across clouds, swapping only the provider-specific controller details.
allowed-tools: Read, Grep, Glob
---

# LC2 DNS, TLS & Ingress

## Goal
Standardize TLS with cert-manager and Let's Encrypt across clouds, keeping ingress patterns uniform.

## Steps
1. Install cert-manager via Helm (if not present).
2. Apply a **ClusterIssuer** for Let's Encrypt (staging first, then production).
3. Configure ingress with the appropriate annotations for ALB/NGINX/GKE Ingress.
4. Use DNS records or static IPs as required by each provider.
5. Validate certificate issuance and ingress health.

## Example ClusterIssuer (Let’s Encrypt Staging)
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: you@example.com
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
    - http01:
        ingress:
          class: nginx
```

## Notes
- Keep provider specifics in the adapter; maintain a consistent ingress contract to applications.
