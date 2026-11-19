# Spring Framework: `NameGenerator.generateResourcePath` can return the same path twice

**Project:** spring-projects/spring-framework, branch main, at `7ce40c871aafa90fec2d3147fdb9c3c0cdcf26ca`
**Area:** `spring-core`, `org.springframework.aot.generate.NameGenerator`
**Status:** verified on GitHub Actions, including Checkstyle — [run 44483814331](https://github.com/roegoncodes-code/oss-bug-hunter/actions/runs/44483814331)

## Problem

`NameGenerator` promises unique generated names. Uniqueness comes from appending a counter to the root name, and the counter is tracked per root name. For class names this is safe, because `clean()` strips digits from the root. Feature names used for generated resource paths are not cleaned, so a feature name that ends with a digit can collide with a suffixed name:

- `generateResourcePath("json", "metadata", null)` twice gives `…-metadata.json`, then `…-metadata1.json`.
- `generateResourcePath("json", "metadata1", null)` then gives `…-metadata1.json` again.

The collision also happens in the other order (`metadata1` first, then `metadata` twice). `GeneratedResources.writeTo` would then create the same file twice.

The resource-generation feature is recent (#35862, "Add support for generating resources linked to a feature"); no existing issue or PR covers this.

## Root cause

`NameGenerator.generateUniqueName` appends the sequence number without checking that the resulting name is not itself a root name already handed out.

## Solutions compared

1. **Use a separator such as `-1`.** Avoids most collisions but changes every existing generated path — a compatibility break for AOT output.
2. **Clean digits out of feature names.** Strips legitimate digits that users chose.
3. **Chosen:** reserve each suffixed name in the same sequence map with `putIfAbsent`; if the candidate is already taken, move to the next number. Existing outputs (`bytes`, `bytes1`, `bytes2`) are unchanged.

## The fix

See `fix.patch` (12 lines in `NameGenerator.generateUniqueName`). Tests in `test.patch`: two new tests in `NameGeneratorTests.ResourcePathTests`, one per collision order.

## Proof

- **Without the fix:** `generateResourcePathIsUniqueWhenFeatureNameEndsWithDigit` fails with an `AssertionError` (duplicate path).
- **With the fix:** both new tests pass, all `org.springframework.aot.generate` tests pass, and `checkstyleMain` / `checkstyleTest` pass.

## How to submit upstream

1. Spring takes a PR directly, without an issue first, against `main`. Every commit needs a `Signed-off-by` trailer (DCO: `git commit -s`).
2. Commit subject: imperative, at most 55 characters, no prefix — e.g. `Avoid duplicate resource paths in NameGenerator`.
3. Run `./gradlew :spring-core:check` locally (JDK 25 from `.sdkmanrc`).
4. Spring's CONTRIBUTING.md asks contributors to review changes produced by coding agents carefully and not to submit from bot-managed accounts. Review it yourself and describe it in your own words.
