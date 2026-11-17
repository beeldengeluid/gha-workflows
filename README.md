# gha-workflows

Shared workflows for the beeldengeluid GitHub organization.

## Maven OpenShift deployments

Two reusable workflows build a Maven project, build and push its Docker images to ECR with [Docker Bake](https://docs.docker.com/build/bake/), and update the image tags in [k8s-workloads](https://github.com/beeldengeluid/k8s-workloads).

| Workflow | What it does | Deploys to |
|---|---|---|
| [`maven-openshift-tst-deployment.yml`](.github/workflows/maven-openshift-tst-deployment.yml) | `mvn deploy` to CodeArtifact, then builds and pushes the images | `tst` |
| [`maven-openshift-release-deployment.yml`](.github/workflows/maven-openshift-release-deployment.yml) | Maven release (`release:prepare`, `release:perform`) and the release PR, then builds and pushes the images from the released code | the `environments` input, default `acc, prd` |

`maven-deploy.yml` and `maven-release.yml` stay as they are for projects that build their images with a plain Docker loop.

### Usage

```yaml
jobs:
  tst-deployment:
    uses: beeldengeluid/gha-workflows/.github/workflows/maven-openshift-tst-deployment.yml@<version>
    secrets: inherit
    permissions:
      actions: write
      contents: write
      id-token: write
      pull-requests: write
```

```yaml
jobs:
  release-deployment:
    if: github.ref == 'refs/heads/main'
    uses: beeldengeluid/gha-workflows/.github/workflows/maven-openshift-release-deployment.yml@<version>
    with:
      environments: acc, prd
    secrets: inherit
    permissions:
      actions: write
      contents: write
      id-token: write
      pull-requests: write
```

| Input | Default | Workflow |
|---|---|---|
| `time-zone` | `Europe/Amsterdam` | both |
| `timeout-minutes` | `10` | both |
| `environments` | `acc, prd` | release |
| `working-directory` | `.` | test, tst and release: folder of the Maven project, for a repository with the project in a subfolder (see below) |
| `k8s-workloads` | `true` | both: set `false` to only push the images, without updating k8s-workloads |

The workflows use the organization secrets `GH_USER`, `GH_PAT` and `APP_PRIVATE_KEY`, and the variables `BG_APP_ID` and `MAVEN_CODE_ARTIFACT_SETTINGS`.

### Image tag

The workflow passes `IMAGE_TAG` to the bake file:

| Workflow | `IMAGE_TAG` | Example |
|---|---|---|
| tst | Maven project version, plus `-<short sha>` for SNAPSHOT versions | `1.4.0-SNAPSHOT-abc1234` |
| release | Released Maven project version | `1.4.0` |

### docker-bake.hcl

A repository can describe its images in a `docker-bake.hcl` in its root. The file must declare `IMAGE_TAG` and have a `default` group with the targets to build:

```hcl
variable "IMAGE_TAG" {}

group "default" { targets = ["my-service-api"] }

target "my-service-api" {
  context = "./my-service-api"
  tags    = ["917951871879.dkr.ecr.eu-west-1.amazonaws.com/my-service/api:${IMAGE_TAG}"]
}
```

This is the file the action generates for a repository `my-service` with a Dockerfile in `my-service-api/`.

Without a `docker-bake.hcl`, the [`docker-bake-file`](.github/actions/docker-bake-file/action.yml) action generates one, with a target for every folder that has a Dockerfile (Dockerfiles under `target/` are skipped):

| Dockerfile in | Image |
|---|---|
| the repository root | `<ecr>/<repository>` |
| a folder | `<ecr>/<repository>/<folder>`, where a `<repository>-` prefix is dropped from the folder: `musicweb-fallback-api` gives `musicweb-fallback/api` |

The ECR repositories must exist before the first push.

### Deployment in k8s-workloads

For every pushed image, the workflow runs [`update-maven-projects-image-tag.yml`](https://github.com/beeldengeluid/k8s-workloads/blob/main/.github/workflows/update-maven-projects-image-tag.yml) in k8s-workloads with `image_name` (the image without the tag) and `image_tag`. That workflow sets `newTag` on the `images` entry whose `newName` is `image_name`, in `<repository>/<environment>/kustomization.yml`, and merges the change in a pull request. So:

- each image needs its own ECR repository, and
- each environment's `kustomization.yml` needs an `images` entry with that `newName`.

### Maven project in a subfolder

Set `working-directory` when the `pom.xml` is not in the repository root:

```yaml
jobs:
  tst-deployment:
    uses: beeldengeluid/gha-workflows/.github/workflows/maven-openshift-tst-deployment.yml@<version>
    with:
      working-directory: my-project
      k8s-workloads: false     # do not update image tags in k8s-workloads
    secrets: inherit
```

All Maven steps run in that folder, and `.java-version` and `pom.xml` are read from it. The folder can have its own
`docker-bake.hcl` or, like a project in the root, gets a generated one by the `docker-bake-file` action. Bake runs from the
folder, so the `context` and `dockerfile` paths in the file are relative to it. For a release, the bake file is read from
the released code in `<working-directory>/target/checkout/<working-directory>`.

For a release, `release:prepare` needs an `<scm>` section in the project's pom. In a repository with more than one released
project, give each a distinct `tagNameFormat` in the release plugin configuration, for example `my-project-@{project.version}`.

### Changing the docker-bake-file action

Both workflows pin the action to a commit SHA. After changing [`action.yml`](.github/actions/docker-bake-file/action.yml), commit it first, then update the `uses:` line in both workflows to that commit.

## Releasing

The [Release Workflows](https://github.com/beeldengeluid/gha-workflows/actions/workflows/release-workflows.yml) workflow can be used to create releases. It can be started manually, but is also scheduled to run at the end of every day. Version numbers follow [Calender Versioning](https://calver.org/).
