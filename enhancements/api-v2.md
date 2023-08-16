---
title: Quay API V2 design
authors:
  - "@sdadi"
reviewers:
  - "@syahmed"  
  - "@sleesinc"
  - "@obulatov"
  - "@bcaton"
approvers:
  - "@syahmed"  
  - "@sleesinc"
  - "@obulatov"
  - "@bcaton"
creation-date: 2023-07-24
last-updated: yyyy-mm-dd
status: implementable
---

# Quay API v2

This is a proposal to design a generic strategy to improve Quay API's to support pagination, filtering and sorting.

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Summary

Currently, only a few APIs have the option to paginate results on the backend. The goal of the proposal is to have a re-usable system to perform actions on the backend and send the curated result on the frontend. Ideally frontend should only display the results and allow the backend to perform
all the heavy lifting.

## Motivation

In the quay ui, filtering, sorting and occasionally paginating the results happen on the frontend. 
This leads to high rendering time on the frontend and thereby a poor user experience.

### Goals

Implement a generic system that is consistent across different endpoints to support the following operations on Quay endpoints:
- pagination
- filtering
- sorting

## Design Details

### API Design

- Proposed path of the endpoints: In an attempt to not change the existing endpoints, new APIs will be under `/api/v2_draft/`.
  `v2_draft` will be replaced with `v2` when ready to release. 
- Existing decorators like the scope of API, page parameters will be reused where ever required.

### Pagination

It is difficult to say that one pagination strategy suits all needs. After understanding the pros and cons of various pagination strategies, 
cusor-based pagination seems to work best for our current needs.  

Cursor will contain a unique sequential db column to base cursors on along with additional data when required.
For example, sorting results on non-unique keys, like datetime (currently used in the UI for sorting usage logs)

Cursor data is extensible and can look like:
```
{"id": 123}
{"id": "2023-08-02T12:34:56Z", "offset": 20}
{"id": "2023-08-02T12:34:56Z", "some_other_key": val1, "another_key": val2}
```

The following will be expected request parameters for an API that is to be paginated:
1. limit: number of items to be returned for the page, max limit set to 100 items per page (based from current Quay API guidelines).
2. prev_cursor/next_cursor

    a. prev_cursor: encrypted string to fetch previous page results

    b. next_cursor: encrypted string to fetch next page results

Response body:
```
{
  results: [items per page],
  total_count: count of all items on all pages,
  page_info: { 
    next_cursor: encrypted metadata for next page
    prev_cursor: encrypted metadata for prev page
  }
}
```

### Filtering

Typical syntax of an API filter will look like: `key1[operation1]=<value1>&key2[operation2]=<value2>` where:
- key: is the field on which the search will be performed (Eg: repo name, org name, etc)
- operation: can be different based on the datatype of the key. (Eg: for integers, `lt|lte|gt|gte`, for strings: `like|eq|regex`)
- value: is what the query will filter for
- multiple supported filters can be added to the query parameters using `&`

Eg: (GET `/tags?tag_name[like]=quay`, GET `/logs?created_at[lte]=2023-07-20 19:56`)

### Sorting

Sort order will be represented by the query parameter `sort`, ascending order by `+` and descending order by `-`.
Multiple sorts can be passed in the url by adding comma separated keys.
Eg: (GET `/tags?sort=+created_at`, GET `/users?sort=+creation_date,-last_accessed`)

Sorting will be supported only on select keys. Every API endpoint will maintain a list of supported sortable keys. If a user
makes a request with an unsupported key, 400 Bad Request response will be sent along with a list of supported sortable keys.
Example of 400 response:
```
{
    "title": "Unsupported sort key",
    "detail": "The request is sortable only on one of these keys: [a, b, c]",
    "status": 400,
}
```

### Notes/Constraints

The major caveat of cursor-based pagination is that we cannot fetch results from page number. User cannot directly navigate to the nth page.
They'd need to request pages from 1 - n as next page results are derived from the previous page's cursor.

### Graduation Criteria

##### Dev Preview -> Tech Preview

- Ability to utilize the enhancement end to end
- Sufficient test coverage
- End user documentation, relative API stability

##### Tech Preview -> GA 

- More testing (on scale)
- Sufficient time for feedback
- Available by default

## Open questions

> 1. Do we want to set expiration time on cursors? If yes, how to choose this time?

