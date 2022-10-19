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

## Motivation

The [referrers API](https://github.com/opencontainers/distribution-spec/blob/main/spec.md#listing-referrers)
connects OCI Artifacts with OCI Images. We add OCI Artifact support in
preparation for fully supporting the referrers API.

## Goals

- Support the new OCI Artifact manifest type
- OCI Artifacts are first-class citizens of Quay, in other words, we do not
reuse the `OCIManifest` class

### Non-Goals

- Implement support for the OCI referrers API (will come in a future enhancement)
- Implement references between OCI Artifacts and OCI Images on the database
level (also coming in a future enhancement)

## Proposal

The `OCIManifest` Python class is an implementation of the OCI Image Manifest
Specification. With the introduction of the OCI Artifact Manifest, this type
needs to have a more specific name as to not be source of confusion for
developers unfamiliar with the code base.

Along with renaming `OCIManifest` we also intrudoce a new `OCIArtifactManifest`
class, with OCI Artifact specific fields.

### User Stories [optional]

#### As an artifact developer I want to successfully push and pull OCI Artifacts to Quay

OCI Artifacts are identified by the `application/vnd.oci.artifact.manifest.v1+json`
media type. They are also composed of different fields than OCI Images, thus
Quay should treat them differently when parsing.

#### As a Quay user I want artifacts in my repository to not get GC'd

Image manifests that do not point to a tag and are not connected to another
manifest (via `ManifestChild` table, used for manifest lists) are garbage
collected in Quay.

Artifact manifests do not require tagging.
Initially, artifact manifests will not be directly connected to Image manifests,
so keeping them from being GC'd might prove challenging.

The [OCI Distribution Specification](https://github.com/opencontainers/distribution-spec/blob/main/spec.md)
suggests that for [backwards compatibility](https://github.com/opencontainers/distribution-spec/blob/main/spec.md#backwards-compatibility)
a special [tag schema](https://github.com/opencontainers/distribution-spec/blob/main/spec.md#referrers-tag-schema)
be used.

Until Quay is able to keep untagged artifact manifests around Quay can tag
artifact manifests using this tag schema.

Once relationships between artifact manifests and image manifests are implemented
on a database level, the GC can be made aware of that and Quay will no longer
need to rely on the special tag schema to keep artifacts around.

Are OCI Artifacts tagged?
