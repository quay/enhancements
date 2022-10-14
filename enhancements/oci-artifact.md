---
title: OCI Artifact support
authors:
  - "@fmissi"
  - "@bcaton"
reviewers:
  - "@obulatov"
approvers:
creation-date: 2022-10-14
last-updated: 2022-10-14
status: implementable
---

# OCI Artifact Manifest Support

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA


## Summary

An OCI Artifact is a way of defining and storing content addressable artifacts
in a registry alongside the images the artifacts relate to.

Quay currently supports multiple media types for OCI Manifests. These existing
media types currently reuse the `OCIManifest` Python class. This class
implements the [OCI Image Manifest specification](https://github.com/opencontainers/image-spec/blob/main/manifest.md).

OCI Artifacts manifests and OCI Image manifests are similar, but key properties
differentiate them.

OCI Artifact manifests and OCI Image manifests **share** the following properties:

 * `mediaType`
 * `subject`
 * `annotations`

The following properties are **absent** from the OCI Artifact manifest specification:

 * `schemaVersion`
 * `layers`

The following properties are **unique** to the OCI Artifact manifest specification
(and not present in the OCI Image manifest specification):

 * `artifactType`
 * `blobs`

This enhancement proposal designs a plan to support OCI Artifacts independently
of OCI Images.
