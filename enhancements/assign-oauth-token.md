---
title: Assigning Oauth Tokens by Organization Owners
authors:
  - "@bcaton85"
reviewers:
  - "@sleesinc"
  - "@dmage"
  - "@syed"
approvers:
  - "@sleesinc"
  - "@dmage"
  - "@syed"
creation-date: 2024-16-05
last-updated: 2024-16-05
status: proposed
---

# Oauth Token Assignment 

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Summary

Organization owners can create OAuth tokens and the tokens are assigned to the token creator. When the token is created for and used by some organization member, the action is logged to the token creator, in this case the organization owner. In restricted environments, where only dedicated registry administrators are organization owners, this is undesirable due to inaccurate auditing. For accuracy, token ownership should be mutable and can be reassigned by a superuser.

## Motivation

This will allow assignment of oauth tokens so the correct user will appear in the usage logs when authenticating with that token.

### Goals

* Organization owners are able to assign the oauth token to a user besides themselves

### Non-Goals

* Creation of the oauth application view in the React UI
* Allowing the organization owner to revoke those assigned tokens
* This will only apply to the token oauth flow for API authentication, not the code oauth flow


## Proposal

### Selecting a user through the UI

In the application view when creating a token there will be a new entity select field that will allow them to select a user to assign the token. This field will only allow users to be selected, not robots or organizations. Any user within the Quay registry can be selected for assignment.

[select-user-image]

When a different user is chosen, the button will change to `Assign token` and a confirmation modal listing the scopes and assigned user will appear on button click. On confirmation the `POST /oauth/authorize/assignuser` endpoint will be called.


### The `/oauth/authorize/assignuser` API

A new endpoint will be created to handle token assignment. These parameters would match those that would normally go to the `/oauth/authorize` endpoint in the normal oauth flow. These parameters will be stored in a new table `oauthtokenassignment` which will be used by the assigned user to go through the oauth flow themselves. The `/oauth/authorize/assignuser` endpoint is only accessible by organization owners.

```
POST /oauth/authorize/assignuser

Query parameters:
client_id: Application client ID that this token assignment belongs too
response_type: Which grant to execute (code, token). As of now token will only be supported
redirect_uri: Where to redirect after authorization
scope: Which permissions are being granted
username: User that the authorization is being assigned to
```

A new table **oauthtokenassignment** will be created to store the information:
| Field | Datatype | Attributes | Description|
| --- | ----------- | ----------- | ----------- |
| uuid | text (length=255)| unique, not null, indexed | The UUID of the token assignment |
| assigned_user_id | integer| FK, Not unique, not null, indexed | User that the authorization is being assigned to |
| application_id | integer | FK, Not unique, not null, indexed | Application ID that the token is being created on behalf of |
| response_type | text (length=255)|  text, not null | Which grant to execute (code, token). As of now token will only be supported |
| redirect_url | text (length=255) | text, null | Where to redirect after authorization |
| scope | text (length=255) | text, not null | Which permissions are being granted |

The API will execute the following psuedocode:
- Begin execution
- Does the application exist? If not return 404
- Does the assigned user exist? If not return a 404
- Is the user creating the assignment an org owner of the org that the application belongs to? If not return a 403
- Create an entry in the `oauthtokenassignment` table indicating that the user needs to go through the oauth token flow to generate a token
- End execution

### Authorizing the assigned user

At this point an authorization and therefore token hasn't been created for the assigned user, they need to go through the auth flow themselves.

A new API will be created to fetch the authorizations that have currently been assigned to the user. This API can only be called by the current logged in user. The information will be fetched from the `oauthtokenassignment` and `oauthapplication` tables.
```
GET /v1/user/assignedauthorizations

response:
{
  authorizations: [
    {
        application: {
            name: name of the application
            clientId: client ID of the application
            description: description of the application,
            url: URL of the application
            avatar: avatar of the application
            organization: {
                name: name of the organization
                avatar: avatar of the organization
            },
        },
        uuid: UUID for this authorization assignment
        redirectUri: redirect URI for this authorization assignment
        scopes: the scopes being granted to the assigned user
        responseType: The response type for the oauth flow, in this case will always be "token"
    },
    {...},
    {...}
  ]
}
```

Currently in the Angular UI under User settings > External logins and authorizations > Authorized Applications there exists the list of authorized applications on behalf of the user. Within this same view a call will be made to `GET /v1/user/assignedauthorizations` that will list the assigned authorizations within the same table. Each field will be the same as the other entries with the exception of a new column `Approve` which will give the option to `Authorize application` for each assigned authorization row.

[authorize-app-via-assigned-user-image]

Selecting that option brings them through the normal app authorization flow. If they are being granted scopes that are greater than they currently have, a screen will appear confirming the scopes begin granted. A depication of this flow can be seen below with new functionality being highlighted in yellow.

[oauth-flow]

When selecting `Authorize application` this flow will execute with the addition of the `assignment_uuid` parameter. This indicates to the oauth flow that the token is being created as a part of an authorization assignment. Both the `/oauth/authorize` and `/oauth/authorizeapp` endpoints will check whether the current user is an admin and if not verify the current user exists in the `oauthtokenassignment` table matching the given `assignment_uuid`. If this succeeds, when the token is generated it will also delete the row in the `oauthtokenassignment` table matching the `assignment_uuid` as the token has been succesfully assigned and the user shouldn't be able to generate another token. Screen shots of the flow can be found below.

Confirming the scopes after clicking `Authorize application`:
[scope confirmation]

Displays the token once:
[token-display]

On user or application deletion the corresponding rows in the `oauthtokenassignment` will also be deleted.

### Constraints

* This will currently only work for the token oauth flow
* Only the assigned user will be able to revoke the token, the org owner will not have access to revoke the token themselves. Global superusers will still have access to revoke the token.

### Risks and Mitigations

* This requires changes to Quay's permissions system. Due diligence is required to ensure no permissions are inadvertantly applied.

### Test Plan

* TBD

### Implementation History

* 2024-16-05 Initial proposal
