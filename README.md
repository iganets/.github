# IGAnets organization configuration

This repository contains shared GitHub configuration for the
[IGAnets organization](https://github.com/iganets). Its main purpose is to keep
CI/CD behavior consistent across IGAnets repositories by providing reusable
GitHub Actions workflows.

The organization profile displayed at
[github.com/iganets](https://github.com/iganets) is maintained separately in
[`profile/README.md`](profile/README.md).

## Reusable workflows

Workflows are called from a repository-level workflow using this form:

```yaml
jobs:
  example:
    uses: iganets/.github/.github/workflows/<workflow>.yml@main
```

The caller controls when the workflow runs. The reusable workflow contains the
implementation of the job itself.

### Docker image publishing

[`docker.yml`](.github/workflows/docker.yml) builds and publishes four
multi-platform images to the GitHub Container Registry:

| Default image | Dockerfile | Build target | Platforms |
| --- | --- | --- | --- |
| `ghcr.io/iganets/iganets:cpu` | `Dockerfile.cpu` | Final stage | `linux/amd64`, `linux/arm64` |
| `ghcr.io/iganets/iganets:cpu-dev` | `Dockerfile.cpu` | `dev` | `linux/amd64`, `linux/arm64` |
| `ghcr.io/iganets/iganets:cuda` | `Dockerfile.cuda` | Final stage | `linux/amd64`, `linux/arm64` |
| `ghcr.io/iganets/iganets:cuda-dev` | `Dockerfile.cuda` | `dev` | `linux/amd64`, `linux/arm64` |

The development images retain the source tree, build tree, compiler toolchain,
and other files from the `dev` Docker stage. The final-stage images contain the
installed runtime.

Example caller:

```yaml
name: Build and publish Docker images

on:
  push:
    branches:
      - main
  workflow_dispatch:

permissions:
  contents: read
  packages: write

jobs:
  docker:
    uses: iganets/.github/.github/workflows/docker.yml@main
    with:
      IGANET_BUILD_TYPE: Release
      IGANET_BUILD_CPUONLY: "OFF"
      IGANET_BUILD_DOCS: "OFF"
      IGANET_BUILD_PCH: "ON"
      IGANET_OPTIONAL: "examples;python"
      IGANET_WITH_GISMO: "OFF"
      IGANET_WITH_MATPLOT: "OFF"
      IGANET_WITH_MPI: "OFF"
      IGANET_WITH_OPENMP: "ON"
      IMAGE_NAME: iganets
```

The workflow uses the caller's `GITHUB_TOKEN`; no registry password is needed.
The caller must grant `contents: read` and `packages: write`.

#### Docker inputs

| Input | Default | Description |
| --- | --- | --- |
| `IGANET_BUILD_TYPE` | `Release` | CMake build type |
| `IGANET_BUILD_CPUONLY` | `OFF` | Enable a CPU-only IGAnets build |
| `IGANET_BUILD_DOCS` | `OFF` | Build the documentation |
| `IGANET_BUILD_PCH` | `ON` | Build precompiled headers |
| `IGANET_OPTIONAL` | Empty | Semicolon-separated optional modules |
| `IGANET_WITH_GISMO` | `OFF` | Enable G+Smo support |
| `IGANET_WITH_MATPLOT` | `OFF` | Enable Matplot++ support |
| `IGANET_WITH_MPI` | `OFF` | Enable MPI support |
| `IGANET_WITH_OPENMP` | `ON` | Enable OpenMP support |
| `IMAGE_NAME` | `iganets` | GHCR package name, without owner or registry |
| `CPU_TAG` | `cpu` | CPU runtime image tag |
| `CPU_DEV_TAG` | `cpu-dev` | CPU development image tag |
| `CUDA_TAG` | `cuda` | CUDA runtime image tag |
| `CUDA_DEV_TAG` | `cuda-dev` | CUDA development image tag |
| `CPU_DOCKERFILE` | `Dockerfile.cpu` | CPU Dockerfile path |
| `CUDA_DOCKERFILE` | `Dockerfile.cuda` | CUDA Dockerfile path |
| `BUILD_CONTEXT` | `.` | Docker build context |

Values passed to CMake-style `ON`/`OFF` inputs should be quoted so YAML treats
them as strings. Quote `IGANET_OPTIONAL` as well when it contains semicolons.

### GitLab synchronization

[`gitlab-sync.yml`](.github/workflows/gitlab-sync.yml) mirrors the complete Git
repository, including all branches and tags, to a GitLab repository.

Example caller:

```yaml
name: GitLab Sync

on:
  - push
  - delete

jobs:
  sync:
    uses: iganets/.github/.github/workflows/gitlab-sync.yml@main
    secrets:
      GITLAB_REPO_TOKEN: ${{ secrets.GITLAB_REPO_TOKEN }}
      GITLAB_REPO_URL: ${{ secrets.GITLAB_REPO_URL }}
      GITLAB_REPO_USERNAME: ${{ secrets.GITLAB_REPO_USERNAME }}
```

Configure the three referenced secrets in the calling repository. The GitLab
token must have permission to push to the target repository.

### CMake build and test

[`cmake-multi-platform.yml`](.github/workflows/cmake-multi-platform.yml) builds
and tests an IGAnets CMake project using:

- GCC and Clang on Ubuntu
- Clang on macOS
- LibTorch CPU

Example caller:

```yaml
name: CMake CI

on:
  - push
  - pull_request

jobs:
  test:
    uses: iganets/.github/.github/workflows/cmake-multi-platform.yml@main
    with:
      IGANET_OPTIONAL: "examples"
```

`IGANET_OPTIONAL` is optional and accepts the same semicolon-separated module
list used by the IGAnets CMake configuration.

### Documentation deployment

[`gh-pages.yml`](.github/workflows/gh-pages.yml) configures the project with
documentation enabled, generates the Doxygen/Sphinx documentation, and deploys
the result to GitHub Pages.

Example caller:

```yaml
name: Publish documentation

on:
  push:
    branches:
      - main
  workflow_dispatch:

permissions:
  contents: write
  pages: write
  id-token: write

jobs:
  docs:
    uses: iganets/.github/.github/workflows/gh-pages.yml@main
    with:
      IGANET_OPTIONAL: "examples"
```

GitHub Pages must be configured to use **GitHub Actions** as its deployment
source in the calling repository.

## Repository layout

```text
.
├── .github/
│   └── workflows/          # Reusable organization workflows
├── profile/
│   └── README.md           # Public organization profile
└── README.md               # Contributor documentation
```

## Maintenance

Changes to a workflow referenced with `@main` are immediately used by every
calling repository. Test changes with a branch reference in a caller before
merging them:

```yaml
uses: iganets/.github/.github/workflows/docker.yml@my-workflow-branch
```

For reproducible or security-sensitive callers, reference a release tag or
commit SHA instead of `@main`.
