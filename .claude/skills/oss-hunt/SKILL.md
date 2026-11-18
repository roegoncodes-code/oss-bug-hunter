---
name: oss-hunt
description: Nightly bug hunt in top open-source Java projects (Kafka, Tomcat, Spring…): Opus audits, Sonnet fixes with TDD, GitHub Actions verifies, Haiku fixes CI, then PR + report.
---

# /oss-hunt — nightly open-source bug hunt

Runs unattended at night. Finds up to 3 **real, test-proven** bugs in recently merged code of a top open-source Java project, fixes them, verifies the fixes on a GitHub Actions VM, and reports. Nobody is watching: never ask questions; decide, log the decision, and continue.

## Inputs

- **Hunt repo:** `roegoncodes-code/oss-bug-hunter` (public). It contains:
  - `PLAYBOOK.md` — target lists, rotation, bug patterns, budgets. Read it first; it overrides defaults here.
  - `LESSONS.md` — what earlier runs learned: working test commands, false alarms, audited areas, known upstream issues, open leads.
  - `runs/log.md` — one line per run.
  - `.github/workflows/verify.yml` — the test VM. Inputs: `project`, `upstream_ref`, `finding_dir`, `test_cmd`, optional `module_test_cmd` and `java_version`. It checks out upstream at `upstream_ref`, proves the new test **fails without the fix** (and not by a compile error), applies the fix, and proves the test (and module tests, if given) **pass with the fix**.
  - `findings/<date>-<project>-<slug>/` — `test.patch`, `fix.patch`, `REPORT.md` per verified finding.
- Upstream projects are cloned read-only from public GitHub. Never push, comment, open issues or open PRs upstream.

## Hard rules

1. **Proof or nothing.** A finding counts only if `verify.yml` passes. Unproven suspects go to `LESSONS.md` as leads, never into a PR.
2. **Security findings stay private.** A suspected vulnerability never goes into the hunt repo, a commit, a branch name or a PR. It appears only in the private report artifact and in a separate push notification starting with `SECURITY:`, with the project's private reporting route from its `SECURITY.md`.
3. **Time budgets are hard limits.** Opus and Sonnet run **once** per night. Only Haiku may be re-run.
4. **Up to 3 confirmed findings.** Zero is a valid night. Never pad with weak findings.
5. **Rules change only by PR.** Update `LESSONS.md` directly. Propose `PLAYBOOK.md` changes as a PR; never commit them to `main`.
6. **Commit `LESSONS.md` after every phase**, so a run cut off by the usage limit still leaves its leads for the next night.
7. Run the phases in order. When a phase leaves nothing to continue with, jump to Phase 6, which always runs.

## Time budget enforcement

Start each timed sub-agent with the Agent tool: `run_in_background: true`, a fixed `name`, and the `model` below. Tell it its soft deadline, that it must append progress to its work file as it goes, and that the last line must be `STATUS: DONE`. Then wait in the foreground with Bash (each blocking wait costs almost no tokens):

```bash
timeout <hard_seconds> bash -c 'until grep -q "STATUS: DONE" <work_file>; do sleep 15; done'
```

If the wait times out, stop the agent with TaskStop (by its name) and continue with whatever the work file contains. Sub-agents start cold: pass them the work-file paths and everything they need in the prompt.

| Phase | Agent name | Model | Soft target | Hard stop | Re-runs |
|---|---|---|---|---|---|
| 1 Audit | `auditor` | opus | 7 min | 10 min | none |
| 2 Implement | `implementer` | sonnet | 7 min | 15 min | none |
| 4 CI fix | `ci-fixer` | haiku | 4 min | 5 min | up to 3 per finding |

## Phase 0 — Setup (orchestrator)

1. Note the start time. Clone the hunt repo. Read `PLAYBOOK.md`, then `LESSONS.md`.
2. **Backlog check:** count open PRs in the hunt repo whose branch starts with `hunt/`. If there are 5 or more, skip the hunt: send a push notification `5+ hunt PRs waiting for review — hunt paused tonight`, log the night in `runs/log.md`, and stop.
3. **Pick the target:** Monday–Friday (IST) from the core list, Saturday–Sunday from the broad list in `PLAYBOOK.md`, rotating to the project audited least recently. An open lead in `LESSONS.md` for a project in tonight's list goes first.
4. Shallow-clone the upstream project (`git clone --filter=blob:none`) and record HEAD as `upstream_ref`. If the clone fails, record the error and jump to Phase 6.
5. **Scope:** files changed by commits merged in the last 30 days (`git log --since=30.days --name-only`) inside one module. Skip test-only, docs and generated files, and areas `LESSONS.md` marks as already audited.
6. Write `work/scope.md`: project, `upstream_ref`, module, file list, relevant lessons.

## Phase 1 — Audit (Opus, 7 min target, 10 min hard)

Start `auditor` (model `opus`). It follows the `superpowers:systematic-debugging` discipline: evidence before conclusions. Instructions:

- Read `work/scope.md` and the scoped code. Hunt for real defects: concurrency and ordering bugs, lost or duplicated messages, wrong retry/backoff, resource leaks, unchecked input, deserialization and auth gaps, off-by-one and overflow, error paths that swallow failures, behaviour that contradicts the javadoc or the commit message.
- For each suspect (max 3), append to `work/plan.md`: location (`file:line`), the defect, the evidence, why existing tests miss it, a **concrete failing test** (class, method, assertion) that compiles against upstream without the fix, 2–3 solution options with trade-offs, the chosen option, `security: yes/no`, and confidence high/medium/low.
- Put low-confidence suspects in a `## Leads` section.
- At ~7 min, stop exploring and finish the write-up. Last line: `STATUS: DONE`.

After the agent ends, for each high/medium suspect, search the upstream issue tracker and open PRs for the same bug (GitHub search via `gh`, or the project's Jira/Bugzilla via web search). **Already reported → drop it** and log it in `LESSONS.md` as known. Commit `LESSONS.md`. If no suspect remains, jump to Phase 6.

## Phase 2 — Implement (Sonnet, 7 min target, 15 min hard)

Start `implementer` (model `sonnet`) once for all remaining suspects. It follows `superpowers:test-driven-development`: write the failing test first, then the smallest fix matching the chosen option, in the upstream project's code style. For each suspect, write `findings/<date>-<project>-<slug>/test.patch` and `fix.patch` (`git diff` against `upstream_ref`; the test patch must not depend on the fix) and record the narrowest test command and the JDK version in `work/impl.md`. The build cannot run in this workspace, so it self-checks by reading: imports, signatures, and that the test really exercises the bug. Last line of `work/impl.md`: `STATUS: DONE`. Suspects without finished patches become leads.

**Security suspects:** keep their patches only under `work/private/`, never in `findings/`, and never push them. They are reported in Phase 6, marked unverified.

Commit `LESSONS.md`.

## Phase 3 — Verify on the GitHub Actions VM (orchestrator)

For each non-security finding:

1. Commit its `findings/...` folder to branch `hunt/<date>-<project>-<slug>` and push.
2. `gh workflow run verify.yml --ref <branch> -f project=<owner/repo> -f upstream_ref=... -f finding_dir=... -f test_cmd=... [-f module_test_cmd=...] [-f java_version=...]`
3. Wait up to **45 minutes total**, as blocking waits of at most 10 minutes each: `timeout 590 gh run watch <id> --exit-status`.
4. Pass = all proof steps green. Failure → Phase 4. Still running after 45 minutes → cancel the run, drop the finding, and add a lesson to use a narrower test command.

## Phase 4 — CI fixes (Haiku, 5 min each, up to 3 runs per finding)

Start `ci-fixer` (model `haiku`) with the trimmed failed log (`gh run view <id> --log-failed`) and the patches. It first classifies the failure:

- **Mechanical** (compile error, missing import, checkstyle/spotless, wrong test command or JDK, a flaky unrelated test): fix the patch, describe the change in `work/ci-<slug>.md`, then `STATUS: DONE`. The orchestrator re-runs Phase 3 for that finding.
- **Logic** (wrong assertion, the test passes without the fix, different behaviour): do not fix. Write the diagnosis and `STATUS: DONE`. The orchestrator drops the finding and records why in `LESSONS.md`.

After 3 Haiku runs without a pass, drop the finding and record why. Commit `LESSONS.md`.

## Phase 5 — Review (orchestrator, independent of the writers)

Review each verified diff with the `engineering:code-review` skill: correctness, edge cases, upstream style, minimal scope. Fix only trivial issues, and re-verify through Phase 3 if code changed. Apply `superpowers:verification-before-completion` before calling anything verified: quote the CI run URL as evidence.

## Phase 6 — Publish and learn (always runs)

1. **PRs:** for each verified finding, open a PR in the hunt repo from its branch with `REPORT.md` as the body: problem, root cause, solutions compared, chosen fix, CI run link, and how to submit upstream (that project's route, e.g. Kafka: Jira ticket + GitHub PR; Tomcat: Bugzilla or GitHub PR; OpenJDK: needs the OCA).
2. **Report artifact:** publish one private HTML artifact titled `OSS hunt — <date>`: one section per verified finding (problem, root cause, solutions compared, fix, test proof, PR link); a private security section if any; dropped findings with reasons; leads. On a zero-finding night, list what was audited, what was ruled out and why, and any infrastructure error.
3. **Learn:** append to `LESSONS.md`: working build/test commands, false alarms and why, audited areas, known upstream issues, open leads. Put any rule change in a `PLAYBOOK.md` PR. Append to `runs/log.md`: date, project, module, phase durations, suspects, verified count, outcome. Commit and push to `main`.
4. **Notify:** one push with a one-line summary (e.g. `2 verified bugs in Kafka Connect, 1 lead — report ready`, or `No verified bugs in Netty tonight — 3 areas ruled out`). If there is a security suspect, send a **separate** push starting with `SECURITY:`.
