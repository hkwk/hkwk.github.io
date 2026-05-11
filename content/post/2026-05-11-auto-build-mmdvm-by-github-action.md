---
author: HKL
categories:
- Default
date: "2026-05-11T22:42:00+08:00"
slug: auto-build-mmdvm-by-github-action
draft: false
tags:
- Programming
- Radio
title: Auto Build MMDVMhost And DMRGateway by Github Action
---

This post introduces my projects for automatically building `MMDVMHost` and `DMRGateway` with GitHub Actions.

## Why automated builds?

Building radio gateway software from source can be time-consuming and brittle. Each release may require specific dependencies, toolchains, and build options. For users and maintainers, the build process is easier to manage if it is automated and produces ready-to-download artifacts.

## Project repositories

- `https://github.com/BG7KUH/MMDVMHost-Builds`
- `https://github.com/BG7KUH/DMRGateway-Builds`

Each repository contains a GitHub Actions workflow that checks out the corresponding upstream source, installs the required build dependencies, compiles the project, and publishes binary artifacts.

## What the workflows do

The GitHub Actions workflows are designed to:

- build the latest source code automatically
- run on push or release events
- install dependencies in a clean runner environment
- compile the project with the correct options
- package the resulting binaries for download

This makes it simple to obtain the latest `MMDVMHost` and `DMRGateway` builds without needing to build locally.

## Benefits

- reproducible builds in GitHub-hosted runners
- no need to maintain a local toolchain
- easy binary distribution through GitHub releases
- clear separation between the source project and build automation

## How to use

Visit each repository and check the latest workflow run or release assets. The builds are published as artifacts for easy download, so users can focus on deploying the binaries rather than on building them.

## Future plans

I intend to keep both workflows up to date with upstream changes, improve dependency handling, and support additional build platforms if needed.

If you are interested in automated build workflows for amateur radio software, these repositories show a practical example of using GitHub Actions to build and deliver `MMDVMHost` and `DMRGateway` reliably.

