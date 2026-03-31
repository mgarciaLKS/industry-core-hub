# Migration Guide: Ingress NGINX → Traefik

**Date**: March 2026
**Reason**: [Ingress NGINX has been retired](https://www.kubernetes.dev/blog/2025/11/12/ingress-nginx-retirement/) as of March 2026. No further releases, bugfixes, or security patches will be provided.

---

## Table of Contents

1. [Background](#background)
2. [What Changed in This Project](#what-changed-in-this-project)
3. [Annotation Mapping](#annotation-mapping)
4. [Prerequisites](#prerequisites)
5. [Migration Steps](#migration-steps)
6. [Rollback Plan](#rollback-plan)
7. [Applying to Other Projects](#applying-to-other-projects)
8. [FAQ](#faq)

---

## Background

Kubernetes SIG Network and the Security Response Committee announced the retirement of
Ingress NGINX in November 2025. As of March 2026, the project receives no further
maintenance, security patches, or releases. The GitHub repositories are now read-only.

**Traefik** is a widely adopted, actively maintained ingress controller that natively
supports the standard Kubernetes Ingress API (`networking.k8s.io/v1`). The Ingress
resource specification remains the same; only the `ingressClassName` and any
controller‑specific annotations need to change.

Key Traefik documentation:
- [Traefik Kubernetes Ingress Provider](https://doc.traefik.io/traefik/providers/kubernetes-ingress/)
- [Traefik Kubernetes Ingress Routing](https://doc.traefik.io/traefik/routing/providers/kubernetes-ingress/)

---

## What Changed in This Project

### Chart values (`charts/industry-core-hub/`)

| File | Change |
|------|--------|
| `values.yaml` | `backend.ingress.className`: `"nginx"` → `"traefik"`, `pathType`: `ImplementationSpecific` → `Prefix` |
| `values.yaml` | `frontend.ingress.className`: `"nginx"` → `"traefik"`, `pathType`: `ImplementationSpecific` → `Prefix` |
| `values.yaml` | `keycloak.ingress.annotations`: Removed `nginx.ingress.kubernetes.io/use-regex` → `{}` |
| `values-int.yaml` | pgadmin4 `ingressClassName`: `"nginx"` → `"traefik"` |
| `values-int.yaml` | Frontend/backend ingress: removed nginx annotations (rewrite‑target, use‑regex, enable‑cors, cors‑allow‑*), regex paths `/(.*)`→`/`, `pathType` → `Prefix` |
| `values-int-jupiter.yaml` | className, annotations, pathType migrated |
| `values-int-manufacturer.yaml` | className, annotations, pathType migrated |
| `values-int-manufacturer-jupiter.yaml` | className, annotations, pathType migrated |
| `values-jupiter.yaml` | className, annotations cleaned |
| `values-saturn.yaml` | className, annotations cleaned |
| `values-keycloak.yaml` | Removed `nginx.ingress.kubernetes.io/use-regex` annotation |
| `README.md` | Updated auto‑generated docs tables |

### Deployment values (`deployment/`)

| File | Change |
|------|--------|
| `data-consumer/values-int.yaml` | EDC controlplane/dataplane `className` → `"traefik"` |
| `data-consumer/values-int-vault.yaml` | EDC controlplane/dataplane `className` → `"traefik"` |
| `data-provider/values-int.yaml` | EDC, DTR className → `"traefik"`, nginx annotations → Traefik equivalents, removed deprecated `kubernetes.io/ingress.class` |
| `data-provider/values-int-vault.yaml` | Same as above |
| `data-manufacturer/values-int.yaml` | Same as data-provider |
| `data-manufacturer/values-int-vault.yaml` | Same as above |

### Documentation

| File | Change |
|------|--------|
| `docs/umbrella/umbrella-deployment-guide.md` | Installation instructions updated from ingress‑nginx to Traefik Helm chart |

### Templates — NO changes needed

The Ingress templates (`ingress-backend.yaml`, `ingress-frontend.yaml`, `_helpers.tpl`)
are already controller‑agnostic: they read `className` and `annotations` from values.
They use the standard `networking.k8s.io/v1` Ingress API which is fully compatible with
Traefik.

---

## Annotation Mapping

The following NGINX‑specific annotations were present in this project and have been
replaced or removed:

| NGINX Annotation | Action | Reason |
|---|---|---|
| `nginx.ingress.kubernetes.io/rewrite-target: /$1` | **Removed** | Replaced with `path: /` + `pathType: Prefix` (Traefik native) |
| `nginx.ingress.kubernetes.io/use-regex: "true"` | **Removed** | No longer needed without rewrite‑target regex patterns |
| `nginx.ingress.kubernetes.io/enable-cors: "true"` | **Removed** | FastAPI backend handles CORS natively via middleware |
| `nginx.ingress.kubernetes.io/cors-allow-origin` | **Removed** | Handled by backend CORS middleware |
| `nginx.ingress.kubernetes.io/cors-allow-credentials` | **Removed** | Handled by backend CORS middleware |
| `nginx.ingress.kubernetes.io/force-ssl-redirect: "true"` | → `traefik.ingress.kubernetes.io/router.tls: "true"` | Traefik equivalent for TLS enforcement |
| `nginx.ingress.kubernetes.io/ssl-passthrough: "true"` | → `traefik.ingress.kubernetes.io/router.tls: "true"` | Traefik equivalent |
| `kubernetes.io/ingress.class: nginx` | **Removed** | Deprecated annotation; use `ingressClassName` field instead |
| `cert-manager.io/cluster-issuer` | **Unchanged** | Standard annotation, works with any ingress controller |

### Additional NGINX → Traefik Annotation Reference

For annotations not used in this project but common in other deployments:

| NGINX Annotation | Traefik Equivalent |
|---|---|
| `nginx.ingress.kubernetes.io/proxy-body-size: "10m"` | Use `traefik.ingress.kubernetes.io/router.middlewares` with Buffering middleware |
| `nginx.ingress.kubernetes.io/proxy-read-timeout: "60"` | Configure via ServersTransport CRD |
| `nginx.ingress.kubernetes.io/whitelist-source-range` | Use IPAllowList middleware |
| `nginx.ingress.kubernetes.io/rate-limit-*` | Use RateLimit middleware |
| `nginx.ingress.kubernetes.io/configuration-snippet` | Use middleware chain or IngressRoute CRD |

---

## Prerequisites

### 1. Install Traefik in Your Cluster

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
helm install traefik traefik/traefik --namespace traefik --create-namespace
```

### 2. Verify the Traefik IngressClass Exists

```bash
kubectl get ingressclass
```

Expected output:
```
NAME      CONTROLLER                      PARAMETERS   AGE
traefik   traefik.io/ingress-controller   <none>       5m
```

### 3. (Optional) Remove Ingress NGINX

After verifying Traefik works:
```bash
helm uninstall ingress-nginx --namespace ingress-nginx
kubectl delete namespace ingress-nginx
```

---

## Migration Steps

### Step 1: Update Helm Values

The main `values.yaml` sets defaults; overlay files only need changes where they
explicitly override the ingress configuration.

**Main values.yaml** — change the `className` and `pathType`:

```yaml
# Before
backend:
  ingress:
    className: "nginx"
    hosts:
      - host: ""
        paths:
          - path: "/"
            pathType: "ImplementationSpecific"

# After
backend:
  ingress:
    className: "traefik"
    hosts:
      - host: ""
        paths:
          - path: "/"
            pathType: "Prefix"
```

Apply the same pattern to `frontend.ingress` and any overlay files that explicitly set
`className: "nginx"`.

### Step 2: Remove NGINX‑Specific Annotations

Remove any `nginx.ingress.kubernetes.io/*` annotations from values files. If the
annotation has a Traefik equivalent (see [Annotation Mapping](#annotation-mapping)),
replace it. Otherwise, remove it.

For CORS annotations specifically — if your backend application already handles CORS
(as Industry Core Hub's FastAPI backend does), simply remove the ingress‑level CORS
annotations.

### Step 3: Simplify Regex Paths

NGINX required regex paths with rewrite:
```yaml
# NGINX pattern (remove this)
annotations:
  nginx.ingress.kubernetes.io/rewrite-target: "/$1"
  nginx.ingress.kubernetes.io/use-regex: "true"
paths:
  - path: "/(.*)"
    pathType: "ImplementationSpecific"
```

Traefik handles prefix routing natively:
```yaml
# Traefik pattern (use this)
annotations: {}
paths:
  - path: "/"
    pathType: "Prefix"
```

### Step 4: Validate with Helm Template

```bash
# Default values
helm template test charts/industry-core-hub \
  --set backend.ingress.enabled=true \
  --set frontend.ingress.enabled=true \
  | grep -A 20 "kind: Ingress"

# With overlay
helm template test charts/industry-core-hub \
  -f charts/industry-core-hub/values-int.yaml \
  | grep -A 20 "kind: Ingress"
```

Verify that rendered Ingress resources show `ingressClassName: traefik`.

### Step 5: Deploy and Verify

```bash
# Deploy
helm upgrade --install <release-name> charts/industry-core-hub \
  --namespace <namespace> \
  --set backend.ingress.enabled=true \
  --set frontend.ingress.enabled=true

# Verify ingress resources
kubectl get ingress -n <namespace>

# Check Traefik picked them up
kubectl describe ingress -n <namespace>

# Test connectivity
curl -sv -H "Host: <backend-host>" http://<traefik-ip>:<traefik-port>/
curl -sv -H "Host: <frontend-host>" http://<traefik-ip>:<traefik-port>/
```

---

## Rollback Plan

If you need to revert to NGINX temporarily (while it may still be running in your cluster):

```bash
helm upgrade <release-name> charts/industry-core-hub \
  --namespace <namespace> \
  --set backend.ingress.className=nginx \
  --set frontend.ingress.className=nginx
```

> **Warning**: Ingress NGINX is no longer maintained. Only use this as a short‑term
> rollback while troubleshooting Traefik configuration.

---

## Applying to Other Projects

### General Migration Checklist

- [ ] **Verify Traefik is installed** in the target cluster and the `traefik` IngressClass exists
- [ ] **Search for `nginx` references** in all Helm values, templates, and documentation:
  ```bash
  grep -rn "nginx" charts/ deployment/ docs/ --include="*.yaml" --include="*.md"
  ```
- [ ] **Change `ingressClassName`**: Replace `nginx` with `traefik` in **main** `values.yaml` first, then only in overlays that explicitly override it
- [ ] **Audit annotations**: Replace or remove `nginx.ingress.kubernetes.io/*` annotations (see [Annotation Mapping](#annotation-mapping))
- [ ] **Simplify regex paths**: Replace `/(.*)`+`rewrite-target` patterns with `path: /` + `pathType: Prefix`
- [ ] **Check external chart configs** (EDC, DTR, etc.): These may have their own ingress values that reference nginx
- [ ] **Verify TLS**: cert‑manager annotations work unchanged with Traefik
- [ ] **Validate with `helm template`** before deploying
- [ ] **Deploy to a staging environment first**
- [ ] **Update documentation**: README tables, deployment guides, etc.
- [ ] **Test connectivity** through Traefik endpoints

---

## FAQ

### Do I need to change my Ingress templates?

Only if they contain hardcoded nginx‑specific logic. If your templates read `className`
and `annotations` from values (as this project does), **no template changes are needed**.

### What about `pathType: ImplementationSpecific`?

This pathType delegates interpretation to the ingress controller. Since NGINX and Traefik
may interpret it differently, it's safer to use `Prefix` or `Exact` explicitly. This
project migrated all `ImplementationSpecific` to `Prefix`.

### Does cert‑manager still work?

Yes. cert‑manager annotations (`cert-manager.io/cluster-issuer`, etc.) are
controller‑agnostic and work identically with Traefik.

### What about CORS?

If your backend application handles CORS (e.g., FastAPI CORSMiddleware, Spring CORS
config), remove the nginx ingress CORS annotations — they're redundant and can cause
conflicts. If you need ingress‑level CORS with Traefik, use the Headers middleware.

### Can I use Traefik's IngressRoute CRD instead?

Yes, but it's optional. Standard Kubernetes Ingress resources work perfectly with
Traefik. IngressRoute CRDs offer more advanced routing features if needed in the future.

### What about the Gateway API?

Traefik also supports the newer [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/)
as a future‑proof alternative to Ingress. This can be considered for a future migration
but is not required now.
