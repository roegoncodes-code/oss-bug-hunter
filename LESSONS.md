# LESSONS — what the nightly runs have learned

The nightly run reads this before auditing and appends to it after every phase. Keep entries short and dated.

## Working build and test commands

- 2026-11-19 netty (4.2): JDK 17 works. `./mvnw -B -ntp -pl codec-http -am test -Dtest='<Class>#<method>' -Dsurefire.failIfNoSpecifiedTests=false -Danimal.sniffer.skip=true -Drevapi.skip=true -Djapicmp.skip=true` (~3 min on Actions). Checkstyle did NOT run with these flags; upstream wants `./mvnw verify -DskipTests` for checkstyle/revapi.
- 2026-11-19 spring-framework (main): JDK 25 (from .sdkmanrc). `./gradlew :spring-core:test --tests '<pattern>'`; add `:spring-core:checkstyleMain :spring-core:checkstyleTest` to the module command (~2 min on Actions).

<!-- - YYYY-MM-DD kafka: `./gradlew :clients:test --tests <Class>` works; needs JDK 21 -->

## False alarms (do not flag again)

- 2026-11-19 netty HttpContentDecoder: NOT completing a truncated body that is delimited by the end of the connection is deliberate upstream (testTruncatedGzipResponseFails). Read the intent of existing tests before choosing a fix.

<!-- - YYYY-MM-DD project file:line — why it is not a bug -->

## Known upstream issues (already reported)

- 2026-11-19 netty #18445 (merged) introduced the JdkZlibDecoder truncation failure; #18471 is its open 4.1 port.

<!-- - YYYY-MM-DD project — issue/PR link — what it covers -->

## Audited areas

- 2026-11-19 kafka clients/consumer/internals (share consumer leadership, subscription-state refactor, streams heartbeat): no defects found.
- 2026-11-19 tomcat java/org/apache/coyote/http2 (sendfile error/reset, invalid response headers, INITIAL_WINDOW_SIZE): see private report.
- 2026-11-19 spring-framework spring-core (aot.generate, util, core.io.buffer, ConcurrentLruCache, ProfilesParser, MimeTypeUtils, NumberUtils): 2 verified findings (PR #2, #3).
- 2026-11-19 netty codec-http (HttpContentDecoder, HttpContentCompressor, websocketx origin checks): 1 verified finding (PR #1).
- 2026-11-19 jackson-databind 3.x ser/deser (#6299, #6437, #6219, #6357, any-setter, PatternDeserializer, Map.Entry): no defects found.

<!-- - YYYY-MM-DD project/module — files or packages covered -->

## Open leads (continue these first)

- 2026-11-19 netty SpdySessionHandler (older code): when the stream window goes negative, the newWindowSize < 0 branch writes a sliced received data frame back to the peer instead of passing it up. Worth a check.
- 2026-11-19 netty HttpContentCompressor: explicit low-q encoding may beat a q=1 wildcard (commit 5a638e4); encoding names matched by substring.
- 2026-11-19 netty websocketx: Origin rejection (0b60a7d) never completes handshakePromise; allow-list entries with trailing slash/default port never match.
- 2026-11-19 jackson-databind: BinaryFloatArraySerializer.createContextual returns the shared instance; SimpleFilterProvider copy constructor NPE on null map values.
- 2026-11-19 spring SimpleAliasRegistry: aliasNames ArrayList can collect duplicates when aliases are overridden.

<!-- - YYYY-MM-DD project file:line — suspicion, evidence so far, next step -->

## CI and infrastructure

- 2026-11-19 GitHub from the session: the preset GH_TOKEN is invalid; run `env -u GH_TOKEN gh ...` (the proxy authenticates as roegoncodes-code). GraphQL is blocked, so `gh search`, `gh pr create` and `gh workflow run` fail: use REST via `gh api` (dispatch: `POST repos/<repo>/actions/workflows/verify.yml/dispatches`; PRs: `POST repos/<repo>/pulls`; search: `GET search/issues` with `is:pr`/`is:issue` in q).
- 2026-11-19 First scheduled run (01:02 IST) ended after 17 s with no output and no notification; the task prompt now avoids depending on add_repo and must notify on every exit.
- 2026-11-19 Netty proof 3 first failed because the chosen fix broke a deliberate upstream test; the fix was adjusted (connection-active check) and re-verified.

<!-- - YYYY-MM-DD what broke and how it was fixed -->
