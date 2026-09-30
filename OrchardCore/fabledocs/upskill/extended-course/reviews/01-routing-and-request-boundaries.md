# Review 01: routing and request boundaries

Attempt [chapter 01](../01-tenant-selection.md) and [chapter 02](../02-request-scope-and-paths.md) before using these answers. This review grades causal explanations. A correct tenant name with no lookup trace earns less credit than a complete trace, because the latter can be transferred to a new configuration.

## OC-01A: competing dimensions

The first request attempts portal.example.test:9443/docs, then portal.example.test/docs. Amber wins at the second key. Blue's exact-port host-only registration is not reached. After changing the path to /news, the first two candidate keys are portal.example.test:9443/news and portal.example.test/news; neither is registered in the fixture. The host-with-port-only key portal.example.test:9443/ then selects Blue. Coral's prefix-only key is later and is not reached in either successful trace.

The important explanation is not that Amber is "more specific." The actual ordering chooses host-without-port plus prefix before host-with-port alone. A generalized specificity slogan could predict a different result. Award full credit when the answer names both successful keys, preserves the lookup order, and explains the changed path's effect. Deduct credit if the answer treats port matching as an unconditional top-level priority.

For transfer, add a tenant registered at portal.example.test:9443 with prefix docs. That registration would satisfy the first attempted key and win. This extension tests whether the learner understood the combination of dimensions rather than memorizing Amber as the expected tenant. It also supplies a useful regression fixture if a refactor flattens the nested conditionals into an ordered list of candidate keys.

## OC-01B: fallback and removal

With fallback enabled, the unmatched request can select the non-default catch-all through the later empty-host/root-path lookup. The default tenant is not a catch-all because it has a host restriction. With fallback disabled, the described unmatched request returns null. After removing the second tenant, the unmatched request returns null even with fallback enabled, since the restricted default does not qualify and the other catch-all key is gone.

A request matching internal.example.test can still select the default tenant through ordinary matching when fallback is disabled. Disabling fallback does not remove the default tenant from the dictionary or prohibit all matches to it. Tests should assert the returned tenant identity, preferably the relevant settings instance or distinct name as appropriate to the test style. A generic non-null assertion would miss accidental selection of the wrong tenant.

The removal explanation should not invent a full reconfiguration transaction. The source removes keys associated with the given name and clears the separate default reference when appropriate. Broader lifecycle coordination belongs to other components. A strong answer remains inside the algorithm shown while listing lifecycle questions that would matter for a live deployment.

## OC-01C: reviewer challenge

The proposed description is inaccurate. Ordinary TryMatchInternal executes before wildcard matching, and within that ordinary lookup host-only checks precede prefix-only matching. Thus a matching exact host-only tenant can decide a request before either a later prefix-only alternative or a wildcard search. Wildcards are not universally preferred. The corrected description should state the ordered phases and identify first-success behavior.

The removal claim is also unsupported. Add can set the wildcard flag, while the shown Remove method does not reset it after removing the last wildcard entry. The retained true flag can cause extra wildcard lookup work; it does not preserve removed dictionary entries. Avoid turning this observation into a claim of cross-tenant leakage without a failing selection trace.

A useful pair of tests pits exact host-only against prefix-only, then exact host-only against a wildcard registration. Each fixture must include both candidates simultaneously and assert the expected selected settings. A separate removal test can verify no removed wildcard tenant is returned. It should not expect private implementation flags to have a particular value unless the test deliberately targets that implementation detail.

## OC-02A: two coordinate systems

The captured feature values are OriginalPathBase /edge and OriginalPath /school/lesson/7. For the prefixed tenant, the router appends /school to produce PathBase /edge/school and removes the matching segment to produce Path /lesson/7. For the prefixless tenant, the adjustment branch is skipped, so the request fields remain /edge and /school/lesson/7. The feature still captures the incoming values on the normal container path.

The compound assignment to PathBase is direct evidence that an existing base is appended to rather than replaced. These values do not alone prove that every rendered link is correct, every proxy forwarded value is trusted appropriately, or every endpoint route resolves. Each of those claims needs additional code and runtime evidence. Full credit requires one concrete limitation, not a generic statement that more testing is good.

## OC-02B: classify the early exits

With no matched settings, this method does not invoke next, does not install the feature through its guarded normal path, and does not explicitly assign the initialization 503 or Retry-After header. Do not claim that the file explicitly sets a 404. With initializing settings, it assigns 503, appends Retry-After 10, writes the initialization message, and returns before normal scope setup and next invocation. With matched non-initializing settings, it performs the scope and feature setup and invokes next inside UsingAsync.

On the described normal-path 400 response with no exception-handler error, the shown post-delegate check makes zero HandleExceptionAsync calls. Status alone is not its condition. A recorded exception feature with a non-null error would cause a call after the delegate returns. This answer says nothing about other exception handling inside ShellScope, which has not yet been traced in these chapters.

Award full credit for a table that treats these as separate predicates. Deduct credit for equating every non-success status with an exception or for claiming the initialization branch proves the underlying reason initialization is taking time. The visible code establishes the response branch, not the whole causal history of the shell state.

## OC-02C: review a concurrency claim

The file shows an initial pipeline check and another check in an asynchronous helper before BuildPipelineAsync. That is evidence of avoiding an unnecessary call when the pipeline becomes available before the second check. It is not sufficient evidence of exclusive construction. Inspect BuildPipelineAsync and the shell context lifecycle before claiming a single construction under concurrent requests.

The automatic rollback claim needs ShellScope's implementation and persistence integration. The middleware calls HandleExceptionAsync when a recorded exception feature has an error; it does not visibly perform a database rollback. An isolated concurrency test plan could coordinate two requests at the construction boundary, count construction executions, and verify both reach the correct tenant pipeline. Such a test needs a controlled host or fake shell context with well-defined synchronization, not a timing-dependent sleep that happens to pass once.

A good plan also includes a failure case and a scope limitation. For example, one failed construction should not be reported as evidence for database rollback unless a real session transaction is part of the test. The observed outcome may only establish how the pipeline build task is shared or retried. Reviewers should reward that honest scope rather than demanding broad claims unsupported by the fixture.

## Review rubric

Use four dimensions: exact inputs, ordered transitions, observable output, and limitation. Each exercise can earn one point for each dimension. An answer that names a correct result but cannot explain the intervening keys or request fields should not pass the independent milestone. An answer that is locally correct but asserts global isolation or rollback without evidence needs revision before moving on.

The final oral check is simple: give the learner a new host, prefix, incoming base path, and lifecycle state. Ask them to locate the first decisive branch and state what work does not happen afterward. If they can do that without quoting a memorized slogan, they are ready to study shell scopes and persistence. Keep the completed trace as a baseline artifact for later incident exercises.
