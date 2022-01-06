---
title: Proposal to Add Build Option Without Bare-metal Constraint
authors:
  - "@bcaton"
reviewers:
  - "@kleesc"
  - "@bcaton"
  - "@harishsurf"
approvers:
  - "@kleesc"
  - "@bcaton"
  - "@harishsurf"
creation-date: 2021-12-03
last-updated: 2022-01-06
status: implementable
---

# remove-bare-metal-build-constraint

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Open Questions

> 1. Will we encounter issues down the line running builds in unpriviliged containers with Podman?
> 1. Does running builds in unprivileged containers give us the build isolation we need?

## Summary

Currently Quay builds run Podman commands in virtual machines launched via Pods. To run builds on virtual platforms this would require enabling nested virtualization, which is not possible in RHEL or OCP. This places the requirement of builds having to run on bare-metal clusters, which are a precious resource. This proposal is to remove the bare-metal constraint required to run builds by adding an additional build option which does not contain the virtual machine layer.

### Goals

- Builds are able to run on virtualized platforms
- Have backwards-compability to run previous build configurations

## Design Details

This solution proposes adding an additional build option to remove the VM layer to permit builds on virtualized platforms. This will take the form of an additional executor added to the build manager. An unprivileged Podman container will be used to run the image build.

This diagram describes the current workflow:

![image](https://user-images.githubusercontent.com/22058392/146998028-60e79958-5ef8-49ea-af07-7c9aaeef938a.png)

The following diagram describes the proposed workflow:

![image](https://user-images.githubusercontent.com/22058392/146998050-32168e74-dbdf-4193-aa10-c28ac52f182a.png)

The changes to the workflow can be described as:
1. The build manager will create the job object.
2. The job object will create a pod using the **quay-builder image**.
    - The **quay-builder image** will contain the **quay-builder binary** and the **Podman service**.
    - This pod will run as unprivileged
3. The **quay-builder binary** will run commands to build the image while also communicating status and retrieving build info from the build manager

A quick POC has been completed where a very basic Dockerfile was submitted through the UI and the build was completed in a virtual environment in an unprivileged container. 

- [Dockerfile producing the **quay-builder image**.](https://github.com/bcaton85/quay-builder/blob/podman-build/Dockerfile.podman.secure)
- [Changes made to Quay to run the new **quay-builder image**.](https://github.com/quay/quay/pull/1034/files)
- Test Dockerfile used:
  ```Dockerfile
  FROM hello-world 

  ENV TEST_VAR=test
  ```

### Constraints

- Running builds in an unprivileged context may cause some commands that were working under the previous build strategy to fail

### Risks and Mitigations

- Changing the current build strategy could cause impact to the performance and reliability of builds
- Running builds directly in a container will not have the same isolation has using virtual machines
- Changing the build environment may cause builds that were previously working to fail
- Difficulty in testing implementation since there are many ways to run Dockerfile builds

## Implementation History

- 2021-12-03 Initial proposal
