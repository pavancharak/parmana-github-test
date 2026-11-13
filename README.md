# parmana-github-test

A throwaway repository for testing the Parmana GitHub connector before it is used on any real repository.

**Do not use this repository for real work.** Its pull requests are merged during tests, and the connector cannot undo a merge.

## What we are doing and why

Parmana connectors carry out an action only after a trusted person has approved that exact action. The GitHub connector does three things, and each one changes or reads real GitHub state:

- it reads pull requests,
- it merges them,
- it posts comments on them.

Each of these has to be tested against the real GitHub API before the connector is used on a real repository. This repository is the place to do that. It is disposable, so a wrong merge or a stray comment costs nothing here. The tests are written down in this README and in `docs/TEST-CASES.md`, and each result is recorded with its evidence.

The overall go-live is a sequence of phases and gates. The steps are below. No step is skipped, and nothing moves to a real repository until the owner signs off the go / no-go decision.

## What the connector does

| Capability | Policy | What it does |
|---|---|---|
| `github:pr-fetch` | `github-pr-read` 1.1.0 | Reads mergeable state, head SHA and base branch. Read-only. |
| `github:pr-merge` | `github-pr-approval` 1.1.0 | Merges with `merge`, `squash` or `rebase`. Can refuse if the head SHA has moved (`expectedHeadSha`). |
| `github:comment` | `github-pr-comment` (not yet in effect) | Posts one comment, with a body of up to 4,000 characters and a hidden marker that carries the transaction ID. A comment is never sent again if the answer is lost. |

Every action needs a signed approval from Parmana. The connector refuses anything without one, and refuses any parameter it does not list. Targets have the form `owner/repo#number`.

The connector's own documentation is in `parmana-connectors`, under `src/connectors/github/`: `README.md`, `RUNBOOK.md`, `COMMENTS.md` and `ROLLOUT.md`.

## Current state

- `main` has this README and a workflow named `CI` with one job, `test`. The job is a placeholder check that prints a message.
- PR #1 (`test/pr-fetch-merge`) was merged into `main`. Each merge test needs a new throwaway PR.
- Branch protection on `main` requires the status check `test`. Required approvals are 0, for an owner-run test.
- **Open:** PR #1 once showed "Cannot update this protected ref" before it was merged. The ruleset rule that caused it is not yet recorded here. Record it before the next merge test (`SEC-08`).
- **Not yet verified:** which GitHub App permission lets the App post comments on a pull request. `CMT-00` settles this.
- The `github:comment` cases are written here but have not been run. They are on branch `docs/github-comment-cases`, which is not merged.
- The approver and the App owner are not named yet (see Contacts).

## How the go-live will be done

Each step ends with an exit gate. Record every result in `docs/TEST-CASES.md`, in the Result and Evidence columns of the test's row.

### Step 1: prepare (before any test)

1. Create the GitHub App and install it on this repository only (`CFG-09`, `CRD-01`). It needs write access to pull request conversation comments for `github:comment`. Which permission grants that is verified by `CMT-00`.
2. Store the App key outside any repository. Set `GITHUB_APP_CREDENTIAL`, the Parmana public key, `PARMANA_RELEASE_AUDIENCE` (https) and the Postgres nonce store (`CFG-01` to `CFG-08`).
3. Name the two approvers. Each keeps a signing key on their own machine. One of them must not be the PR author.
4. Confirm that `github-pr-read` 1.1.0 and `github-pr-approval` 1.1.0 are in effect on the test instance. `github-pr-comment` must be in effect before the comment tests run.

**Gate:** every `CFG` case passes, and the App, approvers and policies are recorded in Contacts.

### Step 2: automated tests

1. In `parmana-connectors`, run typecheck, lint and `npm test` on a clean checkout (`AUT-01`).
2. Run the input-boundary cases (`AUT-03` to `AUT-14`).

**Gate:** all pass, with the test count recorded.

### Step 3: staging runs against this repository

Run these in order. Each depends on the one before it.

1. Credentials (`CRD-01` to `CRD-07`).
2. Release envelope and nonces (`ENV-01` to `ENV-15`).
3. Token lifecycle (`TOK-01` to `TOK-07`).
4. Read path (`RD-01` to `RD-12`), using open PRs only.
5. Merge path (`MRG-01` to `MRG-23`). Open a new throwaway PR for each merge test, and never reuse a merged one. Name each PR after its test ID, for example `MRG-04 throwaway`.
6. Failure, concurrency and evidence (`FAL`, `CON`, `EVD`).
7. Comment path (`CMT-00` to `CMT-12`), on an open throwaway PR. Comments do not change a PR's state, so one open PR can hold all of them. Record each comment's ID and marker.

**Gate:** no test has merged a PR it should have refused. Every "verify first" case is confirmed, and its result recorded. Every `CMT` case has a recorded result, and no test left more than one comment for the same transaction ID.

### Step 4: decisions

1. Decide `MRG-14`: whether `expectedHeadSha` is required on every merge. Record the decision here and in the policy, not only in the test file.
2. Resolve the branch-protection rule that blocked PR #1, and record which rule it was (`SEC-08`).

**Gate:** both decisions are written down, and the named owner signs them off.

### Step 5: security checks and drills

1. Run the security checks (`SEC-01` to `SEC-11`).
2. Run the operational drills (`OPS-01` to `OPS-10`), timing each one.

**Gate:** every security check is recorded, and each drill has been run once, with its time.

### Step 6: go / no-go

The named owner reviews the recorded results against the gates above.

- **Go** only if every gate is met, with evidence linked from the test file.
- **No-go** if any merge was approved that should have been refused, any trust record fails verification, or any token appears in a log. Fix the cause, then repeat Step 3 from the start.

### Step 7: limited live pilot

1. Choose one low-stakes repository with branch protection. Not this repository.
2. Set `PARMANA_ALLOW_LIVE_KEYS=true` only after the go decision is recorded.
3. Week 1: `github:pr-fetch` only, on that repository.
4. Week 2: `github:comment` only, on open PRs in that repository. Each comment is approved by a named approver, and the approved text is the exact text posted. Check each comment's marker with the search in `COMMENTS.md`.
5. Week 3: `github:pr-merge` for documentation-only PRs, with a named approver for every merge.
6. Watch the first five merges and the first five comments personally. Check each trust record with `parmana-verifier`.

**Stop and disable `github:comment`** if a comment is posted that its approval did not cover, a comment is posted twice for one transaction ID, a token appears in a comment, or an `upstream_error` about a comment cannot be explained.

**Stop and disable `github:pr-merge`** if a merge is approved that policy should have refused, a trust record fails verification, a token appears in a log, or an `upstream_error` cannot be explained.

**Rollback:** the connector cannot undo a merge. Revert the commit with `git revert` on the target branch, and keep the original trust record.

### Step 8: after the pilot

Update this README and `docs/TEST-CASES.md` with the pilot results, the decisions taken and any new cases. Open a PR for that update.

---

## How to run the tests

- **Record every result** with its run ID, error code and PR state. A result without evidence is not a pass.
- **Merge tests** each use a new throwaway PR, named after the test ID. A merge is final, so never reuse a merged PR.
- **Comment tests** use one open PR named `CMT throwaway`. Use a separate transaction ID for each case, and write it down.
- **Timeouts** are set with `PARMANA_CONNECTOR_TIMEOUT_MS`. Keep the default except in the timeout cases.
- **Never commit** a PEM, an App credential, a `.env` file or a database URL, to this repository or any other.

## Staging tests

### Credential

| Test | Expected |
|---|---|
| Valid credential, `pr-fetch` on an open PR | Success, with the head SHA and a token fingerprint |
| Wrong installation id | Refused with `upstream_error` or `credential_invalid`. The message does not show the credential. |
| Malformed credential (two parts, or not a PEM) | Refused as `credential_invalid`, with no call to GitHub |
| Search logs for the installation token | Not found. Only fingerprints appear. |

### Read (`github:pr-fetch`)

| Test | Expected |
|---|---|
| With a valid `github-pr-read` approval for the PR | Success, returning `mergeable`, `headSha` and `baseRef` |
| Without an approval | Refused by policy, with no call to GitHub |
| Approval for a different PR number | Refused by policy |
| Target `owner/repo#abc` | Refused as `invalid_input` |
| An extra parameter | Refused before any call |
| `RD-13` **[High]** Read while GitHub reports `mergeable` as null (still computing) | The read does not report the PR as mergeable. Verify first: record the value, and whether a merge is refused on it. |

### Merge (`github:pr-merge`)

Use a new throwaway PR for each test. A merge is final.

| Test | Expected |
|---|---|
| No `github-pr-approval` approval | Refused by policy. The PR stays open. |
| Approval present, a required check failing | GitHub refuses. The error says to check the PR. |
| Approval present, `expectedHeadSha` matches | Merged. The merged head SHA matches the approval. |
| `expectedHeadSha` stale (push a commit to the PR after the decision) | GitHub refuses the merge. The PR stays open with the new commit. |
| Same release sent again after a merge | Refused by the nonce store. A repeat at GitHub also fails. |
| `merge`, `squash` and `rebase`, one PR each | Each merges with the chosen method. |
| Unknown `mergeMethod` | Refused before any call |

### Comment (`github:comment`)

Each request posts one comment: the body, a blank line, then the hidden marker `<!-- parmana:btx=<businessTransactionId> -->`. Find a comment by searching for its marker, as `COMMENTS.md` describes.

| Test | Expected |
|---|---|
| `CMT-00` App with only "Pull requests: write" posts a comment | Verify first. If it returns HTTP 403, the App needs another permission (for example "Issues: write"). Record which one was needed, and update the App setup in this README. |
| `CMT-01` Valid body, valid transaction ID, open PR | Posted once. The response has `commentId` and `url`. The comment text is the body, a blank line, then the marker. |
| `CMT-02` Marker search for that transaction ID | Exactly one comment matches |
| `CMT-03` Body of exactly 4,000 characters | Posted once |
| `CMT-04` Body of 4,001 characters | Refused as `invalid_input` before any call. No comment appears. |
| `CMT-05` Empty or whitespace-only body | Refused as `invalid_input` before any call |
| `CMT-06` Extra parameter, for example `mergeMethod` | Refused before any call |
| `CMT-07` Missing transaction ID, or one containing `-->` or a space | Refused as `invalid_input` before any call |
| `CMT-08` Same transaction ID sent again after `CMT-01` | Refused by the release ledger. The comment count on the PR stays the same. |
| `CMT-09` Malformed target, for example `owner/repo` with no number | Refused before any call |
| `CMT-10` `PARMANA_CONNECTOR_TIMEOUT_MS=1` | The error says the comment may have reached GitHub, and to check the PR. Search for the marker. Record zero or one match, never two. Do not post again. |
| `CMT-11` Lost answer after the post (fake, or a dropped connection) | As `CMT-10`. A marker search finds the comment, which is recorded without a second post. |
| `CMT-12` Token and credential checks | No installation token or credential appears in the response, the error or the comment text. Only a fingerprint appears. |
| `CMT-13` **[High]** Body containing a fake marker `<!-- parmana:btx=<another id> -->` | The marker search for the real transaction ID finds only the real comment. Verify first: record the search result. |
| `CMT-14` **[High]** Body of 4,000 emoji (16,000 bytes) | Verify first: confirm whether the limit counts characters or UTF-16 units. GitHub accepts it, under its 65,536-byte limit. |
| `CMT-15` **[High]** Target is an issue number, not a pull request | Refused, or reported as an error. No false success. Verify first: record the response. |
| `CMT-16` **[High]** Comment on a closed PR, then on a merged PR | Verify first: record whether GitHub accepts each comment. The connector reports GitHub's answer. |

Mentions are not tested here. A body with `@user` notifies that user, and nothing checks for it. If a mention is ever tested, use a test account only.

### Failure and ambiguity

| Test | Expected |
|---|---|
| `PARMANA_CONNECTOR_TIMEOUT_MS=1` | Timeout error. Check the PR state afterwards. |
| Network interrupted during a merge (or the fake) | The error says the merge may have reached GitHub, and to check the PR |
| GitHub returns HTTP 500 for a merge | `upstream_error`, with the check-PR instruction |
| `FAL-15` **[Lower]** Host clock ten minutes off, so the App JWT is rejected | `credential_invalid` or `upstream_error`, with a clear message. No merge or comment is attempted. |
| `FAL-16` **[Lower]** App uninstalled or suspended during a run | Later calls stop with an error. No success is reported. |
| `FAL-17` **[Lower]** GitHub returns HTTP 503 | `upstream_error`, with the check-PR instruction. No retry. |

### Evidence

| Test | Expected |
|---|---|
| Verify each trust record with `parmana-verifier` | Passes |
| Change one byte of a record, then verify | Fails |
| Check that no record contains the installation token | Confirmed |

## Security checks before phase 5

- **Caller-declared signals.** The merge policy trusts `requiredReviewsCompleted`, `statusChecksPassed` and `branchProtected`, which the caller declares. Test that a caller declaring `statusChecksPassed: true` on a failing PR still cannot merge, because GitHub refuses it.
- **Secrets.** No PEM, App credential or Postgres URL is committed to any repository. Run secret scanning on this repository and on `parmana-connectors`.
- **Host isolation (G-128).** The connector credential can be read by code on the same host. Confirm the host is controlled and its environment restricted.
- **Routing (G-129).** Nothing calls GitHub directly for these actions. Only the gateway holds the App credential.

## Rules for this repository

- Never commit a PEM, an App credential, a `.env` file or a database URL.
- Do not reuse a merged PR for a new merge test. Open a new PR.
- Keep the required status check on `main`. Record any ruleset change here.

## Contacts

- Approver for these tests: **TODO**
- Owner of the App and its key: **TODO**
