# test-ruleset-precedence

Repo used to verify how GitHub reports branch protection when classic branch
protection and repository rulesets both exist. It is the evidence for
[Your branch protection is not where you think it is](https://cameronbrooks11.github.io/devops/2026/09/12/branch-protection-is-not-where-you-think/).
**Do not delete:** the post links here.

Corrections, counter-examples and paid-plan data points are welcome as
[issues](https://github.com/baseliner-sandbox/test-ruleset-precedence/issues).

Tested 2026-09-12 against a **free** org (`baseliner-sandbox`).

## What was verified

**1. No single endpoint reports a branch's effective protection.**

| State | `GET /branches/main/protection` | `GET /rules/branches/main` |
|---|---|---|
| classic only, 1 approval required | `required_approving_review_count: 1` | `[]` |
| classic (1) + ruleset (2), both active | `1` | `2` |

A tool reading either endpoint alone gets a wrong answer, and the two errors
point in opposite directions. `/rules/branches/main` returns an empty array on a
branch that is genuinely protected by classic rules.

**2. `bypass_mode: exempt` is invisible in the effective-rules view.**
The rule is reported with `required_approving_review_count: 2` and the payload
contains no mention of `bypass` or `exempt`. Bypass configuration is only visible
via a separate, admin-scoped `GET /rulesets/{id}`. Per GitHub's OpenAPI spec, an
`exempt` bypass also produces no bypass audit entry.

**3. Plan gating makes the config unreadable, not just unenforced.**
On a free org, making the repo private causes both endpoints to return
**HTTP 403** — *"Upgrade to GitHub Pro or make this repository public to enable
this feature."* The configuration still exists; it cannot be read and is not
enforced.

**4. On a free org, changing repo visibility destroys classic branch protection**
(reproduced 2/2). Before: `approvals=1 enforce_admins=true checks=1`. After a
private → public round-trip: `404 Branch not protected`. Rulesets survive the
same toggle. No warning is given, and the trigger is an administrative action
unrelated to branch protection.

*Paid-plan data point, 2026-10-07:* on a **GitHub Team** org the same protection
survived the same round trip (2/2) and stayed readable while private:
`approvals=1 enforce_admins=true checks=1` before, while private, and after. The
test used a throwaway repo, deleted afterwards. So this finding is the free
tier's forced removal on the way to private, never restored on the way back.

**5. Required-check app pinning uses different field names** in each system:
rulesets use `integration_id`, classic protection uses `app_id`.

## Refuted hypotheses

- `/rules/branches/{branch}` does **not** filter by the calling actor's bypass
  status — `exempt`, `always` and no-bypass all report identically.
- `PUT /rulesets/{id}` with a partial body does **not** wipe `bypass_actors`.

## Not tested

Real enforcement with two distinct approving identities, so community report
#164106 (a ruleset's 2-approval requirement apparently losing to a legacy
1-approval rule) is neither confirmed nor refuted here.
