# PostgresNIO closed-channel query fix

Base: upstream [vapor/postgres-nio 1.33.2](https://github.com/vapor/postgres-nio/releases/tag/1.33.2), commit `649adba966158859f4fb1767e4d11802966d3fb4`. The upstream default `main` remains intact; this fork's version branch is `sauna-1.33.2`. Upstream license and public API are unchanged.

An async query writes a task whose query-result promise is independent from the channel write. When a closed channel rejects the write before it reaches the PostgreSQL handler, the result promise previously remained pending. The minimal fix forwards that write failure as a `clientClosedConnection` PSQLError and acknowledges write acceptance inside the handler. Query completion still belongs to the original query context. An accepted write promise is completed, preventing it from leaking.

Existing upstream history: [PR487](https://github.com/vapor/postgres-nio/pull/487) addressed closed-channel hangs; the earlier changes were subsequently reverted in [PR501](https://github.com/vapor/postgres-nio/pull/501) following the promise/memory concerns discussed in [issue496](https://github.com/vapor/postgres-nio/issues/496), [PR497](https://github.com/vapor/postgres-nio/pull/497), and [PR498](https://github.com/vapor/postgres-nio/pull/498). This small async-query fix includes a direct acceptance/result lifetime regression. No duplicate issue has been opened. An upstream contribution should explain that history and retain a small diff; this document does not claim an upstream PR for this new patch already exists.

## Verification

- Fresh host Swift 6.4.2 test build succeeded. Both embedded regressions passed in approximately 0.003 seconds each: query after complete close fails, and accepted write finishes before its independent query result.
- With the same two-test filter and the original async query source, an external watchdog observed the hang and terminated only its owned process group. Restoring the fix makes the tests finish. Structured task cancellation alone cannot clean up the original unresolved driver promise.
- All embedded PostgresNIO tests (145) and connection-pool tests (108) passed.
- Full upstream suite against a separately created PostgreSQL 18 scratch database passed: 123 XCTest definitions, 114 executed successfully and 9 upstream-default skips (performance, Unix socket and long-running cases), plus the 253 embedded tests. The dedicated database had an ownership marker verified before removal. Existing Sauna databases were not used for upstream fixtures.
- An additional real TCP connection regression was then added and passed separately in 0.026 seconds, verifying that a query after client close promptly throws `clientClosedConnection`.
- Local DB credentials were read from an already authorized private file using a temporary test-only file adapter, which was restored before commit. Credential values were not passed in argv/environment or displayed. No new credentials were generated.

Independent root Review and Security Review examined the separated promise lifetimes and regression cases. Remote checks and GitHub Codex latest-commit results must be evaluated separately; quota failure is not a clean review.

## Fork CI and removal

The inherited `test.yml` (coverage secrets, downstream projects and broader runners) and `api-docs.yml` (deployment and write permissions) have repository gates: their jobs run only in `vapor/postgres-nio`, preserving upstream behavior while preventing their execution in this fork. GitHub initially does not register these inherited workflows, so disabling them by API returns 404 before they receive an event; settings disable will be attempted once they are registered. These jobs are not executed here and are not claimed as successful or as satisfaction of required checks. Apart from these repository gates, their source is preserved. The dedicated read-only, fully pinned workflow only builds and executes embedded tests; it does not run PostgreSQL integration, deploy documentation, submit dependencies or use secrets. Its dependency lock is committed for reproducibility.

Use a fixed consumer revision. Remove the fork after an upstream release contains an adequate fix and consumer deadline, transaction rollback, closed-channel and pool recovery regressions pass. The library fix does not itself implement Sauna authentication, deadlines or session security.
