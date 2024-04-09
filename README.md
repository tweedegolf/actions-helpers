# Helpers for GitHub Actions and Docker Containers
This repository contains a set of helper scripts for working with containers in
GitHub Actions.

## Creating a container image
You can use the `container-image.yml` workflow to create a new container image.

## Cleanup of old tags
You can use the `container-tag-cleanup.yml` workflow to cleanup old tags.

## Cleanup of untagged container images
You can use the `container-untagged-cleanup.yml` workflow to cleanup any
container images that no longer have any associated tags with them.
