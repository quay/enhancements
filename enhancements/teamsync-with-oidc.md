---
title: Team synchronization via OIDC Configuration
authors:
  - "@Sunandadadi"
reviewers:
  - "@dmage"
  - "@syahmed"
approvers:
  - "@dmage"
  - "@syahmed"
creation-date: 2024-01-10
last-updated: 2024-01-10
status: implementable
---

# Team synchronization via OIDC Configuration

## Release Signoff Checklist

- [X] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Open Questions [optional]

1. When team synchronization is disabled from the web UI, do we remove members(synced through OIDC group) from the team or leave it at current state.

## Summary

Quay currently supports team synchronization with LDAP groups to allow flexible permission assignments to user groups defined in LDAP. However, lately there is an increased adaption towards OIDC(OpenID Connect)-based identity providers.
This feature enables users to leverage an OIDC provider that supports user groups to synchronize them with teams in Quay.

## Motivation

Support external OIDC Authentication that can be plugged into Quay registry along with managing quay teams externally through the identity provider.

### Goals

* Admins can set up an OIDC Authentication system in Quay's config tool
* Admins can set up team synchronization configuration (like enabling team sync support, sync interval) in Quay's config tool
* Admin or user(if permitted) can enable team synchronization and configure an OIDC group to sync with, from the Teams page in the UI
* User's teams are synchronised with OIDC groups only after the user logs into Quay
* Teams that are synced with OIDC groups, are regularly updated based on the configured `resycn_interval`

### Non-Goals

* User's teams cannot sync with OIDC groups if the user does not log into quay.
* Team memberships cannot be edited from the UI if team synchronization is enabled.

## Design Details

The workflow of the current proposal revolves around 4 components of Quay:
1. Config Tool
2. UI
3. Login flow
4. TeamSync Worker

**Config Tool**

In the Quay Config tool, configuration to connect with the OIDC system can be set up. If `LDAP` or `OIDC` is selected as the external authentication systems, there is an option to choose team Synchronization using the configured provider.
Other team Synchronization details like resynchronization duration, allowing non-super to enable and mange team syncing can be configured in the tool as well. If the feature `FEATURE_TEAM_SYNCING` is set to `True` and external
authentication is set to `LDAP` or `OIDC` and is successfully validated, the below workflows kicks in.

**UI**

When the user visits a team page at `/organization/<orgname>/teams/<teamname>`, a user can enable or disable team synchronization from the UI. If team synchronization is enabled, additional details of the OIDC group that needs to be synced with the team is entered.
Once the group details are configured and verified, the team page enters into a read-only state since the team memberships are now controlled through the OIDC server.

The flowchart for the UI workflow looks as depicted below:
![](https://github.com/quay/quay/assets/11522230/89ee41de-9be5-4fd0-a462-8cd54504648f)

**Login flow**

This is triggered when the user logins to Quay. Quay's `/signin` page is redirected to the OIDC authentication system. Once a user logins to Quay successfully, a new entry is created for the user in the `FederatedLogin` table. This table
keeps track of all the users in Quay who are signed in through an external authentication system. The user's OIDC groups are fetched from the `/userinfo` endpoint of the configured OIDC server. For each group a user
belongs to, if an entry for the group exists in the `TeamSync` table (i.e., team is syncronized with OIDC group) and if user is not already part of the Quay team, the user gets added to the team.

The flowchart for the Login workflow looks as depicted below:
![](https://github.com/quay/quay/assets/11522230/c0946ac1-67ee-4a59-abe0-bb4b18576fe8)

**TeamSync Worker**

TeamSync Worker is responsible for keeping the OIDC groups and corresponding quay teams synchronized. It runs at the configured `resycn_interval`. The worker finds stale teams i.e., teams from `TeamSync` table where their last synchronization is more than `resycn_interval`.
For each team, `FederatedLogin` users part of the team are fetched. `last_updated` column in `FederatedLogin` table will track when the user's information was last fetched. If the last updated time is more than `resycn_interval`, a new API call to
`/userinfo` endpoint of the OIDC server is made to fetch user details. This information is then used to update the teams that a user is part of. Lastly, user is removed from all the teams from `TeamSync` table that a user is a part of, but the team does not exist in users OIDC groups.

The flowchart for the TeamSync Worker looks as depicted below:
![](https://github.com/quay/quay/assets/11522230/d9e4e3a3-3eb2-45f2-96eb-0c2b1090cd2a)

### Implementation Details

**Database schema change to add `last_updated` column to `FederatedLogin` table**

OIDC does not have an endpoint to fetch users that belong to a group. Instead, the `/userinfo` endpoint is used to fetch the groups a user belongs to. When the TeamSync worker runs, in order to make sure that the teams are synced with the OIDC groups, a users OIDC groups
are fetched using the `/userinfo` endpoint, and the user's quay teams membership is updated. When a user belongs to multiple OIDC groups, `/userinfo` endpoint is initiated multiple times. This is prevented with the `last_updated` column
which keeps track of the last time users details were updated. A call is only made when `timedelta(current_time, last_updated) > sync_interval`.

| Field | Datatype | Attributes | Description |
| --- | ----------- | ----------- | ----------- |
| last_updated | DateTimeField | nullable | Last time the user's information was fetched from OIDC system |

**Endpoints in OIDC**

*Authorization endpoint*

According to Oauth Authorization code flow, the authorization server sends a temporary (authorization) code to the client. This code is exchanged for a token. This is a secure flow for web applications with a backend that can store credentials securely. The authorization code flow is depicted below:
![](https://github.com/quay/enhancements/assets/11522230/9967a80b-77ec-44c4-a5d6-6a0667a8392c)
Picture credits: https://cloudentity.com/developers/basics/oauth-grant-types/authorization-code-flow/

```
  GET /authorize?
    response_type=code%20id_token
    &client_id=<client_id>
    &redirect_uri=<redirect_uri>
    &scope=openid profile email
    &state=af0ifjsldkj
```
`state`: Opaque value used to maintain state between the request and the callback.

Example response:
```
  HTTP/1.1 302 Found
  Location: https://server.example.com:8020/oidcclient/redirect/client01
    code=SplxlOBeZQQYbYS6WxSbIA
    &state=af0ifjsldkj
```

*Token endpoint*

The token endpoint is initiated by the client to obtain `ID token`, `access token` and `refresh token`. This endpoint accepts an authorization_code issued to the client by the authorization endpoint. When the authorization code is validated, appropriate tokens returned in response.

```
  POST /token
    Content-Type: application/x-www-form-urlencoded
    Authorization: Basic auth

    grant_type=authorization_code&code=<authorization_code>
    &redirect_uri=<redirect_uri>
```

Example response:
```
  HTTP/1.1 200 OK
    Content-Type: application/json
    {
     "access_token": "SlAV32hkKG",
     "token_type": "Bearer",
     "refresh_token": "8xLOxBtZp8",
     "expires_in": 3600,
     "id_token": "eyJ ... zcifQ.ewo ... NzAKfQ.ggW8h ... Mzqg"
    }
```

*UserInfo endpoint*

The UserInfo endpoint is a protected resource that returns claims about a user and is authenticated through an `access_token`. The response of the request is returned as a JSON object.

```
  GET /userinfo
    Accept: application/x-www-form-urlencoded
    access_token=<access_token>
```

Example response:

```
  HTTP/1.1 200 OK
    Content-Type: application/json
   {
    "sub"          : "bob",
    "groupIds"     : [ "bobsdepartment","administrators" ],
    "given_name"   : "Bob",
    "name"         : "Bob Smith",
    "email"        : "bob@mycompany.com",
    "phone_number" : "+1 (604) 555-1234;ext5678"
   }
```

### Constraints

* OIDC groups are not synced until the worker runs and the team and user's `last_updated > sync_interval`.
* If a user is deleted from OIDC system, the users team membership is updated but the user still persists in Quay.

### Risks and Mitigations

* Additional API calls are required to the OIDC server to authenticate and fetch user details.

### Test Plan

* TBD

### Implementation History

* 2023-01-10 Initial proposal
