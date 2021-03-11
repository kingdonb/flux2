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
    While this example updates any branch (`branches: ['*']`) from CI, each `Kustomization` in Flux only deploys manifests from **one branch or tag** at a time. This is only the first example.

This workflow example could go into your application's repository, wherever images are built from – then each CI build triggers a commit writing back the `GIT_SHA` to a configmap in the branch, and an example `deployment` YAML manifest that could be deployed with a `Kustomization`, or just as easily using `kubectl apply -f k8s.yml`, since it doesn't take advantage of any other Kustomize features or `envsubst`. We will show each of those ideas in succession below. Take these and adapt them to your use case, it may be possible to get this done with some really basic tools like `sed`.

This example borrows a [Prepare step](https://github.com/fluxcd/kustomize-controller/blob/5da1fc043db4a1dc9fd3cf824adc8841b56c2fcd/.github/workflows/release.yml#L17-L25) from Kustomize Controller's own release workflow, to enable easy migration from or to `GIT_SHA`-based tags and a newer alternative, the `image-automation-controller`'s preferred Timestamp-based formatting for [Sortable image tags](/guides/sortable-image-tags).

```yaml
# ./.github/workflows/manifest-generate.yaml
name: Manifest Generate
on:
  push:
    branches:
    - '*'

jobs:
  run:
    name: Push Git Update
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

Skipping over some hard parts, first see this easy shell script that performs an in-place string substitution with `sed`.

```yaml
# excerpted from above - run a shell script
- name: Update manifests
  run: ./update-k8s.sh $GITHUB_SHA
```

```
# update-k8s.sh
#!/bin/bash

set -feuo pipefail

GIT_SHA=${1:0:8}
sed -i "s|image: kingdonb/any-old-app:.*|image: kingdonb/any-old-app:$GIT_SHA|" k8s.yml
sed -i "s|GIT_SHA: .*|GIT_SHA: $GIT_SHA|" flux-config/app-version-configmap.yaml
```

When CI runs the `update-k8s.sh` script, it passes in the full value of `GITHUB_SHA` which is trimmed in the script to only the first 8 characters. Then, the script runs `sed -i` twice, updating `k8s.yml` and `flux-config/app-version-configmap.yaml` which are given as examples here below.

```
apiVersion: v1
data:
  GIT_SHA: 4f314627
kind: ConfigMap
metadata:
  creationTimestamp: null
  name: any-old-app-version
  namespace: devl
```


### Docker Build and Tag with Version



### Jsonnet for YAML Document Rehydration



### Commit Across Repositories Workflow




