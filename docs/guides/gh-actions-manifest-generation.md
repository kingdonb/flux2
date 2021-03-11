# GitHub Actions Manifest Generation

This guide shows how to handle a basic use-case of manifest generation with GitHub Actions, using third-party tools in CI to run pre-processing that prepares YAML manifests for deployment, and then commit to a deploy branch in git for kustomize-controller to apply them.

## Primary Uses of Flux

Flux's primary use case is to apply YAML manifests from the latest `Revision` in an `Artifact`. For a `GitRepository` source, this is the latest commit in a branch or tag ref. The use cases in this tutorial for manifest generation assume that the CI is given write access to one or more Git repos. This can be complicated to set up. In GitHub Actions though, for a single repo this is automatic without additional configuration wherever Actions are enabled. In the third example, since there is more than a single repository involved, an additional Personal Access Token is used as well.

Flux v2 can not be configured to call out to arbitrary binaries that a user might supply with an `InitContainer`, as it was sometimes done in Flux v1. It is assumed if users want to run anything besides vanilla `Kustomize` and `envsubst` with respect to manifest generation, that it will be done through CI on GitHub, or some similar adaptation of the methods shown below for other CI providers.

### Use Cases of Manifest Generation

There are several use cases presented. We're going to see a few different ways how to add commits and push updates to a branch through GitHub Actions.

One example is doing a simple in-place string substitution using `sed -i` with a variable that is made available by the source host, in CI.

Then, a basic example of Docker Build and Push shows two strategies that can be used for tagging and versioning images.

The next example is using a third-party tool, `jsonnet`, to do what may be colloquially referred to as "YAML rehydration".

Finally we show how a Personal Access Token can enable commits across repositories. This can also be used to replicate the best nearest approximation of Flux's "deploy latest image" feature of yesteryore, without invoking Flux v1's expensive and redundant image pull requirements to get access to build metadata. This is done by leverage of CI to commit and push a subfolder from an application repository into a separate deploy branch of plain YAML manifests for `Kustomization` to apply, that can be pushed to any branch on any separate repository to which CI is granted write access.

* [String Substitution with `sed -i`](#string-substitution-with-sed-i)
* [Docker Build and Tag with Version](#docker-build-and-tag-with-version)
* [Jsonnet for YAML Document Rehydration](#jsonnet-for-yaml-document-rehydration)
* [Commit Across Repositories Workflow](#commit-across-repositories-workflow)

The examples below assume no prior familiarity with GitHub Actions. Everything that is needed for a working functioning deployment of an application with its own routines for manifest generation to be connected with Flux is shown.

#### String Substitution with `sed -i`

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


### Jsonnet for YAML Document Rehydration



### Commit Across Repositories Workflow




