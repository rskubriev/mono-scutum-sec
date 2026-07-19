# Scutum Security Workflow

[Русская версия](README.ru.md)

Reusable GitHub Actions workflow for building Docker images without shipping common configuration, secret, vulnerability, or provenance failures. It runs Hadolint and Trivy before an image is published, creates a CycloneDX SBOM with Syft, and can attach the SBOM and sign the image through Cosign keyless signing.

## Use it

Create `.github/workflows/container-security.yml` in the repository that owns the Dockerfile:

```yaml
name: Container security

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

permissions:
  contents: read
  packages: write
  id-token: write

jobs:
  security:
    uses: rskubriev/mono-scutum-sec/.github/workflows/docker-security.yml@main
    with:
      image-name: ghcr.io/OWNER/IMAGE
      push: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}
      sign: true
```

Replace `OWNER/IMAGE` with the target registry path. Pull requests validate and scan the image without publishing it. A push to `main` publishes the SHA-tagged image, uploads its SBOM as an artifact, attaches the SBOM in the registry, and signs the image through GitHub OIDC.

## What the workflow does

1. **Lint and secret scan.** Hadolint checks Dockerfile practices; Trivy scans the source tree for exposed credentials.
2. **Build and vulnerability scan.** Docker builds the image, then Trivy blocks fixed `HIGH` and `CRITICAL` vulnerabilities.
3. **Produce an inventory.** Syft creates a CycloneDX SBOM that records the components found in the image.
4. **Publish and prove origin.** When `push: true`, the workflow pushes the scanned image, attaches its SBOM, and can sign it with Cosign and GitHub OIDC.

## Tools and limits

| Tool | Role | What it does not prove |
| --- | --- | --- |
| Hadolint | Finds unsafe or non-portable Dockerfile and shell patterns. | It does not inspect the final image or application code. |
| Trivy | Finds recognised secrets and known package vulnerabilities. | It cannot guarantee detection of custom secrets, zero-days, or incomplete package metadata. |
| Syft | Creates an SBOM: an inventory of packages in the image. | An SBOM is not a vulnerability scan and does not make an image safe. |
| Cosign | Binds a published image to a GitHub OIDC identity and records the signing event. | A signature must still be verified and enforced by the deployment environment. |

## Inputs

| Input | Default | Purpose |
| --- | --- | --- |
| `context` | `.` | Docker build context in the calling repository. |
| `dockerfile` | `Dockerfile` | Dockerfile path relative to the calling repository. |
| `image-name` | empty | Fully qualified image name, for example `ghcr.io/owner/image`. Required when `push` is `true`. |
| `push` | `false` | Push the scanned image after all checks complete. |
| `sign` | `true` | Use keyless Cosign signing after a push. |

## Versioning

During early development, reference `@main` to receive the latest workflow. Stable releases use major tags such as `@v1`; update a major tag only for backward-compatible changes. For the strongest supply-chain control, pin the reusable workflow to a full commit SHA and review updates intentionally.

## Trust and security model

Do not trust a reusable workflow solely because it is public. Review the exact revision and pin it by commit SHA in repositories with sensitive data:

```yaml
uses: rskubriev/mono-scutum-sec/.github/workflows/docker-security.yml@FULL_COMMIT_SHA
```

All third-party container tools and GitHub Actions in this repository are pinned to immutable image digests or commit SHA values. This workflow defines no secret inputs and does not require `secrets: inherit`; callers should grant the minimum permissions required and omit `packages: write` and `id-token: write` when they do not publish or sign images.

Treat a Dockerfile as executable input: its `RUN` instructions execute during `docker build`. Do not run unreviewed pull requests on a self-hosted runner that can access sensitive networks or credentials. At deployment time, verify the Cosign identity and image digest, then enforce that policy so unsigned images cannot run.

Use `examples/caller-workflow.yml` as a complete starting point. This repository intentionally contains no application code, Dockerfiles, credentials, or test images.
