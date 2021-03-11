# GitHub Actions Manifest Generation

This guide shows how to handle a basic use-case of manifest generation with GitHub Actions, using third-party tools in CI to run pre-processing that prepares YAML manifests for deployment, and then commit to a deploy branch in git for kustomize-controller to apply them.

Usage:

Flux's primary use case is to apply manifests to a cluster, from the latest commit in a branch ref.

There are two basic use cases described below, both of which demonstrate how to add commits and push updates to a branch with some slight differences. One example is doing a simple in-place string substitution using `sed -i` with a variable that is made available by the source host, in CI. The second example is using a third-party tool, `jsonnet`, to do what may be colloquially referred to as "YAML rehydration".

### String Substitution with `sed -i`

The entry point for this example starts at `.github/workflows/` in your source repository. There are two actions used together here. The use case served is, to build a manifest with the commit hash of the latest commit to deploy. Also shown here, is how to build an image and push a tag from the latest commit on the branch. This example targets any commit on any branch.

!!! warning "`GitRepository` source only targets one branch"
    While this example updates any branch with CI, each `Kustomization` only deploys manifests from one branch or tag at a time. These workflows are meant to go into for your application's repository, wherever images are built from. Use this technique to manage preview environments.

```yaml
# ./.github/workflows/
name: Manifest Generation
on:
  push:
    branches:
    - '*'

jobs:
  run:
    name: Push Update
    runs-on: ubuntu-latest
    steps:
      - name: Prepare
        id: prep
        run: |
          VERSION=${GITHUB_SHA::8}
          if [[ $GITHUB_REF == refs/tags/* ]]; then
            VERSION=${GITHUB_REF/refs\/tags\//}
          fi
          echo ::set-output name=BUILD_DATE::$(date -u +'%Y-%m-%dT%H:%M:%SZ')
          echo ::set-output name=VERSION::${VERSION}
      - name: Checkout repo
        uses: actions/checkout@v2

      - name: Update manifests
        run: ./update-k8s.sh $GITHUB_SHA

      - name: Commit changes
        uses: EndBug/add-and-commit@v7
        with:
          add: '.'
          message: "[ci skip] deploy from ${{ steps.prep.outputs.VERSION }}"
          signoff: true
    steps:
      - name: Setup Flux CLI
        uses: fluxcd/flux2/action@main
      - name: Run Flux commands
        run: flux -v
```

Note that this action can only be used on GitHub **Linux AMD64** runners.
The latest stable version of the `flux` binary is downloaded from
GitHub [releases](https://github.com/fluxcd/flux2/releases)
and placed at `/usr/local/bin/flux`.

You can download a specific version with:

```yaml
    steps:
      - name: Setup Flux CLI
        uses: fluxcd/flux2/action@main
        with:
          version: 0.8.0
```

### Automate Flux updates

Example workflow for updating Flux's components generated with `flux bootstrap --path=clusters/production`:

```yaml
name: update-flux

on:
  workflow_dispatch:
  schedule:
    - cron: "0 * * * *"

jobs:
  components:
    runs-on: ubuntu-latest
    steps:
      - name: Check out code
        uses: actions/checkout@v2
      - name: Setup Flux CLI
        uses: fluxcd/flux2/action@main
      - name: Check for updates
        id: update
        run: |
          flux install \
            --export > ./clusters/production/flux-system/gotk-components.yaml

          VERSION="$(flux -v)"
          echo "::set-output name=flux_version::$VERSION"
      - name: Create Pull Request
        uses: peter-evans/create-pull-request@v3
        with:
            token: ${{ secrets.GITHUB_TOKEN }}
            branch: update-flux
            commit-message: Update to ${{ steps.update.outputs.flux_version }}
            title: Update to ${{ steps.update.outputs.flux_version }}
            body: |
              ${{ steps.update.outputs.flux_version }}
```

### End-to-end testing

Example workflow for running Flux in Kubernetes Kind:

```yaml
name: e2e

on:
  push:
    branches:
      - '*'

jobs:
  kubernetes:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v2
      - name: Setup Flux CLI
        uses: fluxcd/flux2/action@main
      - name: Setup Kubernetes Kind
        uses: engineerd/setup-kind@v0.5.0
      - name: Install Flux in Kubernetes Kind
        run: flux install
```

A complete e2e testing workflow is available here
[flux2-kustomize-helm-example](https://github.com/fluxcd/flux2-kustomize-helm-example/blob/main/.github/workflows/e2e.yaml)
