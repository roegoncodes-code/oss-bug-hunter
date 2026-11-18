# oss-bug-hunter

A nightly, automated hunt for real bugs in top open-source Java projects such as Kafka, Tomcat, Spring and Netty.

Each night a scheduled Claude run audits recently merged upstream code, writes a failing test and a fix for each suspected bug, and proves the fix on GitHub Actions: the test must fail without the fix and pass with it. Verified findings arrive as PRs in this repo, each with a report of the problem, the root cause, the solutions compared and the CI proof.

Nothing is posted to the upstream projects automatically. Each fix is reviewed by hand before it is submitted upstream.

- `PLAYBOOK.md` — the rules: targets, rotation, bug patterns, budgets, proof standard.
- `LESSONS.md` — what earlier runs learned.
- `runs/log.md` — one line per night.
- `findings/` — test and fix patches for each verified finding.
- `.github/workflows/verify.yml` — the test VM.
