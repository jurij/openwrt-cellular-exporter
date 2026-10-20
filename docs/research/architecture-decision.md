# Architecture decision and specification handoff

Research resolution for [Select architecture and define remaining release gates](https://github.com/jurij/openwrt-cellular-exporter/issues/5).
Part of [OpenWrt cellular exporter architecture](https://github.com/jurij/openwrt-cellular-exporter/issues/1).

## Verdict

Proceed to specification, not production implementation or installation.
Build an installable OpenWrt cellular collector for the existing Lua node exporter, with an independently replaceable source adapter.
Use the reference router's existing native status APIs rather than a second modem reader.
No custom firmware, settings page, additional history store, or bridge-owned per-scrape JSON file is required.

The decisive new evidence is the successful native `get_info`, `get_status`, and `get_cells_info` qualification.
It closes the previously assumed need for a custom AT-cache parser to obtain separate LTE/NR serving observations.
It does not prove original measurement freshness or every modem capability.

## Authoritative scope

- Product: reusable OpenWrt package, not a GL-X3000-only application.
- Physical acceptance target: the one available GL-X3000, GL.iNet 4.8.3, RM520N-GL. No additional physical router is required.
- Functional baseline: relevant cellular/modem values actually available through the reference UI/status surfaces, including LTE and NR observations.
- Do not silently reduce the baseline to generic signal bars.
- Do not inflate it to every capability that an arbitrary AT command could query.
- No own settings page. Technical configuration may use the platform's normal configuration mechanisms.
- No firmware replacement, new modem manager, modem resets, active scans, reporting-mode changes or automatic direct-AT fallback.
- Prometheus and Grafana remain the history and presentation services.
- Subscriber/device identifiers, credentials, SMS content and raw responses are excluded.
- The package can be public while only the reference combination is hardware-validated. Distinguish product scope, source support, software compatibility and hardware evidence.

## Selected architecture

### Components

1. `prometheus-node-exporter-lua-cellular`: one collector entry point, normalized field contract, validation, bounded labels and Prometheus output.
2. `prometheus-node-exporter-lua-cellular-gl`: the reference source adapter, requiring the existing GL modem service. It must not install or replace that service.
3. The existing `prometheus-node-exporter-lua` package supplies the HTTP endpoint and service management.

These are proposed package names for the specification, not built artifacts.
Keep adapter modules outside the exporter's automatic collector directory.
Do not introduce a plugin registry, independent cellular webserver or generic modem-control framework.

A stock-OpenWrt adapter is a separate source contract, not a conditional import of GL firmware libraries.
An already controlling ModemManager is the best documented structured candidate.
Do not install ModemManager on the reference GL router to manufacture compatibility.
A second adapter must not be a stub or advertised as tested merely because the common package builds.

### Reference request path

`Prometheus -> existing /metrics endpoint -> cellular collector -> qualified native status reads -> validated metrics`

Use these exact existing-owner operations:

- `get_info({})` for supported discovery/metadata and a valid modem selector.
- `get_status({})` for current native status, generic signal, SIM/operator state and vendor-reported usage.
- `get_cells_info({bus})` for technology-labelled serving observations, bands and bandwidth.

The live research called each once successfully. Approximate durations were below one second per operation; this is not a recurring-load test.
Use the existing local RPC transport on the router, with fixed method and destination allowlists.
Do not turn the exporter into an arbitrary RPC proxy or expose native RPC credentials through its endpoint.
Local trusted/root access used by research is not proof of a reduced-privilege design.

Discover inside bounded collection, not at module import. Avoid stale module-global discovery caches: HTTP handlers fork and do not share updates.
Do not assume that three sequential native calls form one atomic observation.
Read response bodies into bounded memory, normalize them, emit metrics and discard them.
Do not write raw responses, fixture captures or transient JSON documents to disk.

Kernel interface counters may reuse the existing netdev collector, with a qualified cellular-interface mapping.
If the cellular collector exposes them directly, emit each physical device once even when several logical interfaces share it.
Netifd interface uptime must remain named interface uptime, not bearer/session duration.

### What is not on the initial path

- No 3ginfo-lite or modemdata producer installation: their examined paths include additional device access and reporting-mode changes.
- No parsing of the existing AT-cache file to duplicate fields already returned by native cells/status.
- No mandatory RAM-history ubus read: native status already supplies generic signal. History is not needed to start a new Prometheus timeline.
- No background sampler unless later measurements prove the selected request path cannot meet its budget or safe concurrency contract.
- No source-specific privileges for direct serial/QMI access.

The optional RAM signal-history reader remains available for a later justified requirement.
Its collector timestamp cannot be transferred to native LTE/NR cell measurements.

## Measurement contract

### Reference coverage to specify

The detailed, live-qualified schema is in the native-status note. The specification must map every relevant field to an exported metric, an explicit privacy exclusion, a configuration/capability annotation, or a documented unqualified interpretation.

Required groups include:

- Separate LTE and NR quality fields where present; preserve LTE duplex and NR NSA/SA distinctions.
- Serving-band and supported bandwidth observations, with proven unit conversion.
- Generic signal and vendor quality levels, kept separate from technology-labelled physical quantities.
- Selected SIM slot, native SIM/switching states, operator and network codes.
- Native connection state, correctly interpreting zero as connected for the qualified GL schema.
- Vendor-reported data usage, with reset/edit semantics rather than a false permanent counter guarantee.
- Allowlisted modem model/firmware/capabilities.
- Cellular interface traffic/state and explicitly named interface uptime where included.
- Reader availability, successful collection, field availability and known/unknown source age.

Native cell values are already normalized for the vendor UI. Do not apply raw QENG LTE SINR conversion again.
The observed LTE mode was `LTE FDD`, not the literal string `LTE`.
Preserve leading zeros in MCC/MNC. Missing NR RSSI and NR uplink bandwidth remain absent, not zero.

Bandwidth strings use a numeric value followed by `M`. The implementation must establish the source's MHz meaning before normalizing them to Hz or MHz.
`id`, `tx_channel` and `RF` must not receive invented PCI/frequency/duplex meanings.
Location-sensitive cell identifiers are not default public labels. A later opt-in requires a clear semantic and privacy contract.

The reference UI does not establish a routine modem-temperature or bearer-session-history field.
Treat these as optional capabilities, not mandatory fields fabricated from SoC temperature or interface uptime.
Two serving rows do not establish a complete carrier-aggregation inventory or two simultaneously transmitting carriers.
An unobserved or unsupported field remains distinguishable from a collector error.

### Freshness: an explicit limit, not a blocker manufactured by the exporter

Native cells/status do not return original measurement timestamps.
Expose successful API retrieval and request completion independently from original measurement age.
The latter is unknown unless the source supplies evidence.

Publish the native observations as untimestamped vendor observations with documented freshness uncertainty; do not label them guaranteed fresh radio measurements.
Do not create a `last_measurement_timestamp` from the time a response was received.
A fresh scrape proves successful retrieval, not a newly measured modem value.
On reader failure, omit measurement series rather than retain them as current. Prometheus can preserve their historical samples without replaying old data.
Do not infer an active NR bearer from an untimestamped NR row or declare a failure solely because an idle NSA modem omits it.

This contract preserves useful native data without pretending an unavailable timestamp exists.
Further firmware evidence can improve freshness semantics; it is not a reason to implement a second modem controller.

## Runtime budget and concurrency decision

Use synchronous, request-scoped reads first. The source is the existing modem owner; the collector itself opens no modem port.
No new periodic sampler or inter-process cache is justified by the current evidence.

Reconcile the two earlier provisional scrape proposals as follows:

- Initial design target: 30-second Prometheus interval and 10-second scrape timeout, set explicitly for this job only.
- Target adapter whole-operation deadline: at most 5 seconds across discovery, status, cells, bounded response parsing and cleanup, with no automatic retries.
- These are proposed test budgets, not observed production settings or hard guarantees already implemented.
- A native HTTP request deadline must include connect, send, receive, total response size and the whole operation; per-read socket timeouts alone do not prove it.
- Keep the existing exporter's outer uhttpd script timeout as a separate fallback guard. The observed default is 60 seconds; it is not the proposed 5-second adapter deadline.
- The reference native-HTTP path avoids the unbounded ubus object-lookup problem. Do not add optional ubus calls until they satisfy the same deadline contract.
- Do not patch or replace the base exporter init merely to force a new arbitrary timeout. Qualify the bounded in-process client first.
- If the client cannot meet the whole-operation contract with compatible libraries, record the concrete failure and evaluate a bounded one-shot helper. Any base-service packaging change must be explicit, not a hidden overwrite from a collector package.

A timeout in our reader does not cancel work already accepted inside the vendor service.
No test may kill or stall the live vendor service to simulate failure.
Test fake slow/disconnected/invalid sources offline and verify the next request recovers.

The existing uhttpd request model can overlap scrapes. A Lua-table lock is not cross-process coordination.
Start with bounded reader calls through the existing owner, not serial-device locks.
Validate overlapping requests and resource usage before deployment claims. If overlap is unsafe in practice, require explicit bounded serialization or a changed runtime design; do not conceal the issue behind cached success.

## Packaging and integration

Build packages with an SDK, never a firmware image.
The available vendor target needs an IPK. Modern OpenWrt APK is a separate build artifact and test lane, not a renamed IPK.
The pure Lua package can be architecture-independent while its native dependencies remain release-specific.

For the initial reference pilot, resolve dependencies only from compatible vendor feeds.
The preflight found `luasocket` and `uhttpd-mod-lua` missing; their installation is part of the later reviewed dependency delta.
Do not replace modem management or shared webserver packages implicitly.

Keep loopback-only binding until a separately reviewed exposure plan is approved.
A later scrape requires the selected private destination plus a router-local allowlist for the actual monitoring source and exposed address families.
A private bind or downstream firewall alone does not establish source isolation.
Keep the Prometheus listener and all existing jobs unchanged.
Do not publish the exporter through a public reverse proxy or WAN forwarding.

## License and release position

Recommend Apache-2.0 for independently written collector/adapter code, matching the inspected base exporter license.
The owner has not yet selected a code license; do not create a LICENSE file or claim the project is Apache-licensed solely from this recommendation.
Do not copy GPL producer parsing code or redistribute proprietary vendor binaries.
Record actual dependencies and notices before code publication.

The first software release is a prerelease only after a real package, basic live collection, ingestion, bounded failure behavior and scoped rollback are verified.
A regular release requires the agreed field mapping and acceptance checks.
No extra physical router, forced WAN interruption or invented percentage SLA is required.
Other modem and firmware combinations stay explicitly untested.

## Single acceptance checklist for the specification

This is one future checklist, not a report that these tests passed:

- [ ] Every relevant reference UI/status field has an explicit mapping or documented exclusion; no reduced signal-only substitute.
- [ ] LTE/NR labels, units, bandwidth conversion and invalid/sentinel handling are verified.
- [ ] Reader health and unknown measurement freshness are explicit; failures never replay old values as new.
- [ ] Whole-operation deadline and bounded response size work across slow connect/send/read, malformed input and client cancellation.
- [ ] Overlapping scrapes leave no stuck workers, queues or resource accumulation; no modem-controller changes occur.
- [ ] Prometheus exposition passes real promtool checks, with one TYPE per family, correct label escaping and no duplicate label sets.
- [ ] Compatible IPK installation/dependencies/upgrade/removal are verified; any APK support is separately evidenced.
- [ ] Exact listeners, effective UID and allowed/denied source access are verified before remote scraping.
- [ ] Actual Prometheus samples match near-time native values and the explicit job settings.
- [ ] Existing Grafana can display correct units, LTE/NR distinctions and unknown/gap states without another database.
- [ ] Scope-owned package/configuration changes can be rolled back without resetting the modem, rebooting the router or touching WAN failover.
- [ ] Published artifacts exclude private infrastructure details and identifiers, include checksums/documentation and use an approved code license.

The earlier stdin-based promtool attempt failed; it does not count as a passed configuration check.
Normal fixture tests, package builds and live hardware checks are separate evidence classes.

## Handoff

The architectural research is sufficient to write the specification and its blocking implementation tickets.
Do not use this result as permission to start production deployment.
The next specification should preserve the field inventory, source-age limits, deadline contract and explicit installation boundary.
A genuinely failed qualification may require a small architecture revision; it must not silently expand modem access or reduce the requested data scope.

## Primary research records

- [Telemetry source inventory and initial gaps](https://github.com/jurij/openwrt-cellular-exporter/blob/4f32042272e35cbac7096a359eae8406174ece2d/docs/research/telemetry-sources.md).
- [Native status follow-up and updated safety/coverage conclusions](https://github.com/jurij/openwrt-cellular-exporter/blob/1bd9803a7569edcff8346e48f1f28e70002fd8ac/docs/research/native-status-qualification.md). This supersedes the initial blanket concern about compiled read handlers and the conflation of optional hardware capabilities with UI coverage.
- [Package/runtime source review](https://github.com/jurij/openwrt-cellular-exporter/blob/d1eda3fad76a4156ecf59732d5d06637a84a3ba7/docs/research/package-runtime.md). The native HTTP path and no mandatory ubus/history call refine its preliminary transport/deadline recommendation; proposed numbers are not runtime measurements.
- [Integration preflight and acceptance evidence](https://github.com/jurij/openwrt-cellular-exporter/blob/8b87a8676560533c9264d29acca6aec4dcccbf34/docs/research/integration-acceptance.md). The provisional 15s/5s example is superseded by this design's 30s/10s test target, subject to measured qualification.
