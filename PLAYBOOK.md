# PLAYBOOK — rules for /oss-hunt

The nightly run reads this file first. It overrides the skill's defaults. Change it only through a PR.

## Targets

### Core list (Monday–Friday, IST)

| Project | Upstream repo | Build | JDK |
|---|---|---|---|
| Kafka | apache/kafka | Gradle (`./gradlew`) | 21 |
| Tomcat | apache/tomcat | Ant | 21 |
| Spring Framework | spring-projects/spring-framework | Gradle | 21 |
| Spring for Apache Kafka | spring-projects/spring-kafka | Gradle | 21 |
| Netty | netty/netty | Maven (`./mvnw`) | 21 |
| Jackson Databind | FasterXML/jackson-databind | Maven | 21 |

### Broad list (Saturday–Sunday, IST)

google/guava, apache/commons-lang, apache/commons-io, apache/commons-collections, apache/camel, hibernate/hibernate-orm, apache/zookeeper, jetty/jetty.project, apache/httpcomponents-client, apache/logging-log4j2, qos-ch/logback, google/gson, junit-team/junit5, mockito/mockito, resilience4j/resilience4j, micrometer-metrics/micrometer, apache/activemq, apache/dubbo, reactor/reactor-core, apache/maven, apache/lucene, grpc/grpc-java, FasterXML/jackson-core, spring-projects/spring-boot.

Excluded for now (builds too heavy for the CI VM): OpenJDK, Elasticsearch, Flink, Cassandra, Pulsar, Quarkus.

### Rotation

- Pick the project in tonight's list that was audited least recently, according to `runs/log.md`.
- An open lead in `LESSONS.md` for a project in tonight's list takes priority.
- Check the JDK version in the project's build files before trusting the table above.

## Scope

- One module per night.
- Only code merged upstream in the last 30 days.
- Skip tests, docs, generated code, and areas `LESSONS.md` marks as audited.

## Bug patterns to hunt

- Concurrency: races, missing synchronization or volatile, check-then-act, unsafe publication.
- Messaging: lost, duplicated or reordered records; offset/commit mistakes; wrong retry or backoff.
- Resources: leaked streams, sockets, threads, buffers; missing `close` on error paths.
- Input handling: missing bounds or null checks, off-by-one, integer overflow, charset bugs.
- Error handling: swallowed exceptions, wrong exception type, state left half-updated.
- Contract drift: code that contradicts its javadoc, its config docs, or its commit message.
- Security-relevant: deserialization, path traversal, header/request smuggling, auth bypass. These follow the private security rule.

## Budgets

| Stage | Model | Soft target | Hard stop | Re-runs |
|---|---|---|---|---|
| Audit | Opus | 7 min | 10 min | none |
| Implement | Sonnet | 7 min | 15 min | none |
| CI fix | Haiku | 4 min | 5 min | up to 3 per finding |
| CI wait | — | — | 45 min per finding | — |

- Maximum 3 suspects from the audit, maximum 3 verified findings per night.
- Pause the hunt when 5 or more `hunt/` PRs are open.

## Proof standard

A finding is verified only when `verify.yml` passes: the new test fails on upstream without the fix (and is not a compile error), then passes with the fix, and the module tests pass when a module command is given.

## Security rule

A suspected vulnerability never goes into this repo, a branch name, a commit or a PR, because this repo is public. It appears only in the private report and a separate push starting with `SECURITY:`, with the project's private reporting route from its `SECURITY.md`.

## Upstream

Never post anything upstream. The user submits accepted fixes himself.
