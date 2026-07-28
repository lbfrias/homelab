# Test Branches Setup

The test branch workflow automatically deploys `test/*` branches to the `test` namespace.

## How it works

1. Push to a `test/*` branch (e.g., `test/jellyfin`)
2. ResourceSetInputProvider detects the branch (checks every 5 minutes)
3. ResourceSet creates a GitRepository + Kustomization for that branch
4. Flux deploys the manifests to the `test` namespace
5. Delete the branch → resources are automatically pruned

## Rate limits (optional)

For public repos, the GitHub API works without authentication (60 requests/hour).
If you hit rate limits, create a token secret:

```bash
kubectl create secret generic github-token \
  -n flux-system \
  --from-literal=token=ghp_your_token_here
```

Then add `secretRef` to the ResourceSetInputProvider.
