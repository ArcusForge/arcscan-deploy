# arcscan-deploy

Public deploy harness for [ArcusForge/ArcScan](https://github.com/ArcusForge/ArcScan) (private).

## Why this exists

GitHub Actions allots 2,000 minutes/month for private repos. Public repos get unlimited minutes. This repo runs the deploy pipeline for ArcScan on the free public-repo budget while keeping the source private.

## How it works

1. Operator pushes to `main` on the private ArcusForge/ArcScan.
2. A 10-second notify workflow on the private side fires a `repository_dispatch` event at this repo with the commit SHA.
3. This repo's `deploy.yml` workflow:
   - Checks out ArcScan @ that SHA via a fine-grained PAT (read-only on the private repo).
   - Packages a release zip via `git archive HEAD`.
   - Assumes an AWS IAM role via OIDC (no static AWS keys stored anywhere).
   - Uploads the zip to S3.
   - Sends an SSM command to the prod EC2 instance to download, migrate, restart services.
   - Polls SSM for completion and reports back.
   - Smoke-tests `https://arcusautomate.com/`.

There is also a `workflow_dispatch` trigger so the operator can deploy any SHA manually from the Actions tab — useful for rollbacks.

## Security model

- **No source code** lives here. This repo is checked-out *into* by the workflow, but contains no ArcScan source itself.
- **No `pull_request` trigger.** PR workflows on public repos run without secrets, but the existence of any PR-triggered workflow is a footgun. We deliberately omit it.
- **No static AWS keys.** AWS access is via OIDC role assumption. The IAM role's trust policy is scoped to:
  ```
  repo:ArcusForge/arcscan-deploy:ref:refs/heads/main
  ```
  — meaning only this repo, only this branch, can assume it.
- **The AWS role's permissions are scoped** to `s3:PutObject` on the deploy bucket's `releases/` prefix and `ssm:SendCommand` on a single EC2 instance.
- **The PAT is fine-grained**, scoped read-only to ArcusForge/ArcScan, with no other permissions.

## Required secrets

| Name | Type | Purpose |
|------|------|---------|
| `PRIVATE_REPO_PAT` | Fine-grained PAT | Read access to ArcusForge/ArcScan source. Rotate quarterly. |

## Manual deploy

```bash
gh workflow run deploy.yml \
  --repo ArcusForge/arcscan-deploy \
  --ref main \
  -f sha=<full-or-short-git-sha-on-main>
```

## Wiring on the private side

The private repo (`ArcusForge/ArcScan`) has a notify workflow (`.github/workflows/notify-public-deploy.yml`) that fires a `repository_dispatch` to this repo on every push to `main`.
