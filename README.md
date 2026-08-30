# Bump Image Tag

A composite GitHub Action that bumps a container image tag in a
Kustomize-based GitOps manifests repository and pushes the commit.

## Why this exists

A common GitOps setup has an app repo (builds and pushes an image) and a
separate manifests repo (Kubernetes YAML, applied via `kubectl apply -k` or
an ArgoCD/Flux-style controller). If the manifests repo pins every image to
`:latest`, neither `kubectl apply` nor a GitOps controller ever sees a diff
when a new image is pushed — the tag string never changes — so nothing
actually redeploys without a separate manual restart. This action closes
that gap: it runs `kustomize edit set image` in the manifests repo to point
at the real tag you just built, commits, and pushes, so there's a genuine,
git-visible change for your deploy pipeline (or ArgoCD) to act on.

## Usage

```yaml
- name: Bump image tag
  uses: <owner>/bump-image-tag-action@v1
  with:
    manifests-repo: my-org/infra
    manifests-ref: main
    manifests-path: deployment/base/my-app
    image: registry.example.com/my-org/my-app
    tag: ${{ steps.vars.outputs.sha_short }}
    token: ${{ secrets.MANIFESTS_REPO_TOKEN }}
```

This expects `manifests-path` to already contain a `kustomization.yaml`
(any valid Kustomize base or overlay directory works). The action doesn't
create one for you.

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `manifests-repo` | yes | | `"owner/repo"` of the GitOps manifests repository |
| `manifests-ref` | no | `main` | Branch to check out and push the bump commit to |
| `manifests-path` | yes | | Path within the manifests repo to the Kustomize directory |
| `image` | yes | | Full image reference without a tag |
| `tag` | yes | | The new tag to set |
| `token` | yes | | Token with write access to `manifests-repo` (see below) |
| `kustomize-version` | no | `5.5.0` | Kustomize version installed to run `kustomize edit set image` |
| `git-user-name` | no | `bump-image-tag-action[bot]` | Commit author/committer name |
| `git-user-email` | no | `bump-image-tag-action@users.noreply.github.com` | Commit author/committer email |
| `commit-message` | no | `chore: bump {image} to {tag}` | Commit message template (`{image}`/`{tag}` placeholders) |
| `max-retries` | no | `3` | Retries (with fetch + rebase) if the push races a concurrent bump |

## Outputs

| Output | Description |
|---|---|
| `commit-sha` | SHA of the pushed bump commit, empty if `skipped` |
| `skipped` | `"true"` if the image was already at the requested tag (no-op, nothing pushed) |

## About `token`

The default `GITHUB_TOKEN` in a workflow only has write access to the repo
the workflow is running in. If `manifests-repo` is a *different* repo (the
common case — an app repo bumping a separate infra/manifests repo), you
need a token that can push there: a fine-grained personal access token
scoped to that repo's contents, a classic PAT, or a GitHub App installation
token. Store it as a secret in the calling repo and pass it in via `token`.

## Notes

- Only bumps the image tag — it doesn't build, push, or know anything about
  your registry credentials. Run this after your own build/push steps.
- If two workflow runs bump different apps in the same manifests repo at
  the same time, this is safe as long as they touch different files/paths
  (the common case for one Kustomize directory per app). If retries are
  exhausted (`max-retries`), the step fails loudly rather than silently
  dropping a bump.
- Before publishing this publicly (e.g. to the GitHub Marketplace), add a
  `LICENSE` file — MIT is a common, low-friction choice for an action like
  this, but that's your call to make, not something to inherit by default.
