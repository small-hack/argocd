# Longhorn

Longhorn is a lightweight, reliable and easy-to-use distributed block storage system for Kubernetes. Longhorn is free, open source software. Originally developed by Rancher Labs, it is now being developed as a incubating project of the Cloud Native Computing Foundation.

## Prereqs

- Check host system for required pacakages
  ```bash
  curl -sSfL https://raw.githubusercontent.com/longhorn/longhorn/v1.5.1/scripts/environment_check.sh | bash
  ```

## Deploy
You can copy and paste this into your edit section when you're creating a new argocd app:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: longhorn
spec:
  destination:
    namespace: longhorn-system
    server: 'https://kubernetes.default.svc'
  source:
    path: longhorn/
    repoURL: 'https://github.com/small-hack/argocd-apps.git'
    targetRevision: HEAD
  project: default
  syncPolicy:
    automated:
      prune: false
      selfHeal: false
    syncOptions:
      - CreateNamespace=true
```
