# parmana-github-test

Throwaway repository for testing the Parmana GitHub connector (`github:pr-fetch` and `github:pr-merge`) before it is used on real repositories.

**Do not use this repository for real work.** Its pull requests are merged during tests, and a merge cannot be undone by the connector.

## Current state

- `main` has a README and a workflow named `CI` with one job, `test`. The job is a placeholder check that prints a message.
- PR #1 (`test/pr-fetch-merge`) has been merged into `main`. A new throwaway PR is needed for every merge test.
- Branch protection on `main`: a required status check named `test`, with required approvals set to 0 for an owner-run test.
- Open item: PR #1 showed "Cannot update this protected ref" before it was merged. The ruleset rule that caused it is not yet recorded here. Record it before the next merge test.

## What this repository is for

It is the staging target for the connector's go-live test plan. The plan's phases are:

0. **Prerequisites:** a GitHub App installed on this repository only, the App credential, the Postgres nonce store, the Parmana public key, a named approver, and the two policies in effect.
1. **Automated tests:** `npm run typecheck`, `npm run lint` and `npm test` in `parmana-connectors`, all passing on a clean checkout.
2. **Staging tests (this repository):** the credential, read, merge, failure and evidence tests below.
3. **Security review:** resolve the branch-protection blocker, check the caller-declared signals, scan for secrets, and confirm host isolation and routing.
4. **Operational drills:** App key rotation, App revocation, Parmana key rotation, and the runbook steps.
5. **Limited live pilot:** one low-stakes repository with branch protection, not this one.

Do not move to phase 5 until every phase has an exit gate recorded with evidence.

## How the go-live will be done

This is the order of work. Each step ends with an exit gate. Record every result in `docs/TEST-CASES.md`, in the Result and Evidence columns of the test's row.

### Step 1: prepare (before any test)

1. Create the GitHub App and install it on this repository only (`CFG-09`, `CRD-01`).
2. Store the App key outside any repository. Set `GITHUB_APP_CREDENTIAL`, the Parmana public key, `PARMANA_RELEASE_AUDIENCE` (https) and the Postgres nonce store (`CFG-01` to `CFG-08`).
3. Name the two approvers. Each keeps a signing key on their own machine. One of them must not be the PR author.
4. Confirm that `github-pr-read` 1.1.0 and `github-pr-approval` 1.1.0 are in effect on the test instance.

**Gate:** every `CFG` case passes, and the App, approvers and policies are recorded in the Contacts section below.

### Step 2: automated tests

1. In `parmana-connectors`, run typecheck, lint and `npm test` on a clean checkout (`AUT-01`).
2. Run the input-boundary cases (`AUT-03` to `AUT-14`).

**Gate:** all pass, with the test count recorded.

### Step 3: staging runs against this repository

Run the phases in this order. Each order matters: a later test assumes the earlier one has passed.

1. Credentials (`CRD-01` to `CRD-07`).
2. Release envelope and nonces (`ENV-01` to `ENV-15`).
3. Token lifecycle (`TOK-01` to `TOK-07`).
4. Read path (`RD-01` to `RD-12`), using open PRs only.
5. Merge path (`MRG-01` to `MRG-23`). Open a new throwaway PR for each merge test, and never reuse a merged one. Each PR is named after its test ID, for example `MRG-04 throwaway`.
6. Failure, concurrency and evidence (`FAL`, `CON`, `EVD`).

**Gate:** no test has merged a PR it should have refused. Every `Verify first` case is confirmed, and its result recorded.

### Step 4: decisions

1. Decide on `MRG-14`: whether `expectedHeadSha` is required on every merge. Record the decision in the README and in the policy, not only in the test file.
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
4. Week 2: `github:pr-merge` for documentation-only PRs, with a named approver for every merge.
5. Watch the first five merges personally. Check each trust record with `parmana-verifier`.

**Stop and disable `github:pr-merge`** if a merge is approved that policy should have refused, a trust record fails verification, a token appears in a log, or an `upstream_error` cannot be explained.

**Rollback:** the connector cannot undo a merge. Revert the commit with `git revert` on the target branch, and keep the original trust record.

### Step 8: after the pilot

Update this README and `docs/TEST-CASES.md` with the pilot results, the decisions taken and any new cases. Open a PR for that update.

---

## Staging tests

Record the result and the evidence (run ID, error code, PR state) for each test.

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

### Failure and ambiguity

| Test | Expected |
|---|---|
| `PARMANA_CONNECTOR_TIMEOUT_MS=1` | Timeout error. Check the PR state afterwards. |
| Network interrupted during a merge (or the fake) | The error says the merge may have reached GitHub, and to check the PR |
| GitHub returns HTTP 500 for a merge | `upstream_error`, with the check-PR instruction |

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
