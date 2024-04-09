# Helpers for GitHub Actions and Docker Containers
This repository contains a set of helper scripts for working with containers in
GitHub Actions.

## Creating a container image
You can use the `container-image.yml` workflow to create a new container image.
This workflow makes sure that the CI machine is setup for multi-arch, such that
you can build a docker container for multiple architectures. It also logs in to
the GitHub container registry allowing you to upload an image right away. To use
it, add a job to your GitHub workflow calling this workflow:

```yaml
# ...

jobs:

  # ...

  docker:
    uses: "tweedegolf/actions-container-helpers/.github/workflows/container-image.yml@main"
    with:
        push: ${{ github.ref == 'refs/heads/main' }}
        platforms: "linux/amd64,linux/arm64"
        tags: ghcr.io/tweedegolf/example:latest

  # ...
```

### Workflow inputs
This workflow has several parameters that allow customizing the behavior:

<table>
  <thead>
    <tr><th>Option</th><th>Required</th><th>Default</th></tr>
  </thead>
  <tbody>
    <tr>
      <td><code>tags</code></td>
      <td>yes</td>
    </tr>
    <tr><td colspan="3">
      List of image tags (newline separated) for the resulting container image
    </td></tr>
    <tr>
      <td><code>push</code></td>
      <td>no</td>
      <td><code>false</code></td>
    </tr>
    <tr><td colspan="3">
      If true, the image will be pushed to the registry
    </td></tr>
    <tr>
      <td><code>platforms</code></td>
      <td>no</td>
      <td><code>""</code></td>
    </tr>
    <tr><td colspan="3">
      Comma separated list of platforms to build the image for, i.e. <code>linux/amd64,linux/arm64</code>. If left empty, will only build for the native platform.
    </td></tr>
    <tr>
      <td><code>context</code></td>
      <td>no</td>
      <td><code>.</code></td>
    </tr>
    <tr><td colspan="3">
      Context directory for the container image build
    </td></tr>
    <tr>
      <td><code>build-args</code></td>
      <td>no</td>
      <td><code>""</code></td>
    </tr>
    <tr><td colspan="3">
      List of build arguments (newline separated) to be inserted in the container image build
    </td></tr>
    <tr>
      <td><code>file</code></td>
      <td>no</td>
      <td><code>Dockerfile</code></td>
    </tr>
    <tr><td colspan="3">
      Name of the dockerfile to build
    </td></tr>
  </tbody>
</table>

### Example usage
Below, you will find a working example where we either just build (and not push)
a container image on a pull request, and we fully build and push a container on
the main branch, these two require separate permissions. We'll also add a build
matrix to allow multiple images to be generated:

* `.github/workflows/docker.yml`
  ```yaml
  name: Docker

  on:
    workflow_call:

  jobs:
    docker:
      strategy:
        matrix:
          include:
            - version: trixie
              latest: false
              alt: testing
            - version: bookworm
              latest: true
              alt: stable
            - version: bullseye
              latest: false
              alt: oldstable
      uses: "tweedegolf/actions-container-helpers/.github/workflows/container-image.yml@main"
      with:
          push: ${{ github.ref == 'refs/heads/main' }}
          platforms: "linux/amd64,linux/arm64"
          build-args: |
              DEBIAN_VERSION=${{matrix.version}}
          tags: |
              ghcr.io/tweedegolf/debian:${{matrix.version}}
              ghcr.io/tweedegolf/debian:${{matrix.alt}}
              ${{ matrix.latest && 'ghcr.io/tweedegolf/debian:latest' || '' }}
  ```
  This file will be re-used by the two workflows below. As such we only trigger
  it on `workflow_call`. Note how we use a matrix to build multiple images. Of
  course for this workflow to run we'll also need a Dockerfile to be built, but
  that has been omitted in this example.
* `.github/workflows/build-push.yml`
  ```yaml
  name: Build and push

  permissions:
    contents: read
    packages: write

  on:
    push:
      branches:
        - main
    schedule:
      - cron: '15 2 * * SUN'

  jobs:
    build-and-push:
      uses: ./.github/workflows/docker.yml
  ```
  This build and push workflow for the main branch also runs on a schedule every
  week to keep the image up to date. Note how we require package write
  permission in this workflow.

* `.github/workflows/check.yml`
  ```yaml
  name: Checks

  permissions:
    contents: read

  on:
    pull_request:

  jobs:
    build:
      uses: ./.github/workflows/docker.yml
      secrets: inherit
  ```
  The checks workflow will just need read permissions for the repository with
  no write permissions required. We only run it for a pull request.


## Cleanup of old data in the container registry
The GitHub container registry does not automatically remove old tags or untagged
images from itself. To do this, we can setup an actions workflow that does this
for us automatically. There are two steps to this process. First, we remove any
tags that are no longer needed by adding any number of jobs refering to the
`container-tag-cleanup.yml` workflow. Finally we finish by adding a single
`container-untagged-cleanup.yml` workflow. Of course, if your image creation
workflow never creates any new tags but instead always reuses the existing ones
there is no need to include any of the tag cleanup workflows. Let's look at an
example:

```yaml
name: Container Registry Cleanup

permissions:
  contents: read
  packages: write

on:
  schedule:
    - cron: '15 2 * * MON'

jobs:
  old-nightly-cleanup:
    uses: "tweedegolf/actions-container-helpers/.github/workflows/container-tag-cleanup.yml@main"
    with:
      package: debian
      filter: "^nightly-\d{2}-\d{2}-\d{4}$"
      keep_n: 5
  untagged-cleanup:
    needs: [old-nightly-cleanup]
    uses: "tweedegolf/actions-container-helpers/.github/workflows/container-untagged-cleanup.yml@main"
    with:
        package: debian
```

In this example, we run a cleanup job that removes old nightly tagged images
first, and then removes any remaining untagged images after that. Note how this
action is only run on a schedule once a week, there is no need to run it after
every main branch update or every pull request unless lots of container builds
are happening on your repository. You might have additional jobs that run the
tag cleanup workflow if there are multiple patterns you wish to match, but we
always make sure that the untagged cleanup runs after all other jobs have
finished to make sure we capture any additional untagged images from the other
cleanup jobs.

### Tag cleanup workflow inputs
This workflow has several parameters that allow customizing the behavior:

<table>
  <thead>
    <tr><th>Option</th><th>Required</th><th>Default</th></tr>
  </thead>
  <tbody>
    <tr>
      <td><code>package</code></td>
      <td>yes</td>
    </tr>
    <tr><td colspan="3">
      Name of the container registry package
    </td></tr>
    <tr>
      <td><code>filter</code></td>
      <td>no</td>
      <td><code>""</code></td>
    </tr>
    <tr><td colspan="3">
      A regular expression matching some specific tags for the container
    </td></tr>
    <tr>
      <td><code>keep_filter</code></td>
      <td>no</td>
      <td><code>""</code></td>
    </tr>
    <tr><td colspan="3">
      A regular expression matching any tags that should not be removed
    </td></tr>
    <tr>
      <td><code>keep_n</code></td>
      <td>no</td>
      <td><code>-1</code></td>
    </tr>
    <tr><td colspan="3">
      How many of the most recently created and matched tags to keep (if
      non-negative).
    </td></tr>
    <tr>
      <td><code>older_than</code></td>
      <td>no</td>
      <td><code>-1</code></td>
    </tr>
    <tr><td colspan="3">
      Only remove untagged container images at least as old as this (in days),
      only works if larger than zero.
    </td></tr>
  </tbody>
</table>

### Untagged images cleanup workflow inputs
This workflow has several parameters that allow customizing the behavior:

<table>
  <thead>
    <tr><th>Option</th><th>Required</th><th>Default</th></tr>
  </thead>
  <tbody>
    <tr>
      <td><code>package</code></td>
      <td>yes</td>
    </tr>
    <tr><td colspan="3">
      Name of the container registry package
    </td></tr>
    <tr>
      <td><code>older_than</code></td>
      <td>no</td>
      <td><code>-1</code></td>
    </tr>
    <tr><td colspan="3">
      Only remove untagged container images at least as old as this (in days),
      only works if larger than zero.
    </td></tr>
    <tr>
      <td><code>untagged_timestamp_tolerance</code></td>
      <td>no</td>
      <td><code>10000</code></td>
    </tr>
    <tr><td colspan="3">
      Do not remove untagged container images if they have a created timestamp
      this close (in milliseconds) to a tagged container image. We do this
      because multi-arch docker containers will be pushed as separate untagged
      images to the container registry and GitHub cannot recognize them as part
      of a tagged image. Setting this option large enough prevents accidental
      deletion of one of these images.
    </td></tr>
  </tbody>
</table>
