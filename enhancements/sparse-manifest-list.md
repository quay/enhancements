---
title: Sparse Manifest List Support
authors:
  - "@jonathankingfc"
reviewers:
  - "@dmage"
  - "@bcaton"
approvers:
  - TBD
  - "@oscardoe"
creation-date: 2023-12-21
last-updated: 2024-01-04
status: provisional
---

# Quay Container Registry Sparse Manifest List Support

This enhancement proposes the addition of sparse manifest list support to the Quay container registry. The goal is to enable more efficient storage and retrieval of container images, especially in environments where network bandwidth or storage efficiency is a concern. By implementing sparse manifest lists, Quay will be able to more effectively manage container images composed of multiple layers, optimizing storage and reducing data transfer requirements.

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Open Questions [optional]

> 1. What are the potential impacts on Quay's existing storage backend when implementing sparse manifest lists?
> 2. How will this enhancement affect existing Quay users, and what migration steps are necessary?
> 3. What changes will be required in client software to be compatible with handling sparse manifest lists? This includes Podman and OC Mirror

## Summary

The introduction of sparse manifest list support in Quay container registry aims to increase the efficiency of how container images are stored and managed. This enhancement focuses on optimizing the use of storage space and minimizing the data transfer when pulling images from the registry. The feature will be particularly beneficial for users with large image repositories or those operating in bandwidth-constrained environments.

## Motivation

With the growing size of container images and the increasing need for efficiency in container registry operations, there's a clear need for more advanced storage solutions. Sparse manifest lists offer a compelling approach to address these challenges. This enhancement seeks to leverage these benefits to improve Quay's functionality and user experience. This will also help reduce costs on Openshift in supporting OCP installations in various architectures.

### Goals

- Implement sparse manifest list support in Quay container registry.
- Optimize storage utilization and data transfer efficiency.
- Ensure backward compatibility and minimal disruption for current users.
- Ensure that existing clients are able to handle pulling these images, and that other clients that do not support this feature necessarily are not broken

### Non-Goals

- Overhaul of Quay's existing storage architecture.
- Disruption of existing workflows for current Quay users.

## Proposal

The proposal includes developing and integrating sparse manifest list support into the Quay container registry. This will involve changes in how Quay stores and handles container images, focusing on layer deduplication and efficient data management.

### User Stories [optional]

#### Story 1

A user with a large repository of container images can significantly reduce their storage footprint by using sparse manifest lists, as common layers across different images are stored only once.

#### Story 2

In a bandwidth-constrained environment, a user can pull images from Quay more efficiently, as the sparse manifest list allows downloading only the necessary layers, reducing the data transfer volume.

### Implementation Details/Notes/Constraints [optional]

Key considerations for the implementation include ensuring that the new feature integrates smoothly with Quay's existing architecture and that it maintains compatibility with existing OCI image specifications.

### Risks and Mitigations

The primary risks involve potential incompatibilities with existing storage backends and client tools. Mitigation strategies include thorough testing, backward compatibility checks, and clear documentation for users about any new requirements or changes in behavior.

## Design Details

### Test Plan

- Unit and integration tests for new features.
- E2E tests simulating real-world usage scenarios.
- Performance benchmarking to measure improvements in storage efficiency and data transfer.

### Graduation Criteria

- Successful implementation and testing in a dev preview environment.
- Positive feedback from initial user testing.
- Documentation and tutorials for users to leverage the new feature.

### API Design

The API endpoints will remain largely unchanged, with modifications primarily in the internal handling of image manifests and layers.

### Upgrade / Downgrade Strategy

A clear migration path will be provided for users to upgrade to the new version of Quay with sparse manifest list support. For downgrade, users will be able to revert to their previous version with standard rollback procedures.

### Version Skew Strategy

Compatibility with different versions of Quay and client tools will be maintained. Any potential issues will be identified and addressed during the testing phase.

## Implementation History

- 2024-01-10: Proposal for adding sparse manifest list support to Quay.

## Drawbacks

The complexity of implementing sparse manifest lists might pose challenges, especially in ensuring compatibility with existing setups.

## Alternatives

An alternative could be to enhance existing image layer compression methods, although this would not
