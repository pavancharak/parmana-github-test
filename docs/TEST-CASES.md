# GitHub connector test cases

Test cases for the Parmana GitHub connector (`github:pr-fetch` and `github:pr-merge`), run against this repository before the connector goes live. The phases, exit gates and rules are in the [README](../README.md) and [go-live test plan](../README.md#staging-tests).

## How to record a result

For each test, record:

- **Result:** Pass, Fail, or Blocked (with the reason)
- **Evidence:** run ID, error code, PR number, PR state before and after
- **Tester and date**

A test that merges a PR must use a new throwaway PR. Never reuse a merged PR.

## Preconditions

- GitHub App installed on this repository only, with the permissions in the connector README
- `GITHUB_APP_CREDENTIAL`, `PARMANA_PUBLIC_KEY`, `PARMANA_RELEASE_AUDIENCE` and `PARMANA_NONCE_STORE=postgres` set for the test service
- `PARMANA_ALLOW_LIVE_KEYS=false`
- Policies `github-pr-read` 1.1.0 and `github-pr-approval` 1.1.0 in effect
- A named approver whose signing key is on their own machine

---

## Phase 1: automated tests (in `parmana-connectors`)

| ID | Test | Expected |
|---|---|---|
| AUT-01 | `npm ci`, `npm run typecheck`, `npm run lint`, `npm test` on a clean checkout | All pass. Record the test count. |
| AUT-02 | GitHub connector tests (`tests/connectors/github/`) | Pass, with no skipped GitHub tests |
| AUT-03 | A `mergeMethod` other than `merge`, `squash` or `rebase` | Refused as `invalid_input` before a call |
| AUT-04 | `expectedHeadSha` that is not 40 lowercase hex characters | Refused as `invalid_input` before a call |
| AUT-05 | A parameter not allowed for the capability, for example `foo` on `pr-fetch` | Refused before a call |
| AUT-06 | A target that is not `owner/repo#number` | Refused as `invalid_input` before a call |
| AUT-07 | A capability other than `github:pr-fetch` or `github:pr-merge` | Refused as `unknown_capability` |

---

## Phase 2a: credential and authentication

| ID | Test | Expected |
|---|---|---|
| CRD-01 | Valid credential, `pr-fetch` on an open PR | Success, with `headSha` and a `tokenFingerprint` |
| CRD-02 | Wrong installation id | `upstream_error` or `credential_invalid`. The message does not show the credential. |
| CRD-03 | Wrong App id | As CRD-02 |
| CRD-04 | Credential with two parts, or a value that is not a PEM after base64 decoding | `credential_invalid`, with no call to GitHub |
| CRD-05 | Search the service logs and the trust records for the installation token | Not found. Only fingerprints appear. |
| CRD-06 | Revoke the App installation, then call `pr-fetch` | Fails. Record the error. |
| CRD-07 | Reinstall the App, then call `pr-fetch` | Succeeds again |

---

## Phase 2b: read path (`github:pr-fetch`)

| ID | Test | Expected |
|---|---|---|
| RD-01 | Valid `github-pr-read` approval for the open PR | Success, returning `mergeable`, `headSha`, `baseRef` and `mergedAt` |
| RD-02 | No approval | Refused by policy. No call to GitHub. |
| RD-03 | Approval for a different PR number | Refused by policy |
| RD-04 | Approval with `readApproved: false` | Refused by policy |
| RD-05 | Approval signed by a key that is not trusted | Refused. Nothing is read. |
| RD-06 | Approval altered after signing (for example, the PR number changed) | Refused. The signature check fails. |
| RD-07 | Read a PR that is already merged | Success, with `mergedAt` set |
| RD-08 | Read a PR number that does not exist | `upstream_error`. No token or credential is shown. |

---

## Phase 2c: merge path (`github:pr-merge`)

Use a new throwaway PR for each of these tests. A merge is final.

| ID | Test | Expected |
|---|---|---|
| MRG-01 | No `github-pr-approval` approval | Refused by policy. The PR stays open. |
| MRG-02 | Approval present, `mergeApproved: false` | Refused by policy. The PR stays open. |
| MRG-03 | Approval present, but a required status check is failing | GitHub refuses. The error says to check the PR. The PR stays open. |
| MRG-04 | Approval present, `expectedHeadSha` matches the head | Merged. The merged head matches the approved SHA. |
| MRG-05 | Approval present, then a new commit pushed to the PR, then merge with the old `expectedHeadSha` | GitHub refuses. The PR stays open with the new commit. |
| MRG-06 | Same release sent again after MRG-04 | Refused by the nonce store. A repeat at GitHub also fails. |
| MRG-07 | Merge with `merge` | Merged with a merge commit |
| MRG-08 | Merge with `squash` | Merged as one squash commit |
| MRG-09 | Merge with `rebase` | Merged with rebased commits |
| MRG-10 | Approval for PR #A, merge request targeting PR #B | Refused by policy. PR #B stays open. |
| MRG-11 | Branch protection requires signed commits, and the PR head is unsigned | GitHub refuses. The PR stays open. |
| MRG-12 | Caller declares `statusChecksPassed: true` while a check is failing | Still refused. GitHub refuses the merge, whatever the caller declares. |
| MRG-13 | Caller declares `riskScore` above 20 | Refused by policy |

---

## Phase 2d: failure and ambiguity

| ID | Test | Expected |
|---|---|---|
| FAL-01 | `PARMANA_CONNECTOR_TIMEOUT_MS=1`, then `pr-fetch` | `timeout`. Nothing is read. |
| FAL-02 | `PARMANA_CONNECTOR_TIMEOUT_MS=1`, then `pr-merge` on a throwaway PR | `timeout`. Check the PR state. Record whether it merged. |
| FAL-03 | Network interrupted after the merge request is sent (use the fake or a proxy) | The error says the merge may have reached GitHub, and to check the PR |
| FAL-04 | GitHub returns HTTP 500 for the merge | `upstream_error`, with the check-PR instruction |
| FAL-05 | GitHub returns HTTP 401 for the token request | `credential_invalid`. No merge attempted. |
| FAL-06 | GitHub returns a merge answer with `merged: false` | `upstream_error`. The connector does not report success. |

---

## Phase 2e: evidence

| ID | Test | Expected |
|---|---|---|
| EVD-01 | Verify each trust record from phases 2b and 2c with `parmana-verifier` | Passes |
| EVD-02 | Change one byte of a stored record, then verify | Fails |
| EVD-03 | Change the key ID in a record, then verify | Fails |
| EVD-04 | Search every record for the installation token and the PEM | Not found |
| EVD-05 | Each refused request has a refusal record, with the reason | Present |

---

## Security checks before phase 5

| ID | Test | Expected |
|---|---|---|
| SEC-01 | Secret scanning on this repository and on `parmana-connectors` | No credential, PEM, `.env` or database URL found |
| SEC-02 | Host isolation (G-128): confirm the connector host is controlled and its environment is restricted | Confirmed and recorded |
| SEC-03 | Routing (G-129): confirm nothing else calls GitHub for these actions | Confirmed. Only the gateway holds the App credential. |
| SEC-04 | Branch protection on `main` lists the required status check `test`, and force pushes are blocked | Confirmed and recorded |
| SEC-05 | The ruleset rule that blocked PR #1 is identified and recorded in the README | Recorded |

---

## Operational drills

| ID | Test | Expected |
|---|---|---|
| OPS-01 | Rotate the App private key: generate a new key, update the credential, redeploy | New key works. The old key fails. Time the steps. |
| OPS-02 | Revoke the App from this repository | All calls fail at once. Time the steps. |
| OPS-03 | Rotate the Parmana signing key | New records verify with the new key. Old records still verify with the old key. |
| OPS-04 | Follow the check-PR runbook for a simulated ambiguous merge | The named person checks the PR and records the outcome |
