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

## Security model

All third-party container tools and GitHub Actions are pinned to immutable commit or image digests. The workflow gives read access by default; package publishing and OIDC signing permissions are declared only for the build job. The calling repository must allow GitHub Actions to write packages when `push: true` is used.

Use `examples/caller-workflow.yml` as a complete starting point. This repository intentionally contains no application code, Dockerfiles, credentials, or test images.
