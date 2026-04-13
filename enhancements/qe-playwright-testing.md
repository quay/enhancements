---
title: Porting Quay API Tests Upstream as Playwright Tests
authors:
  - "@Marcusk19"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2026-04-13
last-updated: 2026-04-13
status: provisional
see-also:
  - "https://github.com/quay/quay/pull/4919"
---

# Porting Quay API Tests Upstream as Playwright Tests

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Open Questions [optional]

1. **Programmatic OAuth token generation**: Need to verify that Quay's `/oauth/authorize` endpoint accepts a POST without browser interaction. If not, options are:
   - Use the access token from `/api/v1/user/initialize` directly (it has full superuser permissions)
   - Add a test-mode API endpoint for token generation
   - Use Basic auth for API requests instead of OAuth (many endpoints support it)

## Summary

We have ~450 API test cases in the `quay-tests` repo that validate Quay's REST API -- things like creating orgs, managing repos, robot accounts, permissions, image operations, and superuser endpoints. These tests live outside of the main `quay/quay` repo, which means they are invisible during code review, don't run in upstream CI, and can't evolve alongside the code they validate.

This enhancement proposes porting these tests directly into `quay/quay` as Playwright tests, reusing the existing `ApiClient` and test fixture infrastructure already built for quay/quay's Playwright UI tests. The ported tests will live at `web/e2e/api-tests/` with their own Playwright config (no browser, serial execution).

## Motivation

The current approach (PR quay/quay#4919) runs 1 of 6 test files via an external Docker image that packages the Cypress tests and gets pulled into a GitHub Actions job. This is fragile and opaque. We need to move the remaining 5 test files into upstream quay to close the gap.

- **Tests are invisible during code review.** When someone changes an API endpoint in quay/quay, there's no signal that related test coverage exists (or might break).
- **Tests don't run in upstream CI.** The only way they run today is via an external Docker image. This is fragile and opaque.
- **Tests can't evolve with the code.** Contributors can't update tests alongside the code they're changing in the same PR.

### Goals

* API test coverage becomes visible in PRs -- if you change an endpoint, you'll see related tests in the same repo.
* Tests run automatically in upstream CI with no separate container build/publish cycle.
* Failures are easier to debug using standard Playwright test output, traces, and logs.
* The source of truth for API tests moves to `quay/quay`.
* Existing Playwright infrastructure is reused rather than building a parallel test framework.

### Non-Goals

* Replacing or modifying the existing Playwright UI tests.
* Adding new API test coverage beyond what currently exists in `quay-tests`.
* Porting non-API tests (e.g., UI/Cypress tests) from `quay-tests`.

## Proposal

Port the API tests directly into `quay/quay` as Playwright tests.

Why Playwright instead of keeping Cypress:
- quay/quay is already migrating its UI tests from Cypress to Playwright -- this aligns with that direction
- Playwright's `APIRequestContext` supports API-only testing without launching a browser, which is exactly what these tests need
- We can reuse the existing `ApiClient` (80+ methods) and test fixtures already built in quay/quay's Playwright setup

Why not just expand the Docker image approach:
- Tests inside a container can't be reviewed in PRs
- Debugging failures requires pulling the image and inspecting it manually
- It adds a build/publish step that can break independently of the tests themselves
- It keeps the tests in a separate development lifecycle from the code they validate

### Implementation Details/Notes/Constraints

The ported tests will live at `web/e2e/api-tests/` in quay/quay, alongside the existing Playwright UI tests at `web/playwright/`. They'll have their own Playwright config (no browser, serial execution) but share the same helper infrastructure.

The test suites map to different roles and features:

| Suite                   | What It Validates                                                                  |
| ----------------------- | ---------------------------------------------------------------------------------- |
| **Superuser API**       | Full CRUD across all v1 endpoints with superuser privileges                        |
| **Superuser-Exclusive** | Endpoints only superusers can access (service keys, changelog, take ownership)     |
| **Normal User**         | Same operations from a non-privileged user -- verifies 403s on restricted endpoints |
| **Readonly Superuser**  | Read access works, write access is blocked for the readonly superuser role         |
| **Immutability**        | Tag immutability policy enforcement                                                |
| **V2 Registry**         | Docker Registry V2 spec compliance (manifests, tags, catalog)                      |

Each suite runs in its own CI job with the appropriate Quay configuration (e.g., immutability tests need `FEATURE_IMMUTABLE_TAGS: true`, readonly tests need `GLOBAL_READONLY_SUPER_USERS` configured).

**Phase 1 -- Infrastructure + First Suite**

Set up the Playwright API test config, helpers (auth, HTTP client, image operations), and port the first and largest test file (~97 tests). This runs alongside the existing Docker-image job so we can compare results.

**Phase 2 -- Remaining Suites**

Port the other 5 test files. These can be developed in parallel since they all depend only on the shared infrastructure from Phase 1. This covers the remaining ~250 unique tests (after removing duplicates across files).

**Phase 3 -- Cleanup**

Once the Playwright tests are stable, remove the Docker-image-based job from CI and clean up the ported files from quay-tests.

### Risks and Mitigations

* **Risk:** Programmatic OAuth token generation may not work without browser interaction.
  **Mitigation:** Spike this first. Fallback options include using access tokens from `/api/v1/user/initialize` directly or using Basic auth.
* **Risk:** Test execution time increases CI duration.
  **Mitigation:** Each suite runs as a separate CI job via matrix strategy, enabling parallel execution.
* **Risk:** Flaky tests due to timing-sensitive API operations.
  **Mitigation:** Replace `cy.wait()` calls with Playwright's `expect.poll()` pattern (polling with timeout instead of fixed sleeps).

## Design Details

### Test Plan

Each phase is verified by running the Playwright API tests alongside the existing Docker-image Cypress tests and comparing pass/fail counts. Phase 1 Playwright results should match Cypress (minus skipped vulnerability tests). After Phase 2, total Playwright test count should be ~350 (450 minus duplicates minus vulnerability tests). Each phase is delivered as a PR on quay/quay where the `qe-tests` workflow must pass with a CTRF report in PR comments.

### Graduation Criteria

#### Dev Preview

- Phase 1 complete: infrastructure + first suite running in CI alongside existing Docker-image job
- Pass/fail counts match between Playwright and Cypress for the ported suite

#### Tech Preview

- Phases 1 and 2 complete: all 6 suites ported and running in CI
- Total test count ~350 (after deduplication)

#### GA

- Phase 3 complete: Docker-image-based job removed from CI
- All API tests run exclusively via Playwright in upstream CI
- Ported files removed from quay-tests repo

## Implementation History

* 2026-04-13 Initial proposal

## Drawbacks

* Requires maintaining API tests in quay/quay, adding to the review burden for that repo.
* The transition period (Phases 1-2) runs both test approaches in CI, increasing CI resource usage temporarily.

## Alternatives

* **Expand the Docker-image approach**: Run all 6 Cypress test files via the existing Docker image pattern. Rejected because tests remain invisible in PRs, debugging is harder, and it keeps tests in a separate development lifecycle.
* **Keep Cypress but move tests into quay/quay**: Would achieve upstream visibility but conflicts with the ongoing Cypress-to-Playwright migration in quay/quay and would not leverage existing Playwright infrastructure.

## PR Roadmap

| PR | Title | Est. Lines | Est. Tests | Dep |
|----|-------|-----------|------------|-----|
| 1 | API Test Infrastructure (config, fixtures, helpers) | ~400-500 | 3-5 smoke | -- |
| 2 | Organization & User API Tests | ~400-500 | 20-25 | PR 1 |
| 3 | Repository API Tests | ~400-500 | 20-25 | PR 1 |
| 4 | Repository Features (perms, notifs, tags, builds) | ~500-600 | 25-30 | PR 1 |
| 5 | Advanced Features (quotas, proxy cache, mirror, messages) | ~450-550 | 20-25 | PR 1 |
| 6 | Superuser-Exclusive Endpoints | ~500-600 | 40-50 | PR 1 |
| 7 | Permission Boundary Tests (normal user 403s) | ~400-500 | 40-50 | PR 1 |
| 8 | Readonly Superuser Role Tests (3-role auth) | ~400-500 | 30-40 | PR 1 |
| 9 | Immutability API Tests (unique from new_ui) | ~350-450 | ~21 | PR 1 |
| 10 | V2 Registry API Tests (+ V2 client helper) | ~600 | 30-40 | PR 1 |
| 11 | CI Workflow Integration (qe-tests.yaml) | ~100-150 | 0 | PR 1 |

**Dependency graph:** PRs 2-10 can be developed in parallel after PR 1 merges. PR 11 ideally after PRs 2-3.

**Architecture:** Reuse existing `ApiClient` (81 methods) from `web/playwright/utils/api/client.ts` and `TestApi` from `web/playwright/fixtures.ts`. Separate config at `web/e2e/api-tests/` (no browser, serial execution).

**Source location:** All source Cypress tests are in the `quay-tests` repo at `quay-api-tests/cypress/`. No container extraction needed -- reference the files directly:
- Test files: `quay-api-tests/cypress/e2e/quay_api_testing_*.cy.js`
- Custom commands/helpers: `quay-api-tests/cypress/support/commands.js`
- Config: `quay-api-tests/cypress.config.js`
