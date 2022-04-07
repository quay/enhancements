# Proposal to integrate Tekton pipelines for running Quay builds

---
title: Proposal to integrate Tekton pipelines for running Quay builds 
authors:
  - "@hgovinda"
reviewers:
  - "@kleesc"
  - "@bcaton"
approvers:
  - "@kleesc"
  - "@bcaton"
creation-date: 2022-02-08 
last-updated: 2022-02-08 
status: 
---

# 

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Open Questions

> 1. Since Tekton project has APIs readily available for golang (and no support for Python), would using quay operator to run tekton pipelines be a feasible option in the long run given that the eventual goal is to take away build orchestration responsibility from Quay


## Summary

The current Quay build architecture provides two options for running Quay builds: AWS EC2 executor(upstream) and Kubernetes Job executor (only for Redhat customers). Running builds on AWS cluster or on virtualized platforms poses challenges such as machine/resource restrictions, availability of bare metal clusters etc.  
This proposal introduces an additional build option to leverage Tekton pipelines to run quay builds on Openshift cluster. Tekton provides CRDs like [Task](https://tekton.dev/docs/pipelines/tasks/#overview) obj which defines a series of `Steps` to execute a build. The output from the build logs are sent to the Build Manager

### Goals

- Builds are able to run using Tekton Pipelines


### Non-Goals

- Multi-arch builds are not yet supported


## Design Details

The proposed solution involves changes to Quay's Build Manager. There will be a provision for an additional build executor to run builds via Tekton. 

Changes in Quay: 
- Extend the [BuilderExecutor](https://github.com/quay/quay/blob/master/buildman/manager/executor.py#L88) to include a new TektonExecutor. 
- Since Tekton has no out-of-the-box APIs available for python, [openshift-client-python](https://github.com/openshift/openshift-client-python) is used to create tekton resources.
- Install Tekton on a cluster using the openshift-client which deploys Tekton controller in a pod. 
- Define `Task` obj with required build arguments (from the build config file) as [spec.params](https://tekton.dev/docs/pipelines/tasks/#specifying-parameters). 

```Task
apiVersion: tekton.dev/v1beta1
kind: Task
...
spec:
  inputs:
    resources:   # can optionally provide PipelineResources for specifying build inputs (provides build args separation)
      - name: docker-source
        type: git
        ...
    params:
      - name: pathToDockerFile
        type: string
        ...
    steps:
      - name: build-and-push
        image: 
        env:
        command:
         - /bin/sh
        args:
          ...
      
```

- Tekton runs each `Task` obj in the form of a Kubernetes pod.
- A `TaskRun` is an instance of `Task` obj and is responisble for executing the `Task`. Additionally you can also include `ServiceAccount` which has a `Secret` for storing registry credentials.
- Once the image is built, the logs are fetched from the `TaskRun` obj which gives status and logs of steps taken by a `Task` obj.

### Other options
- Alternatively, Tekton hub provides out-of-the-box support for building sources using [Buildah](https://hub.tekton.dev/tekton/task/buildah). When deployed, it would suffice to have a `TaskRun` do the required build.
```
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  name: buildah-build-my-repo
spec:
  taskRef:
    name: buildah
  params:
  - name: IMAGE
    value: <fetched from build arg>
    ...
  workspaces:
  - name: source
    persistentVolumeClaim:
      claimName: my-source
```
- ToDo: Investigate `PipelineRun` to run multiple `Task` jobs in parallel


### Constraints

- Running builds in an unprivileged context may cause some commands that were working under the previous build strategy to fail

### Risks and Mitigations

- Tight coupling of Tekton builds to Build Manager to orchestrate builds 
- Might require config changes at host level (as root) in order to support `unprivileged` configuration

## Implementation History

- 2022-02-08 Initial proposal
