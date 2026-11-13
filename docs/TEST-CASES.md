# GitHub connector test cases

Test cases for the Parmana GitHub connector (`github:pr-fetch`, `github:pr-merge` and `github:comment`), run against this repository before the connector goes live. Cases tagged **[High]** or **[Lower]** are the priority for the run; untagged cases are run as well. The phases, exit gates and rules are in the [README](../README.md).

## How to record a result

For each test, record:

- **Result:** Pass, Fail, or Blocked (with the reason). Enter it in the `Result` column of the test's row.
- **Evidence:** run ID, error code, PR number, PR state before and after, trust record ID. Enter a short reference in the `Evidence` column, and the full record elsewhere.
- **Tester and date:** add them in the `Evidence` column, for example `Pass, run 4169… (tester, 2026-11-04)`.

A test that merges a PR must use a new throwaway PR. Never reuse a merged PR.

Mark a case **Verify first** when its expected result depends on behaviour that has not yet been confirmed against the code or a live GitHub App. Confirm the behaviour, then record the expected result.

## Preconditions

- GitHub App installed on this repository only, with the permissions in the connector README
- `GITHUB_APP_CREDENTIAL`, `PARMANA_PUBLIC_KEY`, `PARMANA_RELEASE_AUDIENCE` and `PARMANA_NONCE_STORE=postgres` set for the test service
- `PARMANA_ALLOW_LIVE_KEYS=false`
- Policies `github-pr-read` 1.1.0 and `github-pr-approval` 1.1.0 in effect
- Two named approvers, each with a signing key on their own machine. One of them must not be the PR author.

---

## Phase 0: configuration (fail closed)

| ID | Test | Expected | Result | Evidence |
|---|---|---|--------|----------|
| CFG-01 | Start with `PARMANA_CONNECTOR` unset or set to another connector | Refuses to start |  |  |
| CFG-02 | Start with `GITHUB_APP_CREDENTIAL` unset | Refuses to start |  |  |
| CFG-03 | Start with both `PARMANA_PUBLIC_KEY` and `PARMANA_PUBLIC_KEY_FILE` set | Refuses to start |  |  |
| CFG-04 | Start with neither public key variable set | Refuses to start |  |  |
| CFG-05 | `PARMANA_RELEASE_AUDIENCE` with `http://`, or with no scheme | Refuses to start |  |  |
| CFG-06 | `PARMANA_NONCE_STORE=memory` with two instances running | Verify first: the design says Postgres is required for more than one instance. Expected: the second instance refuses to start, or the test fails and the gap is recorded. |  |  |
| CFG-07 | `PARMANA_DATABASE_URL` unset with `PARMANA_NONCE_STORE=postgres` | Refuses to start |  |  |
| CFG-08 | Any `rzp_live_`-style or live setting while `PARMANA_ALLOW_LIVE_KEYS=false` | Refused at startup and on each call |  |  |
| CFG-09 | Start with a valid configuration | Starts. Record the configuration with every secret value masked. |  |  |

---

## Phase 1: automated tests (in `parmana-connectors`)

| ID | Test | Expected | Result | Evidence |
|---|---|---|--------|----------|
| AUT-01 | `npm ci`, `npm run typecheck`, `npm run lint`, `npm test` on a clean checkout | All pass. Record the test count. |  |  |
| AUT-02 | GitHub connector tests (`tests/connectors/github/`) | Pass, with no skipped GitHub tests |  |  |
| AUT-03 | `mergeMethod` other than `merge`, `squash` or `rebase`, including `MERGE`, `Squash` and an empty string | Refused as `invalid_input` before a call |  |  |
| AUT-04 | `expectedHeadSha` that is not 40 lowercase hex characters: uppercase, 39 characters, 41 characters, a non-string | Refused as `invalid_input` before a call |  |  |
| AUT-05 | A parameter not allowed for the capability, for example `foo` on `pr-fetch` | Refused before a call |  |  |
| AUT-06 | A target that is not `owner/repo#number` | Refused as `invalid_input` before a call |  |  |
| AUT-07 | A capability other than `github:pr-fetch` or `github:pr-merge` | Refused as `unknown_capability` |  |  |
| AUT-08 | Target with leading or trailing spaces, for example ` owner/repo#1 ` | Refused, not trimmed. Verify first. |  |  |
| AUT-09 | Owner of 40 characters, or a repo name of `.` or `..` | Refused as `invalid_input` |  |  |
| AUT-10 | Pull number of 0, a negative number, `1.5`, or 11 digits | Refused as `invalid_input` |  |  |
| AUT-11 | Target with a path separator or query, for example `owner/repo/../x#1` or `owner/repo#1?x=y` | Refused as `invalid_input` |  |  |
| AUT-12 | Parameter named `__proto__`, `constructor` or `toString` | Refused. The request does not change the object. |  |  |
| AUT-13 | `mergeMethod` with a value of the wrong type, such as a number or an array | Refused as `invalid_input` |  |  |
| AUT-14 | Request body larger than the service limit | Refused before parsing. Verify first: record the limit. |  |  |

---

## Phase 2a: credential and authentication

| ID | Test | Expected | Result | Evidence |
|---|---|---|--------|----------|
| CRD-01 | Valid credential, `pr-fetch` on an open PR | Success, with `headSha` and a `tokenFingerprint` |  |  |
| CRD-02 | Wrong installation id | `upstream_error` or `credential_invalid`. The message does not show the credential. |  |  |
| CRD-03 | Wrong App id | As CRD-02 |  |  |
| CRD-04 | Credential with two parts, or a value that is not a PEM after base64 decoding | `credential_invalid`, with no call to GitHub |  |  |
| CRD-05 | Search the service logs and the trust records for the installation token | Not found. Only fingerprints appear. |  |  |
| CRD-06 | Revoke the App installation, then call `pr-fetch` | Fails. Record the error. |  |  |
| CRD-07 | Reinstall the App, then call `pr-fetch` | Succeeds again |  |  |

---

---

## Phase 2a (part 2): release envelope and nonces

| ID | Test | Expected | Result | Evidence |
|---|---|---|--------|----------|
| ENV-01 | Release with `PARMANA_RELEASE_AUDIENCE` set to a different URL | Refused. No call to GitHub. |  |  |
| ENV-02 | Release past its expiry time | Refused. No call to GitHub. |  |  |
| ENV-03 | Release with `exp` at the exact expiry instant | Refused |  |  |
| ENV-04 | Release issued in the future, beyond the allowed clock skew | Refused. Record the skew limit used. |  |  |
| ENV-05 | Release signed with `alg: none` | Refused |  |  |
| ENV-06 | Release signed with HS256 using the public key as the secret | Refused |  |  |
| ENV-07 | Release signed by a key that is not the configured Parmana key | Refused |  |  |
| ENV-08 | Release with one parameter changed after signing | Refused. The signature check fails. |  |  |
| ENV-09 | Release with no nonce | Refused |  |  |
| ENV-10 | The same release sent twice, one after the other | The second is refused by the nonce store. GitHub is called once. |  |  |
| ENV-11 | The same release sent twice at the same moment (two parallel requests) | Exactly one succeeds. The other is refused. |  |  |
| ENV-12 | Two instances receive the same release at the same moment (requires two instances) | Exactly one succeeds. Verify first with the Postgres nonce store. |  |  |
| ENV-13 | Nonce store (Postgres) unreachable during a release | Refused. Nothing is sent to GitHub. |  |  |
| ENV-14 | Release for `pr-merge` used as a `pr-fetch` request (capability swap) | Refused |  |  |
| ENV-15 | Release that names a target different from the signed target | Refused |  |  |

---

## Phase 2b: token lifecycle

| ID | Test | Expected | Result | Evidence |
|---|---|---|--------|----------|
| TOK-01 | Each call mints a new installation token | Count the token requests in the fake or the logs. One per call. |  |  |
| TOK-02 | The installation token is not reused by a second call | Confirmed by the request trace. Verify first. |  |  |
| TOK-03 | App JWT is valid for 10 minutes and back-dated 60 seconds | Confirmed by decoding a JWT in a test. Expected: `iat` is 60 s before now, `exp` is 600 s after now. |  |  |
| TOK-04 | GitHub returns 401 on the installation token request | `credential_invalid`. No merge attempted. |  |  |
| TOK-05 | GitHub returns 401 on the merge request | `upstream_error`. The message says to check the PR. |  |  |
| TOK-06 | Installation token expires during a slow merge | Verify first. Expected: `upstream_error`, with the check-PR instruction. No retry. |  |  |
| TOK-07 | Private key in `GITHUB_APP_CREDENTIAL` is not a valid RSA key | `credential_invalid`. The key is not shown. |  |  |

---

## Phase 2c: read path (`github:pr-fetch`)

| ID | Test | Expected | Result | Evidence |
|---|---|---|--------|----------|
| RD-01 | Valid `github-pr-read` approval for the open PR | Success, returning `mergeable`, `headSha`, `baseRef` and `mergedAt` |  |  |
| RD-02 | No approval | Refused by policy. No call to GitHub. |  |  |
| RD-03 | Approval for a different PR number | Refused by policy |  |  |
| RD-04 | Approval with `readApproved: false` | Refused by policy |  |  |
| RD-05 | Approval signed by a key that is not trusted | Refused. Nothing is read. |  |  |
| RD-06 | Approval altered after signing (for example, the PR number changed) | Refused. The signature check fails. |  |  |
| RD-07 | Read a PR that is already merged | Success, with `mergedAt` set |  |  |
| RD-08 | Read a PR number that does not exist | `upstream_error`. No token or credential is shown. |  |  |
| RD-09 | Read a PR in a repository the App is not installed on | Refused. No credential is shown. Record the error. |  |  |
| RD-10 | Read a closed, unmerged PR | Success, with `state` closed and `mergedAt` null |  |  |
| RD-11 | Read a draft PR | Success, with the draft state visible. Verify first: confirm the connector returns this field. |  |  |
| RD-12 | Read a PR from a fork | Verify first: record the result. The App must have access to the fork. |  |  |
| RD-13 | **[High]** Read a PR while GitHub reports `mergeable` as null (still computing) | The read does not report the PR as mergeable. Verify first: record the value returned, and whether a merge is refused on it. |  |  |

---

## Phase 2d: merge path (`github:pr-merge`)

Use a new throwaway PR for each of these tests. A merge is final.

| ID | Test | Expected | Result | Evidence |
|---|---|---|--------|----------|
| MRG-01 | No `github-pr-approval` approval | Refused by policy. The PR stays open. |  |  |
| MRG-02 | Approval present, `mergeApproved: false` | Refused by policy. The PR stays open. |  |  |
| MRG-03 | Approval present, but a required status check is failing | GitHub refuses. The error says to check the PR. The PR stays open. |  |  |
| MRG-04 | Approval present, `expectedHeadSha` matches the head | Merged. The merged head matches the approved SHA. |  |  |
| MRG-05 | Approval present, then a new commit pushed to the PR, then merge with the old `expectedHeadSha` | GitHub refuses. The PR stays open with the new commit. |  |  |
| MRG-06 | Same release sent again after MRG-04 | Refused by the nonce store. A repeat at GitHub also fails. |  |  |
| MRG-07 | Merge with `merge` | Merged with a merge commit |  |  |
| MRG-08 | Merge with `squash` | Merged as one squash commit |  |  |
| MRG-09 | Merge with `rebase` | Merged with rebased commits |  |  |
| MRG-10 | Approval for PR #A, merge request targeting PR #B | Refused by policy. PR #B stays open. |  |  |
| MRG-11 | Branch protection requires signed commits, and the PR head is unsigned | GitHub refuses. The PR stays open. |  |  |
| MRG-12 | Caller declares `statusChecksPassed: true` while a check is failing | Still refused. GitHub refuses the merge, whatever the caller declares. |  |  |
| MRG-13 | Caller declares `riskScore` above 20 | Refused by policy |  |  |
| MRG-14 | Approved PR head moves after approval, and the merge request has no `expectedHeadSha` | **Finding to decide.** The policy does not require `expectedHeadSha`, so the merge may go ahead on the new head. Record the result. Decide whether `expectedHeadSha` should be required before go-live. |  |  |
| MRG-15 | Merge a draft PR | GitHub refuses. The PR stays open. Verify first: record the error code. |  |  |
| MRG-16 | Merge a PR with a merge conflict | GitHub refuses. The error says to check the PR. |  |  |
| MRG-17 | Merge a PR whose base branch changed after approval | Verify first: record whether GitHub merges against the new base. |  |  |
| MRG-18 | Merge a PR after its head branch has been deleted | GitHub refuses. Verify first: record the error. |  |  |
| MRG-19 | Merge a closed, unmerged PR | GitHub refuses. |  |  |
| MRG-20 | Merge an already merged PR | GitHub refuses. The connector does not merge twice. |  |  |
| MRG-21 | Caller uses the approver's own account to approve their own PR | Refused by process, not by code. Verify first: confirm the approval flow stops self-approval. |  |  |
| MRG-22 | Approval signed by a key that has since been revoked | Refused. |  |  |
| MRG-23 | Approval signed under an older policy version after the policy changed | Refused. Verify first: record the policy version check. |  |  |

---

## Phase 2e: failure and ambiguity

| ID | Test | Expected | Result | Evidence |
|---|---|---|--------|----------|
| FAL-01 | `PARMANA_CONNECTOR_TIMEOUT_MS=1`, then `pr-fetch` | `timeout`. Nothing is read. |  |  |
| FAL-02 | `PARMANA_CONNECTOR_TIMEOUT_MS=1`, then `pr-merge` on a throwaway PR | `timeout`. Check the PR state. Record whether it merged. |  |  |
| FAL-03 | Network interrupted after the merge request is sent (use the fake or a proxy) | The error says the merge may have reached GitHub, and to check the PR |  |  |
| FAL-04 | GitHub returns HTTP 500 for the merge | `upstream_error`, with the check-PR instruction |  |  |
| FAL-05 | GitHub returns HTTP 401 for the token request | `credential_invalid`. No merge attempted. |  |  |
| FAL-06 | GitHub returns a merge answer with `merged: false` | `upstream_error`. The connector does not report success. |  |  |
| FAL-07 | GitHub returns a merge answer with the body missing | `upstream_error`. No success is reported. |  |  |
| FAL-08 | GitHub returns HTML instead of JSON | `upstream_error`. No success is reported. |  |  |
| FAL-09 | GitHub returns 403 for a secondary rate limit | `upstream_error`. No retry. Record the response headers. |  |  |
| FAL-10 | GitHub returns 429 | `upstream_error`. No retry within the same release. |  |  |
| FAL-11 | GitHub returns 422 on the merge | `upstream_error`. The connector reports the refusal and does not retry. |  |  |
| FAL-12 | GitHub returns a redirect on the merge request | Refused. The credential is not sent to another host. Verify first. |  |  |
| FAL-13 | The Parmana service crashes after the merge request is sent and before the decision is recorded | On restart, the release cannot be replayed. Operator checks the PR. Record the runbook steps. |  |  |
| FAL-14 | Postgres fails after the merge but before the nonce is recorded | Verify first: expected behaviour is that a resend is not merged twice. GitHub refuses the second merge. Record the result. |  |  |
| FAL-15 | **[Lower]** The host clock is ten minutes slow or fast, so the App JWT is rejected | `credential_invalid` or `upstream_error`, with a clear message. No merge or comment is attempted. |  |  |
| FAL-16 | **[Lower]** The App is uninstalled or suspended while a run is in progress | Later calls stop with an error. No success is reported. Record the error code. |  |  |
| FAL-17 | **[Lower]** GitHub returns HTTP 503 | `upstream_error`, with the check-PR instruction. No retry. |  |  |

---

## Phase 2f: concurrency and idempotency

| ID | Test | Expected | Result | Evidence |
|---|---|---|--------|----------|
| CON-01 | Two different approved releases to merge the same PR, sent at once | One merges. The other is refused by GitHub or by policy. The PR is merged once. |  |  |
| CON-02 | Fetch and merge sent for the same PR at once | Fetch reads a consistent state. Merge is refused or merges once. |  |  |
| CON-03 | Twenty parallel `pr-fetch` calls for the same PR | All succeed or fail cleanly. No token is shared between calls. Record the p50 and p95 latency. |  |  |
| CON-04 | Parallel merges on different PRs in the same repository | Each PR is handled on its own. No cross-talk in the trust records. |  |  |
| CON-05 | Parallel calls exceed the GitHub rate limit | Failures are `upstream_error` with no retries. No credential is shown. |  |  |

---

## Phase 2i: comment path (`github:comment`)

Use an open throwaway PR named `CMT throwaway`. Use a separate transaction ID for each case, and record it. Comments do not change the PR's state, so one open PR can hold all of them.

| ID | Test | Expected | Result | Evidence |
|---|---|---|--------|----------|
| CMT-00 | Verify first: an App with only "Pull requests: write" posts a comment | If it returns HTTP 403, the App needs another permission (for example "Issues: write"). Record which one was needed, and update the App setup in the README. |  |  |
| CMT-01 | Valid body, valid transaction ID, open PR | Posted once. The response has `commentId` and `url`. The comment text is the body, a blank line, then the marker. |  |  |
| CMT-02 | Marker search for that transaction ID | Exactly one comment matches |  |  |
| CMT-03 | Body of exactly 4,000 characters | Posted once |  |  |
| CMT-04 | Body of 4,001 characters | Refused as `invalid_input` before any call. No comment appears. |  |  |
| CMT-05 | Empty or whitespace-only body | Refused as `invalid_input` before any call |  |  |
| CMT-06 | Extra parameter, for example `mergeMethod` | Refused before any call |  |  |
| CMT-07 | Missing transaction ID, or one containing `-->` or a space | Refused as `invalid_input` before any call |  |  |
| CMT-08 | Same transaction ID sent again after CMT-01 | Refused by the release ledger. The comment count on the PR stays the same. |  |  |
| CMT-09 | Malformed target, for example `owner/repo` with no number | Refused before any call |  |  |
| CMT-10 | `PARMANA_CONNECTOR_TIMEOUT_MS=1` | The error says the comment may have reached GitHub, and to check the PR. Search for the marker. Record zero or one match, never two. Do not post again. |  |  |
| CMT-11 | Lost answer after the post (fake, or a dropped connection) | As CMT-10. A marker search finds the comment, which is recorded without a second post. |  |  |
| CMT-12 | Token and credential checks | No installation token or credential appears in the response, the error or the comment text. Only a fingerprint appears. |  |  |
| CMT-13 | **[High]** Body that contains a fake marker, `<!-- parmana:btx=<another id> -->` | The marker search for the real transaction ID finds only the real comment. Verify first: record the search result. |  |  |
| CMT-14 | **[High]** Body of 4,000 emoji, which is 16,000 bytes | Verify first: confirm whether the limit counts characters or UTF-16 units. GitHub accepts the comment, which is under its 65,536-byte limit. Record the result. |  |  |
| CMT-15 | **[High]** Target is an issue number that is not a pull request | Refused, or reported as an error. No false success. Verify first: record the response. |  |  |
| CMT-16 | **[High]** Comment on a closed PR, then on a merged PR | Verify first: record whether GitHub accepts each comment. The connector reports GitHub's answer. |  |  |

---

## Phase 2g: evidence and records

| ID | Test | Expected | Result | Evidence |
|---|---|---|--------|----------|
| EVD-01 | Verify each trust record from phases 2c and 2d with `parmana-verifier` | Passes |  |  |
| EVD-02 | Change one byte of a stored record, then verify | Fails |  |  |
| EVD-03 | Change the key ID in a record, then verify | Fails |  |  |
| EVD-04 | Search every record for the installation token and the PEM | Not found |  |  |
| EVD-05 | Each refused request has a refusal record, with the reason | Present |  |  |
| EVD-06 | Each merge has a record of the head SHA at the time of approval and at the time of merge | Present, and the two are compared. Verify first. |  |  |
| EVD-07 | Records are append-only: an attempt to overwrite or delete one is refused | Refused. Verify first: confirm the storage rule. |  |  |
| EVD-08 | Record for a timed-out merge is written, and marked as ambiguous | Present, with the ambiguity marked |  |  |

---

## Phase 2h: security

| ID | Test | Expected | Result | Evidence |
|---|---|---|--------|----------|
| SEC-01 | Secret scanning on this repository and on `parmana-connectors` | No credential, PEM, `.env` or database URL found |  |  |
| SEC-02 | Logs during a full run contain no installation token, JWT, PEM or database URL | Not found |  |  |
| SEC-03 | Error messages contain no credential value | Not found, for every error code in the connector |  |  |
| SEC-04 | Target or parameter containing a string that looks like a command or a header | Treated as data. No effect on the request. |  |  |
| SEC-05 | Target pointing at another host, such as `evil.example/owner/repo#1` | Refused by the target format |  |  |
| SEC-06 | Base URL cannot be changed by a release | Confirmed. Only the configured GitHub API is called. |  |  |
| SEC-07 | Branch protection on `main` lists the required status check `test`, and force pushes are blocked | Confirmed and recorded |  |  |
| SEC-08 | The ruleset rule that blocked PR #1 is identified and recorded in the README | Recorded |  |  |
| SEC-09 | Host isolation (G-128): the connector host is controlled and its environment is restricted | Confirmed and recorded |  |  |
| SEC-10 | Routing (G-129): nothing else calls GitHub for these actions | Confirmed. Only the gateway holds the App credential. |  |  |
| SEC-11 | Caller-declared signals: a caller declaring `requiredReviewsCompleted: true` on a PR with no reviews | Still refused by GitHub, because the connector does not authorize on the declaration alone. Verify first with the branch rules. |  |  |

---

## Operational drills

| ID | Test | Expected | Result | Evidence |
|---|---|---|--------|----------|
| OPS-01 | Rotate the App private key: generate a new key, update the credential, redeploy | New key works. The old key fails. Time the steps. |  |  |
| OPS-02 | Revoke the App from this repository | All calls fail at once. Time the steps. |  |  |
| OPS-03 | Rotate the Parmana signing key | New records verify with the new key. Old records still verify with the old key. |  |  |
| OPS-04 | Revoke an approver's key | New approvals from that key are refused. Old records still verify. |  |  |
| OPS-05 | Follow the check-PR runbook for a simulated ambiguous merge | The named person checks the PR and records the outcome |  |  |
| OPS-06 | Restart the service during a fetch | The next call succeeds. No state is lost. |  |  |
| OPS-07 | Postgres failover during a run | Releases are refused while the nonce store is unavailable. No duplicate merge. |  |  |
| OPS-08 | GitHub outage simulated with the fake | Calls fail with `upstream_error` or `timeout`. The runbook is followed. |  |  |
| OPS-09 | An alert fires for an `upstream_error` that says to check the PR | The alert reaches the named owner. Time the response. |  |  |
| OPS-10 | Rollback: `git revert` on the test repo's `main` after a test merge | Revert merged. The trust record of the original merge still verifies. |  |  |
