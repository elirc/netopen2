# 01: Selecting a tenant is an ordered decision

## The question before the controller

A request reaches the host with a host name and path. Before discussing a content controller, a permission check, or a query, Orchard Core must decide which tenant owns that request. This decision is easy to underestimate because the inputs resemble familiar web routing inputs. Tenant selection has its own algorithm, and that algorithm determines which tenant container and pipeline become relevant. A correct explanation therefore starts earlier than a controller route attribute.

Read [RunningShellTable](../../../src/OrchardCore/OrchardCore/Shell/RunningShellTable.cs), especially Add, Match, TryMatchInternal, TryMatchStarMapping, and GetHostAndPrefix. Then inspect [RunningShellTableTests](../../../test/OrchardCore.Tests/Shell/RunningShellTableTests.cs). These are the anchors for this chapter. Do not treat the exercises below as a substitute for the repository's tests. They are deliberately small traces that make the decision order visible.

The table maps a combined host and prefix key to shell settings. The dictionary uses ordinal, case-insensitive comparison. That tells you about key equality; it does not tell you that every component of a URL is normalized in every possible way. In particular, port handling has explicit branches. The host with its port and the host without its port are both considered. IPv6 support is one reason the code asks HostString for Host rather than casually splitting a string on a colon.

Before reading further, write a prediction: if a host-only registration and a prefix-only registration both match, which wins? Many developers answer that the more specific path should win. That answer imports an intuitive routing policy rather than reading this table's policy. The source tries host-only matches before prefix-only matches. Specificity must be defined by the implementation, not by the label that sounds most precise to a reviewer.

## Build the lookup keys first

GetAllHostsAndPrefix creates one or more lookup keys for a tenant. A tenant without a meaningful host restriction contributes a key beginning with a slash and its RequestUrlPrefix. A tenant with configured hosts contributes host-plus-slash-plus-prefix keys. Multiple hosts are an expansion of one tenant's registration, not multiple tenants. When Add sees the default shell, it also remembers that shell separately for fallback decisions.

GetHostAndPrefix is the counterpart used during lookup. Given a request path, it retains the first path segment rather than trying every nested segment as a tenant prefix. For example, the path /academy/articles/42 contributes /academy for a prefix lookup. This is distinct from application endpoint routing, where several remaining segments may carry meaningful route parameters. A learner who merges those stages will incorrectly expect a shell registered under a nested content path to behave like a controller route.

Use an explicit table in your notes. Give each tenant a name, configured host string, prefix, and resulting keys. Only after constructing that table should you run a request through Match. This two-phase method catches registration mistakes before they are hidden inside request reasoning. It also makes duplicate registrations visible. SetItems replaces values associated with existing keys; a dictionary is not a list of competing candidates that are all evaluated by a later score function.

A useful debugging distinction follows. A tenant may have perfectly valid settings in storage and still fail to appear under the expected lookup key. The question "does this tenant exist?" is different from "is the key I think I registered present in the running table?" An incident investigation should preserve that distinction rather than immediately blame authentication or a content route. You can often narrow the problem without starting the application by deriving the keys from the relevant settings.

## Work the ordered search

TryMatchInternal checks host with port plus prefix, host without port plus prefix when different, host with port alone, host without port alone when different, and finally prefix alone. The source's nested conditions avoid unnecessary duplicate lookups when the two host representations have the same length. Those branches are an optimization around the ordered policy, not an invitation to reorder it casually.

Consider three fictional registrations. Atlas uses host learn.example.test and prefix academy. Beacon uses the same host with no prefix. Cedar has no host restriction and prefix academy. Assume each registration produces the obvious distinct key and none is the default catch-all. A request to learn.example.test/academy/articles first finds Atlas. A request to learn.example.test/news can find Beacon through the host-only lookup. A request to other.example.test/academy/articles can find Cedar through the prefix-only lookup. These are different branches, even though two requests contain the same first path segment.

Now add a registration Delta for learn.example.test:8443 with no prefix. A request to that host and port with /academy/articles still has a possible host-without-port plus prefix match for Atlas before the host-with-port-only match for Delta. The presence of an exact port does not automatically outrank every other registration. The algorithm combines dimensions in a prescribed sequence. To explain the result accurately, write the exact attempted keys in order and stop at the first successful lookup.

This trace is valuable during code review because it exposes the cost of changing just one branch. Moving host-with-port-only ahead of host-without-port-plus-prefix would be a policy change for the example above. A seemingly harmless refactor could redirect requests to another tenant. Tests should encode the observable priority relation, not only verify that an isolated registration can match. A test where there is only one possible match cannot detect an accidental inversion between two competing registrations.

## Wildcards are a later search

Match tries the ordinary lookup before considering star mappings. The boolean flag avoids wildcard work when none has been added. TryMatchStarMapping first constructs a star form for the supplied host, then walks dots from left to right to look for progressively broader suffix mappings. The accompanying test names include longest matching host priority and rightmost host matching. Read their actual fixtures to see how this lookup policy is expressed.

Do not summarize this as "wildcards always beat the default" without recording the surrounding conditions. Ordinary exact and prefix matches happen first. Default fallback is later. A prefix-only match found during the first TryMatchInternal can already decide the request before a wildcard host is examined. If you want a different priority policy for a deployment, you need a proposed design and compatibility tests; you cannot describe your preferred policy as current behavior.

Also separate successful matching from secure trust in the request host. RunningShellTable consumes host and path values; it is not a complete reverse-proxy trust configuration. A deployment review must investigate how upstream forwarding middleware and allowed-host policy construct those values. This chapter does not establish that every internet-supplied header is trusted or rejected correctly. Its narrower claim is that, given the values passed into Match and the registered table, you can predict which shell settings are returned.

That scope statement is practical. It allows a reviewer to accept a precise algorithm trace while requesting a separate deployment trace. Overclaiming end-to-end isolation from one lookup function makes the explanation weaker, even when every local branch is described correctly. Good architectural reasoning composes bounded proofs instead of treating one successful lookup as evidence for the entire system.

## Fallback is conditional

After ordinary and wildcard matching, Match may return the default tenant if fallback is enabled and DefaultIsCatchAll is true. The default must exist and have empty host and prefix criteria. Merely being named or recognized as the default shell does not erase its configured criteria. If the default has a host or prefix restriction, it is not automatically the fallback for all unmatched requests.

If that default catch-all condition does not resolve the request, Match can search for another catch-all by looking up empty host and root path, again only when fallback is enabled. Finally it returns null. These are meaningful distinctions for diagnostics. "Default tenant exists" is insufficient evidence that an arbitrary unmatched request should resolve. "Fallback disabled" is insufficient evidence that all default matches disappear, because a default tenant can still match its explicit criteria in the ordinary lookup.

For a worked counterexample, imagine the default tenant is restricted to admin.example.test and there is no other catch-all. An unmatched request for public.example.test does not become a default request simply because that tenant is special. Conversely, a request for admin.example.test can match the registered host even if fallbackToDefault is false. The parameter controls fallback, not ordinary participation in the table. This is exactly the kind of distinction that a vague test name can obscure.

Removal has another detail worth reading. Remove locates keys whose stored settings have the same Name as the supplied settings and removes those keys. For the default shell it clears the remembered default reference. The wildcard flag is not recomputed down to false in the shown removal path. That can leave an extra wildcard search attempt after wildcard registrations are gone, but it does not by itself prove a stale wildcard tenant is returned. Performance state and lookup contents are separate observations.

## Debugging laboratory

An operator reports that /academy is reaching the generic host tenant rather than the expected prefix tenant. The first response should not be "add a longer route." Construct a candidate evidence packet containing the exact host with port, the first path segment, each relevant tenant's configured host and prefix, whether the registrations are currently present, and the ordered lookup keys. Redact private domain names if sharing the packet outside the team, while preserving equality relationships needed to reproduce the ordering.

Suppose the expected prefix tenant has no host restriction and the generic tenant has an exact host registration. The source explains the observed winner: host-only precedes prefix-only. If both tenants were intended to coexist under that host, one design option is a host-plus-prefix registration for the academy tenant. That is a configuration proposal to evaluate, not a command to change a running deployment as part of this exercise. Verify its effects on other hosts and ports before approving it.

The second incident concerns a missing default response. The operator shows that a tenant named Default exists. Ask for its criteria and the fallback flag instead of accepting the name as sufficient. A default with a prefix is not the catch-all described by DefaultIsCatchAll. Your report should cite the branch and the distinguishing input, making it possible for another developer to reproduce the reasoning without relying on your confidence.

## Independent exercises

### OC-01A: competing dimensions

Register fictional tenants Amber at portal.example.test plus prefix docs, Blue at portal.example.test:9443 with no prefix, and Coral with no host plus prefix docs. Trace portal.example.test:9443/docs/start. List every attempted key until a match. Then change only the path to /news. Explain why the winning registration can change without changing the host or port. Do not consult the review guide until your ordered list is complete.

### OC-01B: fallback and removal

The default tenant has host internal.example.test and an empty prefix. A second tenant is a hostless, prefixless catch-all. Predict an unmatched request with fallback enabled and disabled. Then remove the second tenant and repeat. Separately predict a request matching internal.example.test with fallback disabled. State what a test must assert beyond a non-null result to detect selection of the wrong tenant.

### OC-01C: reviewer challenge

A pull request description says, "Wildcard hosts and path prefixes are always preferred over host-only tenants, and removing the last wildcard disables wildcard processing." Evaluate each clause against the source. Produce a corrected two-paragraph description and propose two competing-registration test fixtures. A good answer names the unsupported statement, supplies a counterexample, and avoids claiming a security vulnerability from a priority difference alone.

## Hints and completion standard

For OC-01A, port-aware exactness is only one dimension in a sequence. For OC-01B, distinguish explicit matching from catch-all fallback. For OC-01C, read where the wildcard flag is assigned as well as where it is read. The separate [review guide](reviews/01-routing-and-request-boundaries.md) contains the outcomes and grading guidance.

Complete this chapter when you can explain a winner with attempted keys, identify an unresolved request without inventing an HTTP status, and name the next middleware boundary that consumes the selected settings. A shell selection trace is not yet a controller trace. The next chapter follows that transition and shows why request services, original paths, rewritten paths, and exception handling belong in different columns of the same investigation.
