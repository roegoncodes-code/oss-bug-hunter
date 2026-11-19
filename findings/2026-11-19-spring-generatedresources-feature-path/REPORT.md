# Spring Framework: `GeneratedResources.getOrAdd` ignores resources added for a feature

**Project:** spring-projects/spring-framework, branch main, at `7ce40c871aafa90fec2d3147fdb9c3c0cdcf26ca`
**Area:** `spring-core`, `org.springframework.aot.generate.GeneratedResources`
**Status:** verified on GitHub Actions, including Checkstyle — [run 44483818141](https://github.com/roegoncodes-code/oss-bug-hunter/actions/runs/44483818141)
**Confidence that maintainers see this as a bug:** medium (see below)

## Problem

`addForFeature(...)` creates a resource and adds it to the resource list, but never registers its path in `resourcesByPath`. A later `getOrAdd(path)` for the same path therefore creates a **second** `GeneratedResource` with the same path instead of returning the existing one, and both are written out.

## Root cause

`GeneratedResources.createAndAddGeneratedResource` adds to `resources` only; `getOrAdd` looks up `resourcesByPath` only.

## Solutions compared

1. **Treat the two APIs as separate namespaces and document it.** Keeps the duplicate-file problem.
2. **Make `NameGenerator` aware of paths registered through `getOrAdd`.** Covers both orders, but couples the two classes.
3. **Chosen:** register the feature-based path with `resourcesByPath.putIfAbsent`. One line; covers the feature-then-path order.

Limitation: if `getOrAdd(path)` is called first and a later `addForFeature` generates the same path, the duplicate remains. Closing that needs option 2.

## The fix

See `fix.patch` (1 line). Test in `test.patch`: `GeneratedResourcesTests.getOrAddWithPathOfFeatureResourceReturnsSameInstance`.

## Proof

- **Without the fix:** the new test fails with an `AssertionError` (a different instance is returned).
- **With the fix:** the test passes, all `org.springframework.aot.generate` tests pass, and Checkstyle passes.

## Why medium confidence

Maintainers may consider path-based and feature-based resources deliberately separate. Spring's CONTRIBUTING.md also asks not to submit "straightforward or polish-style changes". Consider submitting this together with the `NameGenerator` fix as one PR about duplicate generated resources, or ask in that PR whether they want it.

## How to submit upstream

Same as the `NameGenerator` finding: PR against `main`, `Signed-off-by` trailer on each commit, `./gradlew :spring-core:check` locally, review and describe it in your own words.
