# GitOps Repo (Argo CD + Kustomize)

This repo is meant to be separate from your app source. Argo CD watches this repo and applies
Kubernetes manifests for each environment (staging/prod). Jenkins (or your CI) only changes
the **image tag** in the env overlay, commits, and pushes.

## Structure
```
base/                 # shared manifests (Deployment/Service)
envs/
  staging/            # overlay patches image to staging tag
  prod/               # overlay patches image to prod tag
apps/
  myapp-staging-application.yaml  # Argo CD Application (points to envs/staging)
  myapp-prod-application.yaml     # Argo CD Application (points to envs/prod)
```

## Setup
1) Replace placeholders:
   - In `apps/*.yaml`, set `repoURL` to this repo's Git URL.
2) Install Argo CD into the cluster (namespace `argocd`) and apply the Applications:
   ```bash
   kubectl create namespace argocd
   # Use a pinned Argo CD version manifest:
   kubectl -n argocd apply -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   kubectl apply -f apps/myapp-staging-application.yaml
   kubectl apply -f apps/myapp-prod-application.yaml
   ```
3) (Optional) Log into Argo CD UI to watch syncs:
   ```bash
   kubectl -n argocd port-forward svc/argocd-server 8080:80
   # username: admin
   kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
   ```

## How Jenkins promotes
- Jenkins replaces `$IMAGE` in `envs/staging/kustomization.yaml` (or `prod`) with something like:
  `123456789012.dkr.ecr.ap-southeast-1.amazonaws.com/myapp:123`
- Then commits and pushes to this repo.
- Argo CD detects the change and syncs.

## Notes
- For canary/blue-green, consider Argo Rollouts (separate controller + CRD).
- For Helm users, you can set `spec.source.chart` instead of directory/kustomize.
