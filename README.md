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
