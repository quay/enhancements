---
title: Spam Detection and Quarantine
authors:
  - "@HammerMeetNail"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2026-06-12
last-updated: 2026-07-15
status: implementable
---

# Spam Detection and Quarantine

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [x] Graduation criteria for dev preview, tech preview, GA

## Summary

Quay needs an operator-controlled way to find repositories that are being used
for spam or other unintended content, especially in public registry
deployments where users can create accounts and empty repositories freely.

This enhancement centers spam detection and quarantine management in
`quay-service-tool`. The service tool owns classifier configuration, scan
execution, scan history, match records, quarantine state, review workflows, and
redaction workflows. Quay's in-tree changes are limited to the repository
description ingress hook needed to block spam at create/update time and any
small integration points required to let the service tool quarantine or redact
repository content.

Detection uses Bayesian filtering rather than manually maintained keyword
lists. The same classifier family is used for ingress checks and for scheduled
or manually triggered scans, so operators tune one model and action policy
instead of maintaining separate detection paths.

## Motivation

Public registry deployments can accumulate empty or low-value repositories that
exist primarily to host spam descriptions or lure users away from Quay. Manual
review does not scale, and direct deletion is too risky when classifiers may
produce false positives. Administrators need a repeatable scan process,
durable review state, auditability, and a reversible remediation path.

### Goals

* Detect likely spam repositories using Bayesian classification over repository
  descriptions and supporting metadata.
* Use `quay-service-tool` as the operator interface for classifier training,
  threshold configuration, action policy, preview, scan execution, run history,
  quarantine review, and redaction.
* Keep spam-detection configuration and quarantine state out of the Quay
  application database schema.
* Limit service-tool writes to the Quay database to explicit repository
  quarantine, restore, and redaction actions.
* Require scan candidates to have no pushed image content before they can be
  treated as spam matches or opened for quarantine review.
* Support preview from `quay-service-tool` so operators can see which
  repositories would match a classifier and filter before any Quay data is
  changed.
* Support dry-run automated scans that persist service-tool run and match
  history without quarantining or redacting repositories.
* Store scan/run metadata in the service tool so operators can answer which run
  matched a repository and how many repositories were flagged.
* Allow administrators to quarantine, restore, dismiss, or redact suspected
  spam through the service tool.
* Make quarantine reversible by preserving original repository content in
  service-tool-owned quarantine records.
* Apply the Bayesian classifier at repository-description ingress so Quay can
  reject spammy create/update requests when the feature is enabled for
  enforcement.
* Use a read-only database replica for service-tool preview, scanning, and
  historical reporting workflows.
* Keep the feature disabled by default and controlled through configuration.

### Non-Goals

* Adding a new in-tree Quay UI for spam detection.
* Storing spam detection rules, scan history, or quarantine state in new Quay
  application tables.
* Building a general-purpose abuse moderation platform outside repository
  spam detection.
* Guaranteeing that all spam content is detected.
* Replacing existing moderation or abuse-handling processes outside Quay.
* Automatically deleting repositories in the initial implementation.

## Proposal

Add a service-tool-owned spam detection subsystem made up of:

* Bayesian classifier configuration and training data,
* action-policy configuration for preview, scan, quarantine, and redaction,
* scan runner logic that reads repository descriptions from a read-only Quay
  database replica,
* persisted service-tool run and match records,
* persisted service-tool quarantine records,
* review workflows for quarantine, restore, dismiss, and redaction,
* a Quay ingress hook that evaluates repository description create/update
  requests with the configured classifier,
* configuration options for safe rollout and scale control.

`FEATURE_SPAM_DETECTION` controls Quay-side ingress evaluation and enforcement.
Service-tool scan and remediation execution is configured in the service-tool
deployment. Automated dry-run mode defaults to enabled, so initial rollout can
collect matches in service-tool history without changing repository
descriptions.

## Design Details

### Ownership Boundaries

Spam detection state is owned by `quay-service-tool`, not by Quay. The service
tool is responsible for all spam-specific database changes, including
classifier configuration, training examples, scan runs, scan matches,
quarantine records, action history, and redaction history.

The service-tool state database and managed classifier artifacts are durable
operational state and must share a persistent data directory. Production
deployments should mount that directory from persistent storage, include it in
backup and restore procedures, and run a single service-tool replica while a
local SQLite state database is used. The classifier JSON must not be committed
to the service-tool source repository.

The service tool's interaction with the Quay database is intentionally narrow:

* read repository descriptions and supporting repository metadata for
  classification;
* apply an approved quarantine action to repository content;
* restore repository content from service-tool quarantine state;
* permanently redact repository content when an operator chooses redaction.

Quay should not add `spamdetectionrule`, `spamdetectionrun`,
`spamdetectionrunmatch`, or `quarantinedrepository` tables to its application
schema. Quay-side schema changes should be avoided unless the final
implementation requires a minimal integration point that cannot be represented
through existing repository update paths.

### Bayesian Classification

The detection approach should use Bayesian filtering instead of keyword-list
matching. A manually curated keyword list creates an ongoing maintenance burden
and encourages a cat-and-mouse cycle with spam content. A Bayesian classifier
lets operators improve detection by reviewing examples and updating the model
with spam and non-spam labels.

The classifier should evaluate repository descriptions as the primary signal.
For service-tool scans, repository emptiness is a mandatory eligibility check,
not just a weighted classifier feature: a repository must have no pushed image
content at scan time before it can be recorded as a spam match, moved into the
review queue, or quarantined. Hyperlink presence is also a mandatory eligibility
check for both service-tool matches and Quay ingress rejection: a description
without a recognized HTTP or HTTPS hyperlink must not be treated as spam even
when its Bayesian score exceeds the applicable threshold. The implementation
must score long descriptions in bounded, overlapping token windows and use the
highest window score. This prevents a high-confidence spam segment from being
diluted by appending a large amount of link or boilerplate content. The initial
defaults are 128 tokens per window with a 64-token stride; service-tool and Quay
must apply identical values from the artifact feature configuration. The
implementation may include supporting features such as repository name,
namespace/account age, URL count, token frequency, and description length,
provided the classifier can explain which features contributed to a match well
enough for operator review.

Classifier configuration stored by the service tool should include:

| Field | Description |
| --- | --- |
| `uuid` | Stable external identifier for the classifier configuration |
| `name` | Human-readable classifier name |
| `enabled` | Whether the classifier is active for scans and ingress |
| `training_corpus_version` | Version of the spam/ham examples used to train the model |
| `model_snapshot` | Serialized model parameters or a pointer to a stored model artifact |
| `feature_config` | JSON options for tokenization and metadata features |
| `scan_threshold` | Probability or score required to flag a repository during scans |
| `ingress_threshold` | Default ingress threshold used when the classifier is trained or exported outside the active policy |
| `created_at`, `updated_at` | Timestamps |
| `created_by`, `updated_by` | Operator identity where available |

Training examples should also be service-tool owned. At minimum, each example
needs bounded repository text, label (`spam` or `ham`), source
(`manual_review`, `quarantine`, `restore`, `dismiss`, or import), operator
identity where available, and timestamps. The service tool should support model
retraining or model refresh from those examples without requiring a Quay
database migration.

The service tool should support importing a validated classifier JSON artifact
as an immutable base model. It should preserve the imported bytes and checksum,
make the selected artifact immediately available to manual preview and scan
operations, and retain its model snapshot as the base for later versions.
Operator-initiated retraining should combine that base snapshot with all active
service-tool feedback without double-counting feedback from earlier retraining
runs. The newly trained version should become available to manual scans without
an export step. Export is only required to promote a selected version into the
Quay image used for ingress blocking.

The service tool should enforce minimum corpus quality gates before exporting a
classifier artifact for ingress use. At minimum, the corpus must contain
configurable minimum counts for both `spam` and `ham` examples so an
accidentally tiny or one-sided training set cannot silently become a production
artifact. Exported artifacts and service-tool classifier views should surface
basic corpus metrics such as total example count, spam/ham counts, class
balance, training corpus version, and any validation metrics available. When a
validation split or labeled holdout set is available, the service tool should
show precision and recall at export time; when those metrics are unavailable,
the artifact should record that validation metrics were not produced rather
than implying that quality was measured.

The initial implementation should use a fixed, reviewed tokenizer pattern for
both service-tool scans and Quay ingress. Arbitrary operator-supplied regular
expressions should not be accepted in classifier artifacts until the
implementation has a regex safety strategy that is suitable for Quay's request
path.

### Action Rules and Policy

The service tool should be augmented to handle rules for classification and
action. In this design, rules are not keyword detectors stored in Quay. They
are service-tool policy records that determine how classifier output is used.

Policy should include:

* which classifier configuration is active;
* minimum score/probability for scan matches;
* minimum score/probability for ingress rejection, with the active service-tool
  policy treated as the source of truth for generated Quay ingress artifacts;
* scan filters such as namespace scope, repository visibility, repository
  emptiness, and maximum repositories per run, where repository emptiness is
  required for scan matches and quarantine-review eligibility and a maximum of
  `0`/`All` explicitly means an unbounded scan;
* whether automated scans are dry-run only;
* whether matches should only be recorded or also moved into the service-tool
  review queue;
* whether repositories with terminal review records should be rescanned when
  the repository description and active classifier artifact have not changed;
* the standard quarantine notice used by approved quarantine actions, including
  restoration contact instructions, remediation requirements, and expected
  review timelines;
* whether redaction is available and which operator role is allowed to run it.

All policy changes should be made through `quay-service-tool` and recorded in
service-tool audit history. When the ingress threshold changes, the service tool
must generate or export a new versioned classifier artifact before Quay can
enforce the updated threshold.

Hard identifiers should be objective repository or namespace predicates that are
evaluated outside the Bayesian score and stored in policy snapshots. The initial
scan policy must require empty repositories. Additional recommended hard
identifiers for reducing false positives include:

* public visibility, because private repositories are less likely to be useful
  for public lure or search spam;
* repository age or namespace/account age below a configurable threshold, because
  new empty repositories are higher-signal than long-lived empty repositories;
* non-exempt namespace status, so trusted, internal, paid, or operator-managed
  namespaces can be excluded before classification;
* external URL count above a configurable threshold, in addition to the
  mandatory hyperlink-presence check, because spam descriptions commonly
  redirect users away from the registry;
* lack of retained successful push/build activity where audit data is available,
  as a secondary confirmation that the repository has not hosted legitimate
  image content;
* absence of established collaboration signals, such as team-managed ownership
  or multiple non-owner collaborators, where those signals are available without
  expensive per-repository queries.

These additional identifiers should be policy configurable rather than baked
into Quay, and preview should show which hard identifiers each candidate matched
so operators can tune the policy before enabling scan history or quarantine.

### Rule and Policy Management in quay-service-tool

`quay-service-tool` should be the operator interface for configuring spam
detection. It should own the backend resources and React/PatternFly views for
classifier configuration, policy configuration, preview, run history, and
review actions.

This maps to the existing service-tool architecture:

* add a new backend resource module, for example
  `backend/tasks/spam_detection.py`, implemented with Flask-RESTful resources
  like the existing `backend/tasks/banner.py` and `backend/tasks/user.py`
  resources;
* register those resources in `backend/app.py` with paths such as
  `/spam-detection/classifiers`, `/spam-detection/policy`,
  `/spam-detection/preview`, `/spam-detection/runs`,
  `/spam-detection/runs/<uuid>/matches`, and
  `/spam-detection/review`;
* protect the endpoints with the existing `@login_required`,
  `@verify_admin_permissions`, and `@log_response` patterns from
  `backend/utils.py`;
* add a spam-detection-specific role, for example `SPAM_DETECTION_ROLE`, to
  `backend/config/config.yaml`, render it from `backend/app.py` into
  `backend/templates/index.html`, and use it for frontend route visibility;
* add backend permission decorators such as
  `verify_spam_detection_read_permissions` for preview/reporting and
  `verify_spam_detection_write_permissions` for classifier, policy, quarantine,
  restore, dismiss, and redaction actions;
* add a new React route in `frontend/src/app/routes.tsx`; the existing
  `AppLayout` renders route entries into the navigation sidebar and already
  gates visibility by the route's configured Keycloak role;
* add a new React/PatternFly view under `frontend/src/app/SpamDetection/` that
  uses the existing `frontend/src/services/HttpService.tsx` axios client;
* organize the React view around concrete operational tabs or sections:
  `Classifier`, `Policy`, `Preview`, `Runs`, and `Review Queue`;
* add backend pytest coverage under `backend/tests/` and frontend unit tests
  beside the new React component, following the existing banner/user utility
  test patterns.

The service-tool spam detection section should support:

* listing classifier configurations with enabled state, thresholds, model
  version, timestamps, and last editor if available;
* importing, activating, downloading, and checksum-verifying classifier JSON
  artifacts without storing private artifact content in Git;
* creating or importing training examples;
* labeling an existing scan/review match as `spam` or `ham` so its reviewed
  description becomes a linked training example without requiring a quarantine
  or dismissal action;
* inspecting a repository that a scan missed and, after an operator supplies a
  reason, adding an eligible false negative to the review queue as canonical
  spam training feedback without automatically quarantining it;
* retraining or refreshing the Bayesian model from approved examples;
* configuring scan and ingress thresholds;
* configuring scan filters and dry-run behavior, with repository emptiness always
  enforced for scan matches;
* previewing a saved classifier/policy or unsaved policy draft against the
  read-only replica;
* showing how many repositories would match the current policy draft before it
  is enabled;
* showing classifier and policy snapshots used by historical runs;
* showing the exact reviewed description, canonical training label, and a link
  to the corresponding Quay repository in active and closed review tables;
* recording enough metadata to audit who changed classifier or policy settings
  and when.

The service-tool review queue should support the human remediation loop:

* list active `flagged` and `quarantined` repositories with filters by run,
  namespace, status, and classifier score;
* show model score, explanation details, description excerpt, current status,
  and original description where available;
* require confirmation for quarantine, restore, dismiss, and redaction actions;
* allow an operator to reopen an accidentally dismissed or restored record as
  `flagged`, with a required audit reason and invalidation of the incorrect ham
  feedback created by the terminal action;
* allow an operator to inspect a repository by namespace and name, including its
  current description, score, threshold, and hard-filter results, then add an
  eligible false negative as `flagged` with a required audit reason; this path
  may bypass only the classifier threshold and must still enforce active,
  visibility, empty-repository, and hyperlink requirements;
* use explicit write-capable paths for quarantine, restore, and redaction;
* refresh the affected row after completion;
* avoid bulk redaction in the first implementation unless a separate job or
  queue model is defined.

The initial backend resource set should be explicit enough to map to the
service-tool UI:

| Endpoint | Purpose | DB path |
| --- | --- | --- |
| `GET /spam-detection/classifiers` | List classifier configurations | service-tool state DB |
| `POST /spam-detection/classifiers` | Create or import classifier configuration | service-tool state DB |
| `PUT /spam-detection/classifiers/<uuid>` | Edit classifier settings or enabled state | service-tool state DB |
| `POST /spam-detection/classifiers/<uuid>/train` | Retrain from approved examples | service-tool state DB |
| `POST /spam-detection/classifiers/<uuid>/export-artifact` | Export the trained model with the active ingress policy embedded in a versioned artifact | service-tool state DB |
| `GET /spam-detection/classifiers/<uuid>/artifact` | Download the latest generated artifact as an attachment | service-tool state DB plus artifact storage |
| `GET /spam-detection/policy` | Read active action policy | service-tool state DB |
| `PUT /spam-detection/policy` | Update action policy | service-tool state DB |
| `POST /spam-detection/preview` | Preview a classifier and policy draft | read-only Quay DB replica plus service-tool state DB |
| `POST /spam-detection/runs` | Start a manual scan or enqueue a scheduled scan request | read-only Quay DB replica plus service-tool state DB |
| `GET /spam-detection/runs` | List historical runs | service-tool state DB |
| `GET /spam-detection/runs/<uuid>/matches` | List repository matches for a run | service-tool state DB |
| `GET /spam-detection/review` | List active flagged/quarantined repositories | service-tool state DB |
| `POST /spam-detection/review/manual/inspect` | Inspect a possible false negative without changing review state | read-only Quay DB plus service-tool state DB |
| `POST /spam-detection/review/manual` | Add an eligible false negative as flagged review and canonical spam feedback | read-only Quay DB plus service-tool state DB |
| `POST /spam-detection/review/<uuid>/quarantine` | Apply quarantine to a flagged repository | service-tool state DB plus write-capable Quay DB |
| `POST /spam-detection/review/<uuid>/restore` | Restore a quarantined repository | service-tool state DB plus write-capable Quay DB |
| `POST /spam-detection/review/<uuid>/dismiss` | Dismiss a flagged or quarantined repository | service-tool state DB |
| `POST /spam-detection/review/<uuid>/classify` | Label the reviewed description as spam or ham training feedback | service-tool state DB |
| `POST /spam-detection/review/<uuid>/reopen` | Return an accidentally dismissed or restored record to flagged review | service-tool state DB plus read-only Quay DB validation |
| `POST /spam-detection/review/<uuid>/redact` | Permanently redact approved spam content | service-tool state DB plus write-capable Quay DB |

The existing service-tool backend configures Quay's global Peewee database once
from `DB_URI` during `backend/app.py` startup. The spam detection
implementation must not swap that global connection between write and replica
databases inside individual requests. Preview, scan, and reporting endpoints
should use a read-only replica access path. Quarantine, restore, and redaction
should use an explicit write-capable Quay DB path. The database user behind the
replica URI should be read-only at the database permission level, not just by
application convention.

If the service tool does not already have a durable state database suitable for
this workflow, the implementation must add one or use a dedicated
service-tool-owned schema. That state store is where classifier configuration,
training examples, run history, matches, quarantine records, and action audit
records live. These records should not be stored in Quay application tables.

The service tool should also update its health or startup checks so operators
can tell whether its service-tool state DB, the read-only Quay database
replica, and the write-capable Quay database path for approved actions are all
configured and reachable. Replica lag should be surfaced in operator
documentation because preview and scan results can lag behind primary-database
writes.

### Service-Tool State Records

The service tool should persist run-level metadata for each scan. A companion
match table should store repository-level matches for persisted automated or
manual runs, including dry-run runs.

Scan run records should include:

| Field | Description |
| --- | --- |
| `uuid` | Stable run identifier |
| `source` | `cronjob`, `manual`, or another service-tool runner source |
| `dry_run` | Whether the run was allowed to apply quarantine actions |
| `status` | `running`, `completed`, or `failed` |
| `started_at`, `completed_at` | Run timestamps |
| `classifier_snapshot` | JSON copy of classifier ID, version, thresholds, and model metadata |
| `policy_snapshot` | JSON copy of scan filters, action settings, batch size, and limits |
| `repos_scanned` | Number of repositories evaluated |
| `repos_matched` | Number of repositories whose score met the threshold |
| `repos_flagged` | Number of match records opened in the review queue |
| `repos_quarantined` | Number of repositories quarantined by the run, usually zero for dry-run |
| `error` | Failure details, if any |

Run match records should include:

| Field | Description |
| --- | --- |
| `run_id` | Foreign key to the service-tool run record |
| `repository_id` | Quay repository ID at match time |
| `namespace_name`, `repository_name` | Snapshot of repository identity |
| `description_excerpt` | Bounded description excerpt for review |
| `classifier_score` | Score or probability assigned by the classifier |
| `explanation` | Bounded feature/explanation details for review |
| `is_empty` | Whether the repository was empty during scan |
| `hard_filter_results` | Snapshot of hard identifier checks used for candidate eligibility |
| `quarantine_record_id` | Linked service-tool quarantine row if one was opened |
| `created_at` | Match timestamp |

Quarantine records should also be service-tool owned:

| Field | Description |
| --- | --- |
| `uuid` | Stable external identifier |
| `repository_id` | Quay repository ID at quarantine time |
| `namespace_name`, `repository_name` | Snapshot of repository identity |
| `status` | `flagged`, `quarantined`, `restored`, `dismissed`, or `redacted` |
| `original_description` | Description captured before quarantine |
| `quarantine_description` | Standard quarantine notice written to Quay during quarantine |
| `classifier_score` | Score that caused the repository to be flagged |
| `classifier_snapshot` | Classifier metadata used for the decision |
| `description_fingerprint` | Stable digest of the description evaluated by the classifier |
| `terminal_classifier_snapshot` | Classifier metadata active when the record entered a terminal review status |
| `terminal_description_fingerprint` | Stable digest of the description when the record entered a terminal review status |
| `run_id` | Run that produced the record, if any |
| `actioned_by`, `actioned_at` | Last administrative action metadata |
| `created_at`, `updated_at` | Timestamps |

The quarantine lifecycle is explicit:

* `flagged` -> `quarantined`
* `flagged` -> `dismissed`
* `quarantined` -> `restored`
* `quarantined` -> `dismissed`
* `quarantined` -> `redacted`
* `restored` -> `flagged` with an audit reason
* `dismissed` -> `flagged` with an audit reason

Invalid transitions should raise a service-tool error. Quarantine stores the
original repository description in service-tool state and applies the approved
repository content mutation in Quay. Restore writes the original description
back to Quay from the service-tool quarantine record. Dismissal closes the
review record without modifying repository content beyond any already-applied
quarantine action. Redaction is permanent cleanup and writes directly to the
Quay repository record while preserving service-tool action history.

Review decisions should also be available as classifier feedback for the next
model version. Each review record must have at most one active canonical
training decision. A dismissed or restored repository is evidence that the
current classifier produced a false positive and sets that decision to `ham`
using the reviewed description. A quarantined or redacted repository is
evidence that the classifier found true spam and sets the decision to `spam`
using the original description captured before quarantine or redaction. A
later decision replaces and invalidates the prior active example instead of
leaving contradictory spam and ham examples for the same review record. The
service tool should persist decision history and source metadata that links the
active example and its superseded predecessors to the review record and audit
actions.

When an operator later initiates retraining for that classifier, review-derived
examples should be included in the training corpus by default unless an
operator has explicitly removed or excluded them. Review actions should not
automatically retrain, export, or deploy a new classifier as part of the
action.

Explicit spam/ham labels applied to existing matches should create training
decisions linked to the review record and operator action. Explicit labeling is
only available while a record is `flagged`; after a remediation decision, the
remediation-derived label is authoritative. Labeling supplies classifier
feedback only and must not implicitly quarantine, restore, dismiss, redact, or
otherwise change review status. Reopening a mistaken restore or dismissal must
invalidate its ham decision and return the record to an unlabeled `flagged`
state; the next explicit or remediation decision establishes the new canonical
label.

Terminal review records should suppress repeated review noise for unchanged
repositories. By default, a repository whose latest review record is
`dismissed`, `restored`, or `redacted` should not be opened as a new review
record on a later scan when both the evaluated description fingerprint and the
active classifier artifact version/checksum match the terminal record. The
repository becomes eligible for review again if the repository description
changes, if a new classifier artifact is active, or if service-tool policy
explicitly enables rescanning terminal records. This suppression applies to
review-record creation; scan implementations may still record aggregate skip
counts or diagnostic metadata so operators can understand why a repository was
not reopened.

When an existing repository is moved to `quarantined`, the service tool should
replace the repository description with a standard quarantine notice. The
notice is the repository-owner-facing restoration path and should include:

* that the repository description was removed because automated spam detection
  flagged it for review;
* the support contact or restoration request URL;
* the remediation expected from the owner, such as removing promotional,
  deceptive, or unrelated link content before requesting restore;
* the expected review timeline after a restore request is filed; and
* a stable reference to the namespace/repository and, where needed, an external
  support case or restoration request reference.

The public repository description must not expose the internal service-tool
quarantine record UUID or any other identifier that allows unauthenticated users
to enumerate quarantine activity. Internal quarantine UUIDs should remain in the
service-tool state database, operator UI, and audit history only.

The original description remains in the service-tool quarantine record so a
restore action can write it back after review. The initial implementation
should use one deployment-provided notice string through service-tool policy or
`SPAM_DETECTION_QUARANTINE_DESCRIPTION`; it should not add a separate template
engine or per-namespace message customization.

### Scanner

The scanner should run from `quay-service-tool`, not as an always-on worker in
every Quay pod. It loads the active service-tool classifier and policy, reads
repositories in batches from a read-only Quay database replica, evaluates each
repository, and writes run/match/quarantine-review state only to the
service-tool state database.

Repository reads should use cursor-based pagination over repository IDs rather
than offset pagination:

```
WHERE id > last_seen_id
ORDER BY id
LIMIT batch_size
```

This avoids skipped rows when repository IDs are sparse and avoids
increasingly expensive offsets on large installations. The scanner should
prefetch tag-existence or repository-emptiness inputs for each page to avoid
per-repository database queries. Repositories that are not empty at scan time
must be excluded before writing match history, opening quarantine-review records,
or applying quarantine actions, even when the classifier score exceeds the scan
threshold.

The read path should use a Quay database account or replica that is read-only at
the database permission layer. The service-tool scanner and preview helpers
should also enable a read-only session where the selected database supports it,
so accidental writes through the scan connection fail early.

The scanner supports:

* configurable batch size,
* configurable sleep between batches,
* configurable classifier threshold,
* mandatory repository-emptiness gating for scan matches,
* terminal-review suppression for unchanged repositories that were dismissed,
  restored, or redacted by an operator,
* dry-run mode,
* optional maximum repositories per scan,
* scan IDs for grouping results.

### Preview in quay-service-tool

`quay-service-tool` is the intended UI for spam detection operations. The
preview workflow must be read-only with respect to Quay:

* connect to a read-only Quay database replica;
* load an existing classifier and policy or accept an unsaved policy draft;
* run the same Bayesian classifier used by service-tool scans and Quay ingress;
* return paginated matching repositories with namespace, repository name,
  description excerpt, classifier score, explanation details,
  empty-repository status, hard-filter results, and other configured feature
  inputs;
* show aggregate counts for repositories scanned and matched;
* avoid writes to Quay repository data;
* avoid writes to service-tool run, match, or quarantine history.

The distinction between preview and dry-run is persistence and source. Preview
is an ad hoc, read-only service-tool operation and does not write run or match
rows. Dry-run is a service-tool scan mode that persists run and match history
in service-tool state for later review while avoiding quarantine, redaction,
and Quay content changes.

### Dry Run and Feature Flag Mapping

`FEATURE_SPAM_DETECTION` controls Quay-side ingress evaluation. If the feature
flag is `false`, Quay does not evaluate repository create/update requests for
spam. Service-tool preview can still be available if the service tool is
authorized and configured.

`SPAM_DETECTION_DRY_RUN` controls whether Quay ingress should enforce a
classifier decision when `FEATURE_SPAM_DETECTION` is enabled:

| `FEATURE_SPAM_DETECTION` | `SPAM_DETECTION_DRY_RUN` | Quay ingress behavior |
| --- | --- | --- |
| `false` | any value | No ingress spam evaluation or enforcement |
| `true` | `true` | Evaluate repository create/update requests and allow them to proceed |
| `true` | `false` | Reject repository create/update requests whose changed fields meet the ingress threshold |

Service-tool automated scans should have their own dry-run setting in
service-tool policy. The scan dry-run mapping is:

| Service-tool scan dry-run | Behavior |
| --- | --- |
| `true` | Persist service-tool run and match history only; do not open quarantine records or change Quay repository content |
| `false` | Persist service-tool run and match history and open review-queue records for matches that exceed the scan threshold; human action is still required to quarantine, restore, dismiss, or redact |

### Automated Scheduling

Automated scans should be scheduled as service-tool work, preferably by an
OpenShift `CronJob` that invokes the service-tool scan entry point on the
desired schedule. This avoids running a periodic spam scanner in every Quay pod
and keeps spam detection orchestration with the service tool.

| Option | Pros | Cons |
| --- | --- | --- |
| OpenShift `CronJob` invoking service-tool scan | Exactly one scheduled job per deployment, native retry/history controls, resource requests/limits, no idle process in Quay pods, clear operational ownership | Requires deployment manifests/operator wiring, missed schedules must be considered, job startup imports service-tool configuration each run |
| Always-running service-tool worker | Can reuse service-tool process patterns if they exist | Unsafe if enabled in multiple service-tool pods without a distributed lock, consumes an idle process, harder to reason about schedule ownership in OpenShift |
| Worker with database/Redis lease | Can preserve worker pattern while preventing concurrent scans | Adds locking complexity and failure-mode handling; still runs idle worker processes unless separately constrained |

Running Quay in multiple pods has no automated-scan duplication impact in this
design because Quay does not own the scheduled scanner. Multiple Quay pods do
matter for ingress: all pods must receive the same classifier and policy
configuration, and policy updates must have a defined propagation path. The
implementation should load the same versioned classifier artifact from a
stable path baked into the Quay image and follow the configured fail-closed or
fail-open behavior if the artifact cannot be loaded or verified.

Configuration keys:

| Key | Default | Owner | Description |
| --- | --- | --- | --- |
| `FEATURE_SPAM_DETECTION` | `false` | Quay | Enables repository-description ingress spam evaluation |
| `SPAM_DETECTION_DRY_RUN` | `true` | Quay | Allows ingress evaluation without rejection |
| `SPAM_DETECTION_CLASSIFIER_PATH` | `/conf/spam-detection/classifier.json` | Quay | In-image Bayesian classifier artifact used for ingress evaluation |
| `SPAM_DETECTION_CLASSIFIER_VERSION` | unset | Quay | Expected classifier/policy version for ingress evaluation |
| `SPAM_DETECTION_CLASSIFIER_SHA256` | unset | Quay | Optional SHA-256 checksum for the local classifier artifact |
| `SPAM_DETECTION_FAIL_OPEN` | `true` | Quay | Allows repository updates if the ingress classifier is unavailable |
| `SPAM_DETECTION_DATA_DIR` | deployment-defined persistent path | service tool | Parent directory containing the service-tool state database and managed classifier artifacts; the whole directory must be persisted and backed up |
| `SPAM_DETECTION_READONLY_DB_URI` | unset | service tool | Read-only Quay replica used for preview and scans |
| `SPAM_DETECTION_WRITE_DB_URI` | unset | service tool | Write-capable Quay DB path for approved quarantine, restore, and redaction |
| `SPAM_DETECTION_QUARANTINE_DESCRIPTION` | deployment-provided notice | service tool | Standard quarantine notice written by approved quarantine actions |
| `SPAM_DETECTION_BATCH_SIZE` | `200` | service tool | Repositories scanned per batch |
| `SPAM_DETECTION_SLEEP_BETWEEN_BATCHES` | `0.5` | service tool | Delay between scan batches |
| `SPAM_DETECTION_SCAN_DRY_RUN` | `true` | service tool | Report matches without opening quarantine records or changing Quay content |
| `SPAM_DETECTION_MAX_REPOS` | `0` | service tool | Max repositories per scan, where `0` means unlimited |
| `SPAM_DETECTION_RESCAN_TERMINAL_RECORDS` | `false` | service tool | Reopen dismissed, restored, or redacted repositories only when the description or active classifier artifact changes unless explicitly enabled |
| `SPAM_DETECTION_MIN_SPAM_EXAMPLES` | deployment-defined | service tool | Minimum spam examples required before training/exporting an ingress artifact |
| `SPAM_DETECTION_MIN_HAM_EXAMPLES` | deployment-defined | service tool | Minimum ham examples required before training/exporting an ingress artifact |

### Ingress Blocking

Quay should apply the Bayesian classifier when a user creates or updates a
repository description. Ingress should use the active classifier and ingress
threshold embedded in the service-tool-generated artifact, subject to Quay's
`FEATURE_SPAM_DETECTION` and `SPAM_DETECTION_DRY_RUN` settings. The service-tool
policy is the source of truth for that threshold; Quay only consumes the
versioned local artifact and never calls service-tool on the request path.
Quay must require at least one recognized HTTP or HTTPS hyperlink in the
proposed description before a score can cause an ingress rejection.

The supported production artifact handoff is build-time export into the Quay
image. `quay-service-tool` exports the active classifier/policy artifact as a
versioned JSON file, plus a SHA-256 sidecar, before the Quay image build. The
Quay image build copies that JSON artifact into a stable in-image location,
`/conf/spam-detection/classifier.json`, and copies the checksum sidecar beside
it. Quay loads the artifact from that local path at startup and verifies the
configured `SPAM_DETECTION_CLASSIFIER_VERSION` and optional
`SPAM_DETECTION_CLASSIFIER_SHA256`.

Updating the classifier requires exporting a new JSON artifact, rebuilding the
Quay image with the baked artifact, and rolling all Quay pods to the same image
and artifact version. The initial implementation should not require a runtime
artifact download, shared mutable volume, or service-tool call from Quay pods.

The classifier artifact is a versioned JSON object generated by
`quay-service-tool`. The enhancement does not require a separate formal JSON
Schema document, but Quay and service-tool tests should treat the following
fields as the required artifact contract:

| Field | Type | Description |
| --- | --- | --- |
| `version` | string | Immutable artifact/policy version expected by Quay configuration |
| `training_corpus_version` | string | Version/hash of the training examples used to build the model |
| `spam_prior`, `ham_prior` | number | Class priors used by the Bayesian classifier |
| `token_spam_counts`, `token_ham_counts` | object | Token count maps for spam and ham examples |
| `spam_token_total`, `ham_token_total` | integer | Total token counts for each class |
| `vocabulary_size` | integer | Size of the combined token vocabulary |
| `smoothing` | number | Smoothing factor used for unseen tokens |
| `ingress_threshold` | number | Default score threshold for ingress blocking |
| `ingress_thresholds` | object | Visibility-specific ingress thresholds, such as public/private |
| `feature_config` | object | Supported tokenizer and feature settings, including `classification_window_tokens` and `classification_window_stride` |
| `training_metrics` | object | Corpus counts and validation metrics where available |

Quay should reject or fail-open/fail-closed according to configuration when the
artifact is not a JSON object, misses required fields, has invalid field types,
uses a custom tokenizer pattern that is not allowed on the request path, has the
wrong `version`, or fails the configured SHA-256 check.

Ingress checks should evaluate the proposed repository description and other
request-local fields available on the create/update path. When enforcement is
enabled, rejections should return a clear validation error. Quay should avoid
persisting spam-specific state for ingress decisions in new Quay tables.
Operational visibility for classifier behavior should come from service-tool
preview and scan history, plus standard Quay logs and low-cardinality Prometheus
metrics. Quay must emit an ingress decision counter that distinguishes create
from update and blocked from allowed, dry-run, fail-open, and fail-closed
outcomes without using namespace, repository, description, or other
high-cardinality labels.

### Audit Logging and Notifications

Spam detection audit history should be stored in the service-tool state
database. The service tool should record classifier changes, policy changes,
scan starts and completions, quarantine actions, restore actions, dismissals,
and redactions with operator identity where available.

Quay application audit-log or notification additions are not required for the
initial design because quarantine state is not stored in Quay. If product
requirements later require repository-owner notifications for quarantine or
redaction, that should be designed as a narrow service-tool-to-Quay
integration rather than as a reason to move spam detection state into Quay.

### Database Migration

Quay should not add spam-detection tables for classifier configuration, scan
history, match history, or quarantine state. The expected Quay database impact
is limited to repository content mutations performed by explicit service-tool
actions and any minimal ingress integration that proves necessary during
implementation.

Service-tool state needs its own migration path for classifier configuration,
training examples, runs, matches, quarantine records, and action history. Those
migrations belong to the service-tool implementation because the service tool
owns the spam detection workflow.

Quay does not support production downgrades. This design reduces Quay rollback
risk by avoiding new Quay spam-detection tables. If a release containing the
Quay ingress hook must be rolled back, the rollback plan is to disable
`FEATURE_SPAM_DETECTION` and leave service-tool state intact. Service-tool
rollback should be handled by pausing scheduled scans and retaining the
service-tool state database for forward recovery.

### Indexing and Query Shape

The scanner's main repository scan uses the Quay repository primary key by
querying `WHERE id > last_seen_id ORDER BY id LIMIT batch_size`; the existing
repository primary-key index satisfies this access pattern.

No new Quay indexes should be needed for the initial scanner because scans read
repositories by primary-key pagination from a replica and do not filter by
repository description text inside the Quay database. If the final scan policy
adds database-side filters by namespace, visibility, or repository state, the
implementation should verify existing Quay indexes for those specific filters
before adding any index to large Quay tables.

Service-tool migrations should create indexes for the new service-tool tables,
including:

* classifier configuration by `enabled` and `updated_at`;
* training examples by `label`, `source`, and `created_at`;
* scan runs by `started_at` and by `status, started_at`;
* scan matches by `run_id, classifier_score, id`;
* scan matches by `repository_id, created_at`;
* quarantine records by `status, classifier_score, id`;
* quarantine records by `repository_id, status` to prevent duplicate active
  review records for the same Quay repository;
* action history by `quarantine_record_id, created_at`.

## Risks and Mitigations

* **False positives:** Scan matches require an empty repository, quarantine is
  reversible, dry-run mode is available, and automatic deletion is out of scope.
* **Classifier drift:** The service tool records training examples, classifier
  versions, policy snapshots, and review outcomes so operators can understand
  what changed between runs.
* **Scanner impact on large registries:** Scans run from the service tool
  against a read-only replica, are paginated by primary key, are batched, can be
  rate-limited between batches, and can be capped with
  `SPAM_DETECTION_MAX_REPOS`.
* **Duplicate automated scans:** OpenShift deployments should run scans through
  one CronJob or enforce a deployment-wide lease if using a worker.
* **Preview disrupting registry traffic:** Service-tool preview and scanning
  use a read-only database replica and must not persist preview results.
* **Quay DB write blast radius:** Service-tool writes to the Quay database are
  limited to explicit quarantine, restore, and redaction actions.
* **Inline blocking false positives:** Ingress blocking follows the same
  classifier/policy configuration, feature flag, dry-run setting, and ingress
  threshold as the rest of spam detection.
* **Classifier availability at ingress:** Quay loads a baked-in JSON artifact
  from the image and uses the configured fail-open or fail-closed behavior if
  the artifact is missing, corrupt, or has the wrong version.

## Test Plan

The Quay backend implementation should include pytest coverage for:

* repository-description ingress evaluation when
  `FEATURE_SPAM_DETECTION=false`;
* ingress dry-run behavior when `FEATURE_SPAM_DETECTION=true` and
  `SPAM_DETECTION_DRY_RUN=true`;
* ingress rejection behavior when enforcement is enabled;
* hyperlink gating that allows a high-scoring description without an HTTP or
  HTTPS hyperlink and rejects the same content when a hyperlink is present;
* bounded window scoring that rejects spam even when a long link or boilerplate
  suffix would lower the whole-description score;
* classifier unavailability behavior for the configured fail-open/fail-closed
  mode;
* create and update request behavior for feature-disabled, dry-run, and
  enforced rejection modes;
* ingress metric outcomes for blocked, dry-run, and classifier-unavailable
  requests;
* configuration schema validation for Quay-owned spam detection keys.

The `quay-service-tool` implementation should be tested in its own repository:

* backend pytest coverage for classifier configuration, training examples,
  retraining or model refresh, active-policy threshold embedding, policy
  changes, artifact export/download, and validation;
* backend pytest coverage for preview workflows against the read-only replica;
* backend pytest coverage for scan execution, dry-run persistence, run history,
  match history, and review queue filtering;
* backend pytest coverage proving non-empty repositories are excluded from
  preview results, scan match history, review queue entries, and quarantine
  actions even when the classifier score exceeds the scan threshold;
* backend pytest coverage proving descriptions without hyperlinks are excluded
  from preview and scan matches even when the classifier score exceeds the scan
  threshold;
* backend pytest coverage proving service-tool uses the same bounded window
  scoring as Quay and reports the highest-scoring window in match explanations;
* backend pytest coverage for quarantine, restore, dismiss, reopen, explicit
  spam/ham labeling, canonical feedback replacement, contradictory-label
  prevention, and redaction lifecycle transitions;
* backend pytest coverage for inspecting and manually adding below-threshold
  false negatives, including required reasons, hard-filter enforcement,
  canonical spam feedback, audit history, and subsequent quarantine;
* backend pytest coverage for bounded scans and the explicit `0`/`All`
  unbounded scan mode;
* backend pytest coverage for role gating between preview/reporting and
  write/remediation actions;
* backend pytest coverage for read-only replica path selection for
  preview/scanning/reporting and write-capable path selection for quarantine,
  restore, and redaction;
* backend pytest coverage proving scan/preview read-only database connections
  reject accidental writes where the database supports session-level read-only
  mode;
* backend pytest coverage for health/startup behavior when the service-tool
  state DB, read-only Quay replica, or write-capable Quay DB path is
  unavailable;
* service-tool migration tests for classifier, training, run, match,
  quarantine, and action-history tables and indexes;
* frontend unit coverage for classifier configuration, policy editing,
  preview, run-history views, reviewed descriptions, canonical labels,
  repository links, false-negative intake, review queue actions, and API error
  states using the existing `HttpService` mocking pattern;
* a seeded local manual-exploration workflow that starts Quay and service-tool,
  imports the configured classifier, creates review data, opens both UIs, signs
  in, and then performs no automated review actions.

No Quay UI tests are required for this enhancement because the Quay application
does not add an in-tree UI surface.

## Graduation Criteria

### Dev Preview

* Quay ingress evaluation is disabled by default.
* Dry-run mode is available for Quay ingress and service-tool scans, and both
  default to non-enforcing behavior.
* `quay-service-tool` can preview classifier matches against a read-only Quay
  database replica.
* Historical service-tool scan records show scanned, matched, flagged,
  quarantined, and redacted counts.
* Operators can configure a classifier and policy and run scans in
  non-production or limited production environments.
* Unit and migration tests cover the Quay ingress hook and the service-tool
  classifier, scan, review, and lifecycle actions.

### Tech Preview

* Operational documentation describes configuration, rollout, dry-run review,
  classifier training, redaction, and recovery.
* OpenShift deployments run automated scans as a service-tool CronJob or
  otherwise enforce single-runner execution.
* Scale testing confirms scanner behavior on large repository counts against a
  read-only replica.
* Service-tool audit history is validated for classifier changes, policy
  changes, quarantine, restore, dismiss, and redaction.
* A supported `quay-service-tool` workflow exists for classifier management,
  policy management, preview, historical run review, quarantine, restore,
  dismiss, and redaction.

### GA

* The feature has sustained production usage with acceptable false-positive and
  scan-performance behavior.
* Quay ingress feature-disable rollback behavior is validated.
* Service-tool state migrations and forward recovery behavior are validated.
* Backend and frontend unit coverage exists for the supported
  `quay-service-tool` workflow.
* Product documentation covers service-tool classifier configuration, policy
  configuration, preview, historical review, quarantine, restore, redaction,
  quarantine notice text, baked artifact updates, scheduling, and
  troubleshooting.

## Upgrade / Downgrade Strategy

Quay upgrades add the repository-description ingress hook and configuration
schema entries only. Existing deployments see no behavior change because
`FEATURE_SPAM_DETECTION` defaults to `false`.

Service-tool upgrades add the spam detection state schema, classifier and policy
management, scan runner, review workflows, and redaction workflows. Existing
service-tool deployments see no behavior change until spam detection is
configured and a scan is scheduled.

Quay does not support production downgrades. The rollback mechanism for Quay is
to disable `FEATURE_SPAM_DETECTION` and leave service-tool state in place.
Service-tool rollback should pause scheduled scans and preserve the
service-tool state database so a later forward upgrade can resume from the last
known classifier, policy, run, and quarantine records.

## Version Skew Strategy

Quay pods that do not enable `FEATURE_SPAM_DETECTION` ignore spam detection
ingress evaluation. During rolling upgrades, the feature should remain disabled
until all Quay application instances have the ingress hook and configuration
schema that understand the spam detection settings.

If ingress evaluation is enabled during a later rolling update, all Quay pods
must use a compatible classifier/policy version. Because the classifier JSON is
baked into the Quay image, operators should roll pods by image version and keep
`SPAM_DETECTION_CLASSIFIER_VERSION` aligned with that image. The implementation
should make classifier version mismatches visible and should follow the
configured fail-open or fail-closed behavior when a pod cannot load the
expected classifier configuration.

Service-tool version skew is handled separately. Scheduled scans should run
from one service-tool version at a time, preferably through an OpenShift
CronJob image pinned to the intended service-tool release. Operators should
pause scheduled scans before rolling back or replacing the service-tool spam
detection implementation.

## Drawbacks

This adds a new service-tool state store or schema, service-tool migrations,
classifier training workflows, scheduled scan operations, and Quay ingress
integration. Bayesian filtering reduces manual keyword maintenance, but it
still requires operators to curate training examples and monitor
false-positive behavior. Scanning very large registries consumes read capacity
on the configured replica even when paginated and rate-limited.

Ingress blocking also adds operational risk because repository create/update
requests can depend on classifier availability and policy propagation. The
feature flag, dry-run mode, baked JSON artifact, classifier versioning, and
explicit fail-open or fail-closed configuration are required to make that risk
manageable.

## Alternatives

* **Manual moderation only:** Avoids new code but does not scale to public
  registry spam volume.
* **Keyword-based rules in Quay:** Simpler to implement initially, but creates
  a long-term keyword maintenance problem and stores spam-specific policy in
  Quay rather than the service tool.
* **Quay-owned scan and quarantine tables:** Keeps all state close to the
  repository data, but expands the Quay schema and conflicts with the desired
  service-tool ownership boundary.
* **Immediate deletion:** Reduces visible spam quickly but is unsafe for false
  positives and does not satisfy the reversibility requirement.
* **External-only scanner without service-tool state:** Keeps Quay smaller but
  lacks a supported operator interface, durable review state, and historical
  run reporting.
* **Always-running Quay worker:** Reuses existing Quay worker conventions but
  creates duplicate-scan risk in multi-pod deployments and puts orchestration
  in Quay instead of the service tool.

## Implementation History

* 2026-06-12 Initial proposal for QUAYIO-1637.
* 2026-06-20 Updated proposal to use Bayesian filtering, service-tool-owned
  spam detection state, service-tool scan orchestration, and Quay ingress as
  the narrow Quay-side integration point.
* 2026-07-03 Clarified that quarantined repository descriptions are replaced
  with a restore-contact notice and that Quay consumes a JSON classifier
  artifact baked into the image.
* 2026-07-07 Required empty repositories for scan matches and quarantine-review
  eligibility and documented additional hard identifiers for reducing false
  positives.
* 2026-07-10 Added terminal-review rescan suppression and review-action
  feedback for future classifier training.
* 2026-07-15 Required hyperlink eligibility and Quay ingress metrics, added
  artifact downloads and explicit match labels, generalized terminal-action
  recovery, and documented unbounded scans and update-path coverage.
* 2026-07-15 Added persistent service-tool classifier storage, validated
  artifact import and base-model retraining, immediate use of trained versions
  for manual scans, canonical review feedback, visible review descriptions and
  repository links, and a seeded manual-exploration workflow.
* 2026-07-15 Added audited false-negative intake so eligible repositories missed
  by the active threshold can enter review and future training without being
  quarantined automatically.
* 2026-07-15 Added bounded overlapping classifier windows to prevent long link
  or boilerplate suffixes from diluting an otherwise high-confidence spam score.
