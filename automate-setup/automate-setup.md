# Automate Setup

## Problem

At the moment, a completely hands-off setup of Quay post-install is difficult if not impossible. A typical automated setup would include creating users, teams, namespaces, etc. To do any of this, the first user must be created and the bearer token obtained for use with API calls.

## Solution

Create an unauthenticated API endpoint that will create the first user in the database, returning credentials for both pushing images as well as the bearer token for API use.

## Implementation Details

### Feature Flag

Add a new feature `FEATURE_USER_INITIALIZE` which defaults to false.

### API

`/api/v1/user/initialize'

Will create a new user iff there are no users in the database.

Input `name`, `password`, `email`, and `access_token`. The first three are as described in regular user create API. The `access_token` is unique in that it is the scope of permissions to create an access token for this user.

The scope may be one or more of the following in a single string separated by spaces:

org:admin - Administer Organization
This application will be able to administer your organizations including creating robots, creating teams, adjusting team membership, and changing billing settings. You should have absolute trust in the requesting application before granting this permission.

repo:admin - Administer Repositories
This application will have administrator access to all repositories to which the granting user or robot account has access

repo:create - Create Repositories
This application will be able to create repositories in to any namespaces that the granting user or robot account is allowed to create repositories
View all visible repositories
This application will be able to view and pull all repositories visible to the granting user or robot account

repo:read, repo:write - Read/Write to any accessible repositories
This application will be able to view, push and pull to all repositories to which the granting user or robot account has write access

super:user - Super User Access
This application will be able to administer your installation including managing users, managing organizations and other features found in the superuser panel. You should have absolute trust in the requesting application before granting this permission.

user:admin - Administer User
This application will be able to administer your account including creating robots and granting them permissions to your repositories. You should have absolute trust in the requesting application before granting this permission.

user:read - Read User Information
This application will be able to read user information such as username and email address.

For example:
```
{
    "username": "admin",
    "password": "changeme",
    "email": "admin@quay.example.org",
    "access_token": "org:admin repo:admin repo:create repo:read repo:write super:user user:admin user:read",
}
```

Calling the API will return a status of 400 with a message indicating problem, if the user creation is not successful.

For example:
```
{"message": "Cannot initialize user in a non-empty database"}
```

A successful call will return the created user and credentials.

For example:
```
{
    "username": "admin",
    "email": "admin@quay.example.org",
    "encrypted_password": "QmPdFZdvfjA0JvnNPEdIZhtKzBjQmXgqQvabEXDYNL0ucHmUXhlPEUzgMThMEN6V",
    "access_token": "6WQPRJZ5HHC6G8L2T5TM9MCGBSIX0AFJXKVVDYOH",
}
```

The `encrypted_password` may be used to login with podman or docker:
```
podman login -u admin -p QmPdFZdvfjA0JvnNPEdIZhtKzBjQmXgqQvabEXDYNL0ucHmUXhlPEUzgMThMEN6V quay.example.org
```

The `access_token` may be used with the API:
```
curl -k -X POST -H "Content-Type: application/json" -H "Authentication: Bearer 6WQPRJZ5HHC6G8L2T5TM9MCGBSIX0AFJXKVVDYOH" --data '@org.json' https://quay.example.org/api/v1/organization/
```
