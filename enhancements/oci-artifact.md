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
- Support OCI Artifacts as first-class citizens, in other words, we do not
reuse the `OCIManifest` class
- Implement references between OCI Artifacts and OCI Images on a database level

### Non-Goals

- Implement support for the OCI referrers API (will come in a future enhancement)
- Solve GC for OCI Images not referenced by any tag

## Proposal

The `OCIManifest` Python class is an implementation of the OCI Image Manifest
Specification. With the introduction of the OCI Artifact Manifest, this type
needs to have a more specific name as to not be source of confusion for
developers unfamiliar with the code base.

Along with renaming `OCIManifest` we also intrudoce a new `OCIArtifactManifest`
class, with OCI Artifact specific fields.

To connect an OCI Artifact manifest with an OCI Image manifest this proposal
suggests reusing the `ManifestChild` table instead of creating a new one.
The `ManifestChild` table currently stores references between a manifest list
or OCI Image Index with its sub-manifests (children).
The idea is to repurpose this table, turning it into a generic way to connect
two manifests, whatever their type (i.e image manifest or artifact manifest).

### User Stories [optional]

#### As an artifact developer I want to push and pull OCI Artifacts to Quay

OCI Artifacts are identified by the `application/vnd.oci.artifact.manifest.v1+json`
media type. They are also composed of different fields than OCI Images, thus
Quay should treat them differently when parsing.

#### As a Quay user I want the Quay UI to show which artifacts are connected to my images

To enable future use by the referrers API, the connection between an OCI
Artifact and an OCI Image (expressed via the `subect` property) must be
expressed in the database level.

The UI can consume this information via Quay v1 API.

#### As a Quay user I want artifacts in my repository to not get GC'd

Image manifests that do not point to a tag and are not connected to another
manifest (via `ManifestChild` table, used for manifest lists) are garbage
collected in Quay.

Artifact manifests do not require tagging. Without any changes Quay would
GC Artifact manifests after the time machine period passes.

This story requires the following changes to Quay's GC:
 * Teach the GC to not garbage collect artifacts that were not deleted
 * Teach the GC about the relationship between OCI Image manifests and OCI
 Artifact manifests (via `subject` property) so that it does not GC OCI
 Artifats that have a relationship to an OCI Image
 * Teach the GC that it should GC OCI Artifacts connected to an OCI Image that
 is about to be GC'd

### Risks and Mitigations

#### Reusing `ManifestChild` table

Although reusing the `ManifestChild` table comes with a lot of advantages, it
is not without risks.
Because this table is exclusevily used to express relationships between a
manifest list (or OCI Image Index) and its sub-manifests (children), there may
be unforeseen effects to both Quay's UI and code-base.
The impact on proxy orgs is also currently unknown.
