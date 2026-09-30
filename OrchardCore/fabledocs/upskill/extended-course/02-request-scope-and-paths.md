# 02: A request crosses two middleware boundaries

## Learn the handoff, not only the class names

The tenant table from the previous chapter returns settings. A functioning request still needs the appropriate tenant services and tenant pipeline. Orchard Core separates those concerns between [ModularTenantContainerMiddleware](../../../src/OrchardCore/OrchardCore/Modules/ModularTenantContainerMiddleware.cs) and [ModularTenantRouterMiddleware](../../../src/OrchardCore/OrchardCore/Modules/ModularTenantRouterMiddleware.cs). Read both files before drawing the sequence. Their short size makes them good training material: nearly every statement changes what later code may safely assume.

The container middleware first awaits shell-host initialization. It then asks the running table to match the request. If settings are found, it checks whether the tenant is initializing. That branch returns a 503 response, appends Retry-After with the value 10, writes a specific initialization message, and returns. The later shell-scope setup is skipped in this branch. Treating all 503 responses as database outages would therefore misclassify an explicitly represented lifecycle state.

If no settings are found, the body guarded by the non-null check is skipped. This method does not invoke its next delegate in that case. It also does not explicitly assign a 404 in the shown null path. A source-based explanation should say precisely that. The final response observed by a client can depend on the surrounding pipeline and host defaults; do not invent a status assignment that is absent from the method. This distinction is a recurring theme in middleware debugging: a component's local control flow and the whole application's final response are related but not identical evidence.

The normal path makes RequestServices aware of the current ShellScope, obtains a shell scope from the host, and installs a ShellContextFeature. That feature holds the shell context plus OriginalPath and OriginalPathBase. The middleware then enters the scope's UsingAsync operation and invokes its next delegate. The values captured in the feature precede the tenant router's path adjustment. Their names are a useful clue, but the assignment order is the evidence.

## A worked request ledger

Use a fictional request whose incoming PathBase is /gateway and whose Path is /academy/articles/42. Assume selection has already chosen a tenant with RequestUrlPrefix academy and RequestPathBase /academy. These assumptions deliberately avoid mixing table matching with the later rewrite. At container entry, record /gateway and /academy/articles/42. At feature creation, record the same values in OriginalPathBase and OriginalPath. After entering the tenant router, record the rewritten values separately.

The router appends the tenant prefix to PathBase because a server or earlier middleware may already have set a base path. It then calls StartsWithSegments with ordinal, case-insensitive comparison and assigns the remaining path to Request.Path. Under our matching assumptions, the result is PathBase /gateway/academy and Path /articles/42. An application helper that builds a tenant-relative link can use the accumulated base while endpoint routing sees the remaining application path.

This is not equivalent to replacing PathBase with /academy. Replacing it would discard the upstream /gateway base and produce links that may work locally but fail behind the gateway. It is also not equivalent to leaving the tenant prefix in Path while adding it to PathBase, which would represent the same segment twice. The trace should show both fields at every boundary, because a single string called "URL" hides the transformation that matters.

Now repeat with incoming Path /ACADEMY/articles/42. The removal call uses case-insensitive segment matching. That does not make every later content route case-insensitive, nor does it define canonical URL casing for the application. It establishes the comparison used at this particular rewrite boundary. Good notes preserve that limited claim rather than applying a global label such as "Orchard ignores URL case."

A prefixless tenant follows a different branch: the path adjustment is skipped. The router still needs the shell feature and still invokes the tenant pipeline. This is a useful control case for tests because it distinguishes path behavior from pipeline behavior. If both a prefixed and a prefixless tenant fail identically, the prefix rewrite may not be the most promising first hypothesis.

## Pipeline availability and lazy construction

After path handling, the router asks HasPipeline. If the pipeline already exists, it directly invokes it. Otherwise it calls a local asynchronous helper, which checks HasPipeline again, builds the pipeline if still needed, and invokes it. The second check is visible in this file. The details of mutual exclusion or lifecycle coordination inside BuildPipelineAsync are outside this file and should be inspected before making concurrency guarantees.

This matters because a common review shortcut turns a double check into a proof of thread safety. The router's two checks show that it avoids an unnecessary build when a pipeline becomes available between observations. They do not, in isolation, prove that two simultaneous requests cannot both call BuildPipelineAsync. That stronger claim depends on the implementation called, its locking or task-sharing behavior, and the shell lifecycle. In your evidence ledger, distinguish the local observation from the next question it creates.

The constructor receives a RequestDelegate but discards it. The router is not a conventional pass-through middleware that inevitably invokes a global next delegate afterward. Its work is to forward to the tenant-specific pipeline. If you sketch a sequence diagram with a return to a global next controller after tenant execution, verify that such a step exists in the actual hosting composition. Do not add it because other ASP.NET middleware examples use that structure.

An incident involving the first request after a tenant change should therefore include pipeline availability and construction in the timeline. A slow first request and a fast second request could be consistent with lazy construction, but the observation alone does not identify which construction phase consumed time. Measure the relevant boundary rather than naming "cold start" as if it were a complete diagnosis. The later course will separate shell creation, feature composition, database access, and application work.

## Exceptions can be represented as features

Inside the container middleware's UsingAsync callback, the next delegate is awaited. After it returns, the code obtains IExceptionHandlerFeature. If that feature contains an error, it calls scope.HandleExceptionAsync with the error. This path is especially useful to study because an exception may have been handled by downstream exception middleware rather than escaping as a thrown exception at this location.

Imagine a downstream component catches an exception, writes an error response, and records an exception-handler feature. From the perspective of the awaited delegate, control can return normally. The subsequent feature inspection communicates the recorded error to the shell scope. A learner who watches only thrown exceptions would miss this route. A learner who assumes every error status necessarily creates such a feature would overstate it in the opposite direction.

Draw two distinct cases. In case one, downstream code returns a 400 response deliberately and no exception-handler feature with an error exists. The shown post-delegate block does not call HandleExceptionAsync based on status alone. In case two, the delegate returns after exception handling and the feature contains an error. The call happens even though the callback reached the statement after await. These are different inputs to scope behavior and may affect later commit or completion decisions elsewhere.

This chapter does not yet claim how ShellScope rolls back, commits, invokes callbacks, or schedules deferred work. Those are questions for the implementation of the scope itself. Keeping this boundary honest improves the next investigation: you know how an error is passed in, and you can now inspect what the receiver does with it. Architectural confidence grows from connected concrete steps, not from assuming a method named HandleExceptionAsync must implement every desired recovery property.

## Debugging laboratory: the wrong base path

An editor reports that links generated under a gateway point to /academy rather than /gateway/academy. Your initial evidence consists only of the broken output link and a screenshot of the tenant prefix. That is insufficient to accuse the tenant router. You need the request's PathBase immediately before the router, the selected tenant's RequestPathBase, and the values after the rewrite. If the incoming base is already empty, the router cannot preserve a gateway prefix it never received.

Build a three-row trace: upstream composition, container feature capture, tenant router adjustment. Put the observed or assumed values in separate columns. Label any value inferred from configuration rather than captured from a request. This prevents a common diagnostic error in which a deployment setting is treated as proof of the runtime field value. The configuration may be correct while a proxy header, middleware order, or hosting integration differs from the developer's mental model.

A proposed fix should target the boundary whose evidence is wrong. If the input PathBase is correct and the output loses it, inspect the router or later rewriting. If the input is wrong, investigate earlier components. If both request fields are correct but the rendered link is wrong, trace the link generator and view. These hypotheses share a symptom but imply different tests. A focused regression test reproduces the failing transition instead of merely asserting that a page loads.

## Debugging laboratory: an initialization response

A monitoring dashboard groups all 503 responses under retrieval or database failure. An Orchard request returns Retry-After 10 and the tenant-initialization message from this middleware. The code gives you a concrete alternative explanation. Record the shell's initialization state and the fact that the normal scope and next-delegate path were bypassed. Do not respond by retrying a controller operation that this request never reached.

Retry-After is a response hint here, not a proof that initialization will finish in exactly ten seconds. The constant does not establish a deadline, recovery guarantee, or upper bound on startup work. A client can use retry policy, but a reliable operations design still needs a way to observe prolonged initialization and distinguish it from a transient expected transition. Those observability additions would be proposed features unless separately found in current source.

The learning exercise is to write an incident note that is both useful and modest: identify the matching response branch, state which downstream work did not occur, record what remains unknown, and propose the next measurement. Avoid broad claims such as "the database is healthy" merely because this particular branch returned early. An earlier initialization failure might still involve the database; the response alone does not settle that cause.

## Independent exercises

### OC-02A: two coordinate systems

Trace a request with PathBase /edge, Path /school/lesson/7, and a selected tenant prefix school. Record OriginalPathBase, OriginalPath, final PathBase, and final Path. Repeat for a prefixless tenant with the same incoming fields. Explain which source statement supports preservation of an upstream base. State one end-to-end behavior that these field values alone do not prove.

### OC-02B: classify the early exits

Make a decision table for no matched settings, matched initializing settings, and matched non-initializing settings. Include whether the next delegate runs, whether this method sets status 503, whether Retry-After is appended, and whether ShellContextFeature is installed on this path. For the no-match case, phrase the status conclusion carefully. Then add a normal-path response with status 400 but no exception-handler error and predict the HandleExceptionAsync call count attributable to the shown post-delegate check.

### OC-02C: review a concurrency claim

A reviewer writes, "The router checks HasPipeline twice, so the file proves only one request can build a pipeline and every failed request rolls back automatically." Split this into claims. Identify the visible supporting evidence, the missing implementation you would inspect next, and a concurrency experiment you would design in an isolated test environment. Do not execute application setup as part of this chapter. Your deliverable is a test plan with an explicit observable outcome and a limitation.

## Hints and completion standard

For OC-02A, the feature is populated before the router adjusts the request. For OC-02B, distinguish a status code from an exception feature. For OC-02C, a repeated read is not a synchronization primitive by itself. Compare your work with [review guide 01](reviews/01-routing-and-request-boundaries.md) after writing predictions.

You have completed the opening request route when you can connect a selected settings object to a tenant scope, explain the rewritten path fields, and name the evidence boundary for failure propagation. The next step is to study the scope lifecycle itself. Carry forward the distinction between method names, local branches, and end-to-end guarantees; it will become even more important when persistence and deferred actions enter the trace.
