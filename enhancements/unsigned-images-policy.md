---
title: Repository Policies - Blocking Pull of Images Not Signed by Cosign
authors:
  - "@bcaton85"
reviewers:
  - 
approvers:
  - 
creation-date: 
last-updated: 
status: provisional
---

> NOTE!: This proposal has an existing pull request. The pull request is referenced through out the document. Referrals to the pull request will be removed before the merge.

# Repository Policies - Blocking Pull of Images Not Signed by Cosign

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Open Questions [optional]

* Are the risks listed in the `Risks and Mitigations` section permissible? 

## Summary

Quay plays a crucial role in the software supply chain, providing a centralized location to store and manage container images. To enhance the security of this supply chain, this proposal introduces a feature that allows users to block pulling unsigned images from Quay.

This feature ensures that only images with cosign signatures are allowed to be pulled and deployed. Unsigned images are considered potentially insecure as they may have been tampered with or originate from untrusted sources. By blocking the use of unsigned images, this security measure helps prevent the deployment of potentially compromised or malicious container images, therefore reducing the risk of security vulnerabilities and ensuring the integrity of the software supply chain.

## Motivation

Ensuring the presence of signatures has typically landed on clients to prevent any compromised images from being deployed. This creates more toil for DevOps engineers and introduces security risks since the verification of signatures follows a opt-in model. By ensuring a signature is present we remove the need for clients to verify the presence of a signature themselves.

It is still required by users to verify the integrity of the signature against the public key to ensure there has been no tampering with the image.

### Goals

* Provide a configurable option at the repository level to block the pull of images that do not have an accompanied cosign signature
* Implementation is capable of large scale use

### Non-Goals

* We will not verify the signature against a provided public key
    - Possibly done by requiring a public key to be provided before hand and verifying the cosign signature against the public key on push

## Proposal

The following implementations will be required:
- A database table for storing the repository policy
- Endpoints for fetching and updating the repository policy
- A UI for managing the policy
- Modifying the GET manifest endpoints to check and block unsigned images

**Database Schema**

A [new table](https://github.com/quay/quay/pull/2547/files#diff-d588c567d195a7b1147193f07d333dde5e546e7d9b0e29fef616caf4ea79b5f6R16-R35) will be required for storing the repository policy configuration.

repositorypolicy
| Field | Datatype | Attributes | Description|
| --- | ----------- | ----------- | ----------- |
| repository_id | integer | FK, not null, indexed | Repository that the policy falls under |
| policy | text | JSON, not null | Policy configuration |

`policy` schema:
```
{
    "blockUnsignedImages": (true|false)
}
```

The schema is intentionaly left generic, as additional policy configurations can be added in the future. There exists a one to one relationship with the `repository` table. 

A row is created when ever a repository policy is fetched or updated. The row is deleted when the corresponding row in the `repository` table is deleted.

**Repository Policy API**

Fetching the repository policy
```
GET /api/v1/repository/{organization}/{repository}/policy
```

This [endpoint](https://github.com/quay/quay/pull/2547/files#diff-626392107d09ca88d20d64309f17b10f1615dda8af1a9c28d3470f8d70bb13d2R462-R473) will fetch the current repository policy configuration. If a row does not exist in the database for this repository, one will be created with the default value of `{"blockUnsignedImages": false}` and returned.

Updating the repository policy
```
PUT /api/v1/repository/{organization}/{repository}/policy
{
    "blockUnsignedImages": true
}
```

This [endpoint](https://github.com/quay/quay/pull/2547/files#diff-626392107d09ca88d20d64309f17b10f1615dda8af1a9c28d3470f8d70bb13d2R475-R493) will update the policy with the provided configuration. If a row does not exist in the database for this repository one will be created with the given policy.

Deletion of the row created by the endpoints is done on the deletion of the corresponding row in the `repository` table.

**Blocking Unsigned Images**

Blocking will occur during the GET of the manifest since this is the first step in the image pull process. This requires modifying the `/v2/{organization}/{repository}/manifests/{tagname}` and `/v2/{organization}/{repository}/manifests/{digest}` endpoints. These endpoints recieve the most amount of traffic so optimization is crucial. This feature will incur two reads to each endpoint: one to lookup the policy and the other to lookup the cosign manifest.

The following workflow will run within the endpoint:
1. [Perform the following conditional checks](https://github.com/quay/quay/pull/2547/files#diff-2487e826aa015b45ecc8a34cbf59ea7bccf235b8d51bf732fd574cf94d11eeedR115-R121):
1. [Check whether blocking of unsigned images has been enabled](https://github.com/quay/quay/pull/2547/files#diff-993d350f368114dfb413baaf31b18ee042ff3f60c3d19388718402e548c3fc1eR41-R50)
    1. Return false if `FEATURE_BLOCK_UNSIGNED_IMAGES` is not enabled
    1. Fetch the repository policy from the database
    1. If no policy exists return false
    1. If a policy does exist return the boolean value of `blockUnsignedImages`
1. [Check whether the block of the unsigned image should be skipped](https://github.com/quay/quay/pull/2547/files#diff-2487e826aa015b45ecc8a34cbf59ea7bccf235b8d51bf732fd574cf94d11eeedR491-R506)
    1. Return true if `cosign` is in the user agent header
        - For cosign to be able to sign images it needs to fetch the manifest being signed
    1. Return true if the media type of a layer is `application/vnd.dev.cosign.simplesigning.v1+json`
        - Always allow the signature image to be pulled
    1. If no other condition met, return false
1. [Check if the image has been signed](https://github.com/quay/quay/pull/2547/files#diff-2b6073b755b708898592c062c907050a1b5052b354e97e87e8ec93fb9c5c4582R522-R543)
    1. Fetch the manifest referenced by the tag `sha256-{digest}.sig`
        - This is how cosign identifies signature images
    1. If this tag and manifest do not exist, return false
    1. Load the manifest and return true if there exists the `application/vnd.dev.cosign.simplesigning.v1+json` media type layer
    1. If no other conditions met, return false
1.[ Based on the previous conditions; if blocking of unsigned images has been enabled, should not be skipped, and the image is unsigned throw a `no cosign signature found` error](https://github.com/quay/quay/pull/2547/files#diff-2487e826aa015b45ecc8a34cbf59ea7bccf235b8d51bf732fd574cf94d11eeedR115-R121)

### Constraints

* 

### Risks and Mitigations

* This implemenation does not verify the signature, which is out of scope for this enhancement. This requires users to verify the signature themselves
* Cosign requires reading the unsigned manifest. This has been resolved by [checking for cosign in the user agent header](https://github.com/quay/quay/pull/2547/files#diff-2487e826aa015b45ecc8a34cbf59ea7bccf235b8d51bf732fd574cf94d11eeedR493-R494). This does allow for pulling the unsigned image by modifying the user agent header. 
* The block would also be skipped if there [exists a layer that contains the `application/vnd.dev.cosign.simplesigning.v1+json` media type](https://github.com/quay/quay/pull/2547/files#diff-2487e826aa015b45ecc8a34cbf59ea7bccf235b8d51bf732fd574cf94d11eeedR497-R504). This is required to be able to pull verify the signature image.

### Test Plan

* TBD

### Implementation History

* 2023-09-07 Initial proposal
