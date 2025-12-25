# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

GitOps configuration repository for managing Kubernetes applications via ArgoCD on a k3s cluster (k3s.astafeev.dev).

## Architecture

```
manifests/k3s.astafeev.dev/
├── applications/              # ArgoCD Application entry points (apply these first)
├── <app-name>/               # App-specific manifests (vault-auth, secrets, configs)
```

**Key Components:**
- **ArgoCD** — GitOps controller, self-manages from this repo
- **Vault** — Secrets management with Kubernetes auth
- **Vault Secrets Operator** — Syncs Vault secrets to K8s Secrets via VaultStaticSecret CRDs
- **CloudNative-PG** — PostgreSQL operator for database clusters
- **Apps** — Ghost (blog), n8n (automation)

## Manifest Patterns

**ArgoCD Applications** (`application-*.yaml`):
- Use `automated: {prune: true, selfHeal: true}`
- Include `CreateNamespace=true` in syncOptions
- For webhooks/CRDs add `RespectIgnoreDifferences=true` and `ServerSideApply=true`
- Helm charts from GitHub repos preferred over helm.releases.hashicorp.com (403 issues)

**Vault Integration:**
- Global `VaultConnection` and `VaultAuthGlobal` in `vault-secrets-operator/`
- Per-app `VaultAuth` references global via `vaultAuthGlobalRef: {allowDefault: true}`
- `VaultStaticSecret` syncs secrets from Vault KV-v2 to K8s Secrets

**Vault paths:** `secret/<app-name>/<secret-type>` (e.g., `secret/n8n/postgresql`)

## Adding New Applications

1. Create Vault secret: `kubectl exec vault-0 -n vault -- vault kv put secret/<app>/<name> key=value`
2. Create Vault policy and role for the app namespace
3. Create directory `manifests/k3s.astafeev.dev/<app>/`
4. Add manifests: `vault-auth.yaml`, `vault-static-secret-*.yaml`, app-specific configs
5. Add ArgoCD Application in `applications/application-<app>.yaml`

## Ingress Convention

All public apps use:
- Ingress class: `nginx`
- TLS via cert-manager: `cert-manager.io/cluster-issuer: letsencrypt-prod`
- Domain pattern: `<app>.k3s.astafeev.dev` or custom domains

## Repository URL

When referencing this repo in ArgoCD Applications:
```yaml
repoURL: https://github.com/AllYouZombies/argocd-apps
targetRevision: HEAD
path: manifests/k3s.astafeev.dev/<app>
```
