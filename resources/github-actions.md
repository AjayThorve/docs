---
layout: default
nav_order: 3
parent: Resources
grand_parent: Maintainer Docs
title: GitHub Actions
---

# {{ page.title }}
{:.no_toc}

## Overview
{:.no_toc}

The RAPIDS team is in the process of migrating from Jenkins to GitHub Actions for CI/CD. The page below outlines some helpful information pertaining to the implementation of GitHub Actions provided by the RAPIDS Ops team. The official GitHub documentation for GitHub Actions is also useful and can be viewed [here](https://docs.github.com/en/actions).

### Intended audience
{: .no_toc }

Operations
{: .label .label-purple}

Developers
{: .label .label-green}

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Implementation

The RAPIDS Ops team provides GPU enabled self-hosted runners for use with GitHub Actions to the RAPIDS and other select GitHub organizations.

To ensure proper usage of these GPU enabled CI machines, the RAPIDS Ops team has adopted a strategy known as _Marking code as trusted by pushing upstream_ which is described in [this CircleCI blog post](https://circleci.com/blog/triggering-trusted-ci-jobs-on-untrusted-forks/).

The gist of the strategy is that the source code from trusted pull requests can be copied to a prefixed branch (e.g. `pull-request/<PR_NUMBER>`) within the source repository and CI can be configured to test only those prefixed branches rather than the pull requests themselves.

Pull requests authored by members of the given GitHub organization are considered trusted and therefore are copied to a `pull-request/*` branch for testing automatically.

Pull requests from authors outside of the GitHub organization must first be reviewed by a repository member with `write` permissions (or greater) to ensure that the code changes are legitimate and benign. That reviewer must leave an `/ok to test` (or `/okay to test`) comment on the pull request before its code is copied to a `pull-request/*` branch for testing.

The `/ok to test` comment is only valid for a single commit. Subsequent commits must be re-reviewed and validated with another `/ok to test` comment.

### Ignoring Pull Request Branches in `git`

One consequence of the strategy described above is that a lot of `pull-request/*` branches will be created and deleted in GitHub as pull requests are opened and closed. To avoid having these branches fetched locally, you can run the following `git config` command, where `upstream` in `remote.upstream.fetch` is the `git` remote name corresponding to the source repository:

```sh
git config \
  --global \
  --add "remote.upstream.fetch" \
  '^refs/heads/pull-request/*'
```

Note that this `git` configuration option requires `git` version `2.29` or greater to support negative refspecs ([source](https://github.blog/2020-10-19-git-2-29-released/#user-content-negative-refspecs)).

### Downloading CI Artifacts

For NVIDIA employees with VPN access, artifacts from both pull-requests and branch builds can be accessed on [https://downloads.rapids.ai/](https://downloads.rapids.ai/).

There is a link provided at the end of every C++ and Python build job where the build artifacts for that particular workflow run can be accessed.

![](/assets/images/downloads.png)

### Skipping CI for Commits

See the GitHub Actions documentation page below on how to prevent GitHub Actions from running on certain commits. This is useful for preventing GitHub Actions from running on pull requests that are not fully complete. This also helps preserve the finite GPU resources provided by the RAPIDS Ops team.

With GitHub Actions, it is not possible to configure all commits for a pull request to be skipped. It must be specified at the commit level.

**Link**: [https://docs.github.com/en/actions/managing-workflow-runs/skipping-workflow-runs](https://docs.github.com/en/actions/managing-workflow-runs/skipping-workflow-runs)

### Rerunning Failed GitHub Actions

See the GitHub Actions documentation page below on how to rerun failed workflows. In addition to rerunning an entire workflow, GitHub Actions also provides the ability to rerun only the failed jobs in a workflow.

At this time there are no alternative ways to rerun tests with GitHub Actions beyond what is described in the documentation (e.g. there is no `rerun tests` comment for GitHub Actions).

**Link**: [https://docs.github.com/en/actions/managing-workflow-runs/re-running-workflows-and-jobs](https://docs.github.com/en/actions/managing-workflow-runs/re-running-workflows-and-jobs)

## Self-Hosted Runners

The RAPIDS Ops team provides a set of self-hosted runners that can be used in GitHub Action workflows throughout supported organizations. The tables below outline the labels that can be utilized and their related specifications.

### CPU Labels

The CPU labeled runners are backed by various EC2 instances and do not have any GPUs installed.

| Label               | EC2 Machine Type            |
| ------------------- | --------------------------- |
| `linux-amd64-cpu4`  | `m5d.xlarge` <sub>1</sub>   |
| `linux-amd64-cpu8`  | `m5d.2xlarge` <sub>1</sub>  |
| `linux-amd64-cpu16` | `m5d.4xlarge` <sub>1</sub>  |
| `linux-arm64-cpu4`  | `m6gd.xlarge` <sub>2</sub>  |
| `linux-arm64-cpu8`  | `m6gd.2xlarge` <sub>2</sub> |
| `linux-arm64-cpu16` | `m6gd.4xlarge` <sub>2</sub> |

Additional specifications:

1. [https://aws.amazon.com/ec2/instance-types/m5/](https://aws.amazon.com/ec2/instance-types/m5/)
2. [https://aws.amazon.com/ec2/instance-types/m6g/](https://aws.amazon.com/ec2/instance-types/m6g/)

The CPU label names consist of the following components:

```text
linux-amd64-cpu4
^     ^     ^  ^
|     |     |  |
|     |     |  CPU Core Count
|     |     CPU Designator
|     Architecture
Operating System
```

### GPU Labels

{% assign earliest_driver_version = "470" %}
{% assign latest_driver_version = "525" %}

The GPU labeled runners are backed by lab machines and have the GPUs specified in the table below installed.

**IMPORTANT**: GPU jobs have two requirements. If these requirements aren't met, the GitHub Actions job will fail. See the _Usage_ section below for an example.

1. They must run in a container (i.e. `nvidia/cuda:11.8.0-base-ubuntu22.04`)
2. They must set the {% raw %}`NVIDIA_VISIBLE_DEVICES: ${{ env.NVIDIA_VISIBLE_DEVICES }}`{% endraw %} container environment variable

Due to our limited GPU capacity and the overhead associated with manually rotating self-hosted runner labels when GPU drivers are updated, there are no driver-specific self-hosted runner labels (e.g. `linux-amd64-gpu-t4-525-1`).

Instead, the driver-version designators `earliest` and `latest` are used. The values of these designators represent the GPU driver version that RAPIDS uses for testing.

The chart below will be kept up-to-date with the corresponding driver versions at any given time.

Supported organizations will be notified whenever these versions are scheduled to be updated.

{% include gpu-labels-table.html %}

The GPU label names consist of the following components:

```text
linux-amd64-gpu-t4-latest-1
^     ^     ^   ^  ^      ^
|     |     |   |  |      |
|     |     |   |  |      Number of GPUs Available
|     |     |   |  GPU Driver Version
|     |     |   GPU Type
|     |     GPU Designator
|     Architecture
Operating System
```

### Usage

The code snippet below shows how the labels above may be utilized in a GitHub Action workflow.

```yaml
name: Test Self Hosted Runners
on: push
jobs:
  job1_cpu:
    runs-on: linux-amd64-cpu8
    steps:
      - name: hello
        run: echo "hello"
  job2_gpu:
    runs-on: linux-amd64-gpu-v100-latest-1
    container: # GPU jobs must run in a container
      image: nvidia/cuda:11.8.0-base-ubuntu22.04
      env:
        NVIDIA_VISIBLE_DEVICES: {% raw %}${{ env.NVIDIA_VISIBLE_DEVICES }}{% endraw %} # GPU jobs must set this container env variable
    steps:
      - name: hello
        run: |
          echo "hello"
          nvidia-smi
```

For additional details on self-hosted runner usage, see the official GitHub Action documentation page here: [https://docs.github.com/en/actions/hosting-your-own-runners/using-self-hosted-runners-in-a-workflow](https://docs.github.com/en/actions/hosting-your-own-runners/using-self-hosted-runners-in-a-workflow)
