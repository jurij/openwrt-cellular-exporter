# Package and exporter runtime qualification

Research for [Qualify package and exporter runtime architecture](https://github.com/jurij/openwrt-cellular-exporter/issues/3).
Part of [OpenWrt cellular exporter architecture](https://github.com/jurij/openwrt-cellular-exporter/issues/1).

Status: research recommendation, not implementation or deployment approval.
Evidence checked on 2026-10-20. No router access, installation, service restart, package build, or software publication occurred during this investigation.

## Recommendation

Use a synchronous, passive Lua collector inside `prometheus-node-exporter-lua`.
Read the existing modem manager's qualified status data. Do not query serial ports or install another modem manager.
Do not add a background sampler now: the known GL signal producer already maintains a RAM buffer.
Reading that buffer again in a new daemon would duplicate caching without establishing additional telemetry coverage.

Keep one exporter endpoint. Reuse its existing dedicated uhttpd instance, not the LuCI instance and not a new cellular HTTP server.
Install a small common collector package plus a GL adapter package.
Add another adapter package only when a different existing producer has a qualified contract and different dependencies.
Do not build a generic plugin framework, a producer fork, or a settings UI.

**Two runtime qualifications remain mandatory.** First, establish the installed uhttpd build and effective limits.
Second, give the synchronous request a verified outer deadline: the Lua ubus timeout does not bound object lookup.
A short uhttpd handler deadline is sufficient for an in-process reader with no descendants; this does not require a sampler.
The inspected exporter init script does not expose that deadline through UCI.
A narrowly scoped, separately approved exporter-service packaging change may therefore be necessary.
Do not hide that change inside the cellular collector package. [S1–S7]

This is a minimal architecture recommendation, not proof that passive native sources provide every required field.
Full LTE/NR, cells, temperature, modem status, and session coverage remains a source-qualification gate.
A signal-only implementation would not satisfy the project destination. [E1]

## Evidence classes and boundaries

- **Reproduced package evidence:** the public vendor archive checksum and three upstream file comparisons were rechecked in this investigation.
- **Source evidence:** behavior below comes from pinned upstream source, not from a running exporter on the reference router.
- **Inherited observation E1:** prior private, read-only reference-device inspection established the GL signal-buffer reader and existing Lua libraries.
- **Recommendations:** limits, package names, failure policies, and test requirements below are proposed contracts, not measured results.

The reference hardware is GL-X3000 with GL.iNet 4.8.3, an OpenWrt 21.02-SNAPSHOT base, and Quectel RM520N-GL.
E1 establishes that `modem.signal.get_signals` reads a RAM buffer; arguments `{ "time": 0 }` request its newest signal record.
It does not establish a public, stable GL API or complete LTE/NR coverage.
Generic signal fields do not independently identify the LTE anchor and NR component.
GL's public documentation confirms signal history and band-status functions, but not the backend safety contract. [S17]

Other GL UI calls, including `modem.get_info`, `modem.get_status`, and `modem.get_cells_info`, remain unqualified here.
Their binary backend must not be treated as passive merely because the UI calls it.
No private capture, device identifier, subscriber identifier, address, hostname, credential, or raw modem response is included in this note.
E1 is a sanitized inherited observation, not a publicly reproducible source citation.

## 1. Exact exporter baseline and collector ABI

The vendor feed provides `prometheus-node-exporter-lua_2024.06.16-2_all.ipk`.
Its SHA256 is:

```text
b9133f2ec9ff17ab28011051dcf817286cffe1c2968782d080ee9e6d36125763
```

The downloaded archive matched the feed checksum.
The following extracted files matched OpenWrt packages commit `c279efb760ca516db7cdaee32216344df47f2203` byte-for-byte:

- `/usr/bin/prometheus-node-exporter-lua`
- `/etc/init.d/prometheus-node-exporter-lua`
- `/etc/config/prometheus-node-exporter-lua`

Archive inspection happened in workstation memory without executing package scripts.
This proves the selected feed artifact's contents, not that it is installed or running. [S1, S2]

### Module contract

| Item | Verified contract |
|---|---|
| Discovery | `/usr/lib/lua/prometheus-collectors/*.lua` |
| Module result | `return { scrape = scrape }` |
| Entry point | `scrape()` without arguments |
| Output API | Global `metric(name, type, labels, value)` |
| Family emitter | `metric(name, type)` returns a function accepting `(labels, value)` |
| Other globals | `get_contents`, `space_split`; do not depend on undocumented output internals |
| HTTP route | `/metrics`; another path receives 404 |
| Selection | Query `collect[]=cellular`; no selection runs all loaded collectors |
| Unknown selection | Unknown collector names are ignored, not rejected |
| Loading | Modules load during exporter initialization, before HTTP requests |
| Errors | `pcall(collector.scrape)` catches scrape exceptions, not module-load failures |
| Result metrics | `node_scrape_collector_duration_seconds` and `node_scrape_collector_success` |

Collectors execute sequentially within one request. Iteration uses `pairs`, so ordering is not a stable contract.
HTTP 200 is written before collection. Success means no Lua exception, not valid source data or complete telemetry.
New modules and changed module code require exporter reinitialization; copying a file alone does not reload it. [S2]

Keep module loading cheap and side-effect free.
Create an ubus connection within each scrape and close it on every exit path.
Do not open a connection before fork and share the inherited socket across request children.
Do not perform producer discovery or native status calls during module initialization.
These recommendations follow from the startup-loading and forked-handler model. [S2, S4, S6]

### Dependencies

The vendor archive declares `libc, luasocket, lua, uhttpd, uhttpd-mod-lua, libubus-lua`.
The pure Lua collector adds no compiler, kernel module, web framework, database, LuCI application, or modem-management dependency. [S1]

E1 observed Lua 5.1, `libubus-lua`, and `lua-cjson` on the reference system.
Native `libubus-lua` returns Lua tables; its normal path does not need JSON encoding or decoding.
Declare `lua-cjson` only for an adapter that actually consumes JSON.
Declare directly used libraries explicitly rather than relying indefinitely on transitive exporter dependencies.
A UCI-reading implementation would additionally need the relevant binding; UCI is not required merely to return a collector module.

Current OpenWrt 25.12.6 still packages Lua 5.1.5.
Do not assume a package-manager transition implies a Lua ABI transition. [S11]

## 2. Exact timeout, concurrency, and cancellation behavior

### What the inspected versions actually do

| Layer | Source-verified behavior | Qualification limit |
|---|---|---|
| Exporter | No per-collector timeout or inter-request lock | `pcall` is not cancellation |
| Lua ubus connection | `ubus.connect(nil, timeout)` accepts integer seconds; default 30 | Timeout applies to `ubus_invoke`, not all earlier work |
| Lua ubus call | Resolves object ID, then invokes with `timeout * 1000` milliseconds | Object lookup uses `ubus_complete_request(..., 0)` |
| libubus lookup | A zero timeout does not establish an elapsed-time deadline | A stalled bus can block before invocation starts |
| Reference uhttpd | `-t` sets script timeout; source default 60 seconds | Actual GL binary and patches not established |
| Reference uhttpd | `-n` sets simultaneous script requests; source default 3 | Shared across script requests in that uhttpd instance |
| Reference uhttpd | Excess script requests are deferred | Queue wait is not part of the running child's script timer |
| Reference uhttpd | Network timeout default 30 seconds; HTTP keepalive default 20 | Distinct from collector and script deadlines |
| Exporter init | Supplies keepalive 70 seconds, unless configured otherwise | No `-t`, `-n`, or `-T` arguments |
| Timeout cleanup | uhttpd sends SIGKILL to its direct handler child | No guaranteed recursive termination of subprocess descendants |

The uhttpd reference is commit `15346de8d3ba422002496526ee24c62a3601ab8c`, selected by upstream OpenWrt 21.02.
It is not proof of the installed GL uhttpd version.
The ubus reference is `4fc532c8a55ba8217ad67d7fd47c5eb9a8aba044`, selected by upstream OpenWrt 21.02. [S3–S7]

**Important correction to a tempting implementation:** `ubus.connect(nil, 1)` is not a one-second whole-collector deadline.
Changing to `ubus -t 1 call ...` does not repair this: that CLI also resolves the object before invoking it. [S6, S7]

uhttpd forks the Lua request handler.
Two scrapes can therefore call the same collector concurrently, despite sequential collectors within each scrape.
A module-level Lua table is copied into children; it is not shared state or a persistent cross-scrape cache.
A lock in that table is equally ineffective. [S2, S4]

A timed-out RPC client does not cancel work already dispatched inside the modem manager.
Therefore, only qualified passive readers belong on the synchronous path.
An exporter-local serial lock would not coordinate with the vendor modem manager anyway. [S6, S7; design implication]

### Recommended bounded synchronous contract

These are proposed limits for qualification, not existing settings:

- One native status read for the proven GL signal-buffer method per collector invocation; no retry loop or history-window request.
- Native invocation timeout: **1 second**.
- Existing exporter uhttpd handler timeout: **5 seconds**, covering all collectors in that running handler.
- Simultaneous HTTP script limit: **3**, retained only after overlap and resource tests pass.
- Prometheus scrape timeout: **10 seconds**, with a proposed interval of **30 seconds**.
- Normalized output must have explicit schema, record-count, and string-length limits derived from the field inventory.
- No unbounded source-history read, recursive schema traversal, raw-response log, or raw-response label.

The 5-second outer deadline covers the stalled lookup that the 1-second invocation timeout cannot cover.
It also covers normalization inside the handler. It does not include waiting before a handler starts.
A saturated endpoint can still fail a Prometheus scrape while requests are deferred.
That is a visible failure, not permission to add workers or return stale measurements as current.

The current exporter UCI contract exposes only `listen_interface`, `listen_port`, and `http_keepalive` on the inspected vendor baseline.
Adding `script_timeout` or `max_requests` to that UCI file has no effect by itself.
If the deployed service has no supported override, the smallest hard-deadline change is an exporter-init enhancement passing `-t` and `-n`.
That is a package/service change, not a firmware build. It requires explicit review and package ownership coordination.
Do not overwrite another package's init script from a collector package or silently replace the vendor exporter. [S2]

**Decision gate:** the pure collector can be prototyped against the existing ABI, but strict deadline qualification is not complete today.
Do not claim the unmodified vendor package provides the proposed 5-second bound.
If changing exporter service packaging is rejected, a bounded one-shot reader is the next alternative, not automatically a periodic sampler.
Such a reader must cover connect, lookup, invocation, output size, process termination, reaping, and parent death.
A bare `io.popen` plus `ubus -t` does not meet that contract.
No helper implementation or additional helper dependency is selected in this note.

### Why no lock or sampler yet

Concurrent reads of a verified RAM snapshot do not require serial-device mutual exclusion.
They still require tests for bus load, response size, latency, and server-side side effects.
Do not set `-n 1` merely to simulate a cache; it serializes the exporter and can increase queue delay.

A periodic sampler becomes justified only if evidence establishes at least one of these conditions:

1. Qualified collection exceeds the acceptable scrape budget.
2. Repeated parallel reads measurably overload the existing producer.
3. Several qualified calls require a consistent snapshot unavailable from the producer.
4. Source access must be coordinated across scrapes and cannot safely remain request-scoped.

If needed, use one procd-managed sampler with one in-flight collection and one bounded latest snapshot in process memory.
Serve that snapshot to the existing collector over a local restricted IPC endpoint, not a second HTTP server.
Here, “shared RAM snapshot” means one owner with IPC access; it does not require an mmap/shared-memory framework.
A Lua table in the sampler is sufficient because that process owns updates; request children must fetch it explicitly.

The sampler must bound every collection attempt and IPC read, skip overlapping polls, and terminate/reap any owned worker on timeout.
It must retain collection time, source time when known, last-success time, and collection status separately.
A failed poll must not refresh the age of the last successful sample.
After a source-specific maximum age, omit stale measurement series and expose failure/staleness status.
Neither sampler restart nor clock adjustment may make old data appear new.
Choose interval and maximum age from the measured producer cadence, not an arbitrary universal TTL.
A sampler does not make an unsafe native method passive or establish missing LTE/NR semantics.

## 3. Prometheus formatting rules

The pinned vendor exporter inserts label values directly between quotes.
The current upstream exporter still does not escape them automatically. [S2, S8]

Use static, allowlisted metric and label names from the legacy-compatible Prometheus character set.
Do not create metric names from arbitrary source keys.
At the output boundary, escape raw label values exactly once:

| Input character | Exposition bytes |
|---|---|
| Backslash | `\\` |
| Double quote | `\"` |
| Line feed | `\n` |

Escape backslashes before inserting the escapes for quotes and line feeds.
Alternatively, use one character-wise transformation so inserted escapes cannot be escaped again.
Do not JSON-escape the whole string: JSON forms such as `\uXXXX` are not the label-escaping contract.
Require UTF-8 text and reject unsupported control characters according to the adapter's validation policy. [S9]

Call `metric(name, type)` once per metric family per scrape, then emit that family's samples through its returned function.
Calling the four-argument helper repeatedly for the same name repeats `# TYPE`, which the exposition specification forbids.
Group each family's samples together. Ensure every metric-name/label-set combination is unique. [S2, S9]

Validate the complete normalized result before emitting measurement families.
Do not emit a partial family and then discover that another record is malformed.
Represent missing measurements by absence, not zero or a string such as `-`.
Emit numeric values, not booleans or formatted display text. Zero remains a valid value.
For this product, reject non-finite source measurements even though Prometheus permits `NaN` and infinities.
Do not attach custom sample timestamps; expose source age or collection time as separate documented metrics if meaningful.

Use a bounded set of failure reasons. Never include a raw exception, response, modem identifier, or credential in a label.
The exporter writes scrape exception text to stderr, so adapter exceptions must also be sanitized.
Unexpected parsing failures must not silently return success.
Expected unsupported fields should be omitted with explicit availability semantics, not raised as arbitrary source errors.
Test both the collector's health metric and actual measurement presence; HTTP 200 is insufficient. [S2]

## 4. Package-only builds and support matrix

The upstream Makefile uses `PKGARCH:=all` and an empty `Build/Compile`.
The same pattern supports a source-Lua adapter without a firmware image.
Use an OpenWrt SDK and the package target, not ImageBuilder or a firmware-flashing workflow. [S1, S10]

A later implementation build should pin the SDK release or image digest, feed revisions, and SDK action revision.
It should select only the intended package targets, such as `make package/<package>/compile V=s`.
These are build instructions for later work, not commands executed in this investigation.

| Target family | Package format | Evidence and proposed support position |
|---|---|---|
| GL.iNet 4.8.3 / reference OpenWrt 21.02-SNAPSHOT | `.ipk` / opkg; Lua package architecture `all` | Primary hardware-validation target. Vendor exporter archive and ABI verified; installed runtime, dependency resolution, and collector not validated. |
| Stock OpenWrt 21.02, 22.03, 23.05 | `.ipk` / opkg | Legacy compatibility candidates, not a claim of maintained upstream security support. No additional physical hardware required. Test only combinations explicitly promised. |
| Stock OpenWrt 24.10 | `.ipk` / opkg by default | Useful modern opkg package/runtime test lane. Inspected branch exporter is `2025.07.15-1`; not the vendor baseline. |
| Stock OpenWrt 25.12 | `.apk` / apk by default | Current stable family. Latest release observed was 25.12.6. Inspected branch exporter is `2026.06.05-1`. Build a separate APK artifact. |
| Current upstream snapshot | `.apk` / apk by default | Moving compatibility lane, not a stable release promise. Pin a concrete SDK and feeds before testing. |

OpenWrt 24.10 exposes `USE_APK` only behind `BROKEN`; 25.12 defaults it to enabled.
In the 25.12 packaging code, `PKGARCH:=all` becomes APK metadata `arch:noarch`.
APK filenames follow the package/version form, not the vendor `*_all.ipk` naming form.
The current 25.12 feed lists `prometheus-node-exporter-lua-2026.06.05-r1.apk`.
Renaming an IPK to APK does not convert it.
Do not mix Alpine Linux binary repositories into OpenWrt merely because both use apk. [S12, S13]

The official SDK action still shows an IPK-only artifact glob in its README example.
Its current implementation supports signing inputs for both package formats and invokes package compile targets.
Choose the artifact glob from the selected SDK's actual output; do not copy `*.ipk` blindly into an APK lane.
Require a nonempty expected artifact set and inspect package metadata, dependency names, installed paths, and maintainer scripts. [S10]

The modern exporter retains the module ABI but changes buffering and adds optional TLS and zero-counter omission configuration.
The adapter must use `metric`, not assume the old global `output` implementation exists.
Keep zero-counter omission disabled when testing counter continuity.
Do not infer that modern exporter TLS options exist in the older vendor package. [S8]

Architecture-independent Lua does not make its binary dependencies portable across releases.
Resolve `libubus-lua`, Lua, uhttpd, and optional libraries from the target's compatible feeds.
Do not install a newer stock binary dependency over a vendor library merely to satisfy a package solver.
Kernel-package dependencies are particularly unsuitable for a generic telemetry collector.

Build success, runtime compatibility, and hardware validation are separate claims:

1. A package build proves archive construction and declared dependency metadata.
2. Runtime tests prove the ABI, parsing, deadlines, overlapping scrapes, and failure handling on the tested software combination.
3. Reference-device tests prove source semantics and ingestion only for the tested hardware and firmware.

No package was built here because implementation and software artifacts are outside this research task.
No combination is advertised as a working release.

## 5. Minimal adapter separation and package split

Use one small normalized record contract: source, field, value, unit, radio technology, carrier/cell role, availability, and known freshness.
The field inventory must distinguish an absent measurement from unsupported source coverage and collection failure.
Keep vendor-specific parsing in the adapter and output escaping in the collector.
This separation needs ordinary Lua modules, not a registration framework or plugin protocol.

Proposed packages:

| Package role | Contents and dependencies | Initial position |
|---|---|---|
| `prometheus-node-exporter-lua-cellular` | Collector, normalized field contract, validation, output. Depends on compatible exporter and Lua. | Small common package. No direct modem commands or vendor library dependency. |
| `prometheus-node-exporter-lua-cellular-gl` | Explicit GL passive-source adapter. Depends on common package and `libubus-lua`. | Reference adapter. Require the existing qualified GL modem service; do not install or replace that service. |
| Future producer adapter | Only mapping and transport for a separately qualified existing producer. | Create when needed, with producer-specific dependencies and support claims. |

Package names are recommendations, not created artifacts.
Adapter modules belong outside `prometheus-collectors` so the exporter does not load them as independent collectors.
Only the common cellular entry point belongs in the automatic collector directory.
Do not make module loading fail on stock OpenWrt because a GL-specific module or object is missing.
Select the adapter explicitly or through a narrow capability check; never probe arbitrary vendor methods or fall back to AT.

A two-package split is warranted by vendor capability and dependency isolation.
Separate “model”, “transport”, “serializer”, and “runtime framework” packages are not warranted.
If only one tiny GL adapter exists initially, the source separation still matters; package granularity must not become a framework exercise.
Do not create an empty stock adapter or describe kernel interface statistics as complete stock cellular support.

Reuse the exporter's existing netdev collector for Linux cellular-interface counters where appropriate.
Discover and document the cellular interface mapping; do not hard-code an interface name.
These are interface byte/packet/error/drop counters, not radio retransmissions or modem-specific session accounting. [S1; design distinction]

## 6. Existing producers and license obligations

| Producer or component | Practical result | License and packaging consequence |
|---|---|---|
| Existing GL signal service | Passive RAM reader established by E1; broad native telemetry remains unresolved. | Call the installed service. Do not copy or redistribute vendor binaries or private source dumps. Vendor redistribution rights were not established. |
| OpenWrt Lua exporter | Appropriate host and package pattern. | Apache-2.0 in the inspected package. Preserve required notices when copying or modifying its code. Dependency use does not assign this repository's license. |
| Upstream ModemManager collector | Uses `mmcli -L -J`, modem JSON, and `--signal-get -J`. Exports signal quality and numeric signal fields. | Exporter package metadata is Apache-2.0. Its optional package depends on `modemmanager` and `lua-cjson`; do not install it on GL merely for telemetry. |
| `3ginfo-lite` | Not a passive producer for the approved first architecture. Includes LuCI, serial readers, and reporting-mode writes. | GPLv3 project. A headless split would be new maintained packaging work, not an existing proven dependency. Do not import its parser into differently licensed project code without review. |

The current ModemManager collector uses unbounded `io.popen` reads and returns silently on several failures.
It does not supply the required whole-operation timeout or strict failure semantics.
It emits `metric(...)` repeatedly while iterating signal fields and modems; shared metric names can therefore produce repeated TYPE declarations.
Its existence is useful integration evidence, not proof that it meets this project's metric contract.
Use ModemManager data only where ModemManager already owns the modem and the selected query reads existing status.
Do not enable signal polling, scan networks, change modem ownership, or install its daemon automatically. [S14]

At pinned `3ginfo-lite` commit `b976ecb9d1b6a1683ccf672ef698a9526bdea390`:

- The main script calls `sms_tool` or `gcom` directly.
- It sets `COPS=3,0`, `COPS=3,2`, `CREG=2`, and `CEREG=2`.
- The RM520N-family path conditionally sets `C5GREG=2`.
- A missing device can produce an error JSON object with exit status zero.
- Helpers use temporary paths; stdout JSON does not mean the complete producer is fileless.
- Numbers and display values need schema validation; generic signal output does not prove separate LTE/NR coverage.
- Its LuCI package declares `sms-tool`, `comgt`, and `kmod-usb-serial-option` dependencies.

These findings exclude it from the passive first implementation, independently of its license.
A headless producer split is therefore **not warranted now**.
An optional future integration requires renewed access approval, ownership coordination, exact release inspection, and bounded execution. [S15]

If GPLv3 producer code is redistributed or modified, retain licensing/notices and meet applicable source-distribution obligations.
A separate process or package is not a universal legal exemption for copied or tightly combined code.
Keep independently written adapters and optional upstream producers distinguishable, and review the actual distribution before release.
The repository has no chosen license. This investigation does not choose one.
License selection and file-level reuse review remain publication gates. [S1, S15, S16]

## 7. Service permissions and endpoint exposure

The vendor init starts:

```text
/usr/sbin/uhttpd -f -c /dev/null -l / -L /usr/bin/prometheus-node-exporter-lua
```

It appends bind addresses, port, and keepalive arguments.
The defaults are the loopback interface and port 9100.
This is a dedicated exporter process. `/etc/config/uhttpd` for LuCI does not configure it automatically. [S2]

The inspected init sets no service user, group, jail, authentication realm, or TLS listener.
Under the usual root-run procd service context, a privileged exporter is expected; the live UID has not been measured here.
A collector package cannot safely change that identity for all other collectors without testing their permissions.
Do not claim least privilege merely because the new adapter only reads data.

Prefer a dedicated unprivileged identity if the exact ubus methods and other enabled collectors permit it.
Qualify native ubus access rules, socket permissions, required read-only files, and denied methods first.
LuCI/rpcd session ACLs do not automatically constrain a native libubus connection.
The inspected ubusd ACL implementation checks local peer credentials and bypasses these access checks for UID zero. [S18]
Do not grant serial-device access or broad modem-control permissions to solve telemetry access failures.

Keep loopback binding until endpoint exposure is approved.
A later remote scrape design must choose a specific trusted interface and an explicit source allowlist for IPv4 and IPv6.
Do not bind `*`, expose the endpoint to WAN, or assume HTTP credentials are protected on plaintext transport.
Authentication and TLS must be evaluated against the selected exporter/uhttpd versions, not the current upstream README alone.
Existing network protection can be used without introducing another HTTP server.

## 8. Remaining blockers and acceptance evidence

| Gate | Required evidence before implementation is called qualified |
|---|---|
| Full passive source coverage | Field-by-field mapping for all relevant reference telemetry, with missing fields explained and no unauthorized AT fallback. |
| Exact deployed runtime | Installed uhttpd, Lua plugin, ubus packages, effective command, UID, and applicable vendor patches. Feed availability is insufficient. |
| Outer deadline | Approved way to configure the existing handler deadline, or a separately qualified one-shot reader. Default UCI cannot do this today. |
| RPC failures | Stalled lookup, stalled invocation, disconnected bus, missing method, malformed reply, and permission denial must terminate predictably. |
| Overlap and cancellation | Simultaneous scrapes, a queued request, client disconnect, and deadline expiry must leave no unexpected workers or locked resources. |
| Exposition | Quotes, backslashes, LF, Unicode, missing values, zero, invalid numerics, duplicate labels, family grouping, and single TYPE per family. |
| Freshness | Timestamp meaning, source update cadence, stale data policy, and clock-change behavior must be established. |
| Packaging | Pinned SDK builds for promised IPK/APK lanes, metadata/dependency inspection, and installation/removal tests on appropriate runtime targets. |
| Reference validation | Compare approved source values with emitted values and real Prometheus ingestion on the one available reference device. |
| Publication | Parent privacy review, project license choice, dependency/reuse notices, and separate release approval. |

Fault tests should use fixtures or an isolated software environment first.
Do not stall the live router's ubus daemon or interrupt its modem manager to test timeout behavior.
No additional physical modem is a prerequisite for a useful reference release.
Other devices and stock adapters must remain explicitly untested until evidence exists.

## Sources

**S1 — Vendor artifact and pinned package definition.**
[Vendor Packages.gz](https://fw.gl-inet.com/releases/v21.02.3/packages-4.0/aarch64_cortex-a53/packages/Packages.gz),
[exact IPK](https://fw.gl-inet.com/releases/v21.02.3/packages-4.0/aarch64_cortex-a53/packages/prometheus-node-exporter-lua_2024.06.16-2_all.ipk),
[OpenWrt Makefile](https://github.com/openwrt/packages/blob/c279efb760ca516db7cdaee32216344df47f2203/utils/prometheus-node-exporter-lua/Makefile).

**S2 — Pinned exporter ABI and service.**
[Exporter](https://github.com/openwrt/packages/blob/c279efb760ca516db7cdaee32216344df47f2203/utils/prometheus-node-exporter-lua/files/usr/bin/prometheus-node-exporter-lua),
[init](https://github.com/openwrt/packages/blob/c279efb760ca516db7cdaee32216344df47f2203/utils/prometheus-node-exporter-lua/files/etc/init.d/prometheus-node-exporter-lua),
[UCI defaults](https://github.com/openwrt/packages/blob/c279efb760ca516db7cdaee32216344df47f2203/utils/prometheus-node-exporter-lua/files/etc/config/prometheus-node-exporter-lua).

**S3 — Upstream 21.02 package selections, not deployed GL versions.**
[uhttpd package](https://github.com/openwrt/openwrt/blob/openwrt-21.02/package/network/services/uhttpd/Makefile),
[ubus package](https://github.com/openwrt/openwrt/blob/openwrt-21.02/package/system/ubus/Makefile).
Their selected source commits are pinned in S4–S7.

**S4 — uhttpd defaults, Lua initialization, and forked requests.**
[main.c](https://github.com/openwrt/uhttpd/blob/15346de8d3ba422002496526ee24c62a3601ab8c/main.c),
[lua.c](https://github.com/openwrt/uhttpd/blob/15346de8d3ba422002496526ee24c62a3601ab8c/lua.c),
[proc.c](https://github.com/openwrt/uhttpd/blob/15346de8d3ba422002496526ee24c62a3601ab8c/proc.c).

**S5 — uhttpd request queuing and direct-child termination.**
[file.c](https://github.com/openwrt/uhttpd/blob/15346de8d3ba422002496526ee24c62a3601ab8c/file.c),
[relay.c](https://github.com/openwrt/uhttpd/blob/15346de8d3ba422002496526ee24c62a3601ab8c/relay.c).

**S6 — Lua ubus timeout units and lookup-before-invoke sequence.**
[lua/ubus.c](https://github.com/openwrt/ubus/blob/4fc532c8a55ba8217ad67d7fd47c5eb9a8aba044/lua/ubus.c),
[current comparison](https://github.com/openwrt/ubus/blob/414c60a2e29e72c2c4f573bc50ad54aef076d169/lua/ubus.c).

**S7 — ubus lookup, request timeout, and CLI implementation.**
[libubus.c](https://github.com/openwrt/ubus/blob/4fc532c8a55ba8217ad67d7fd47c5eb9a8aba044/libubus.c),
[libubus-req.c](https://github.com/openwrt/ubus/blob/4fc532c8a55ba8217ad67d7fd47c5eb9a8aba044/libubus-req.c),
[cli.c](https://github.com/openwrt/ubus/blob/4fc532c8a55ba8217ad67d7fd47c5eb9a8aba044/cli.c).

**S8 — Current exporter comparison.**
[Exporter](https://github.com/openwrt/packages/blob/70ac3a84406cab03865c55be84de960af815436a/utils/prometheus-node-exporter-lua/files/usr/bin/prometheus-node-exporter-lua),
[init](https://github.com/openwrt/packages/blob/70ac3a84406cab03865c55be84de960af815436a/utils/prometheus-node-exporter-lua/files/etc/init.d/prometheus-node-exporter-lua),
[24.10 package](https://github.com/openwrt/packages/blob/a714c6e379f9111b1114e9e585e43f04f266e41a/utils/prometheus-node-exporter-lua/Makefile),
[25.12 package](https://github.com/openwrt/packages/blob/9822cc9ee869d3cc1c5124a8697de4e8cd935d11/utils/prometheus-node-exporter-lua/Makefile).

**S9 — Prometheus exposition specification.**
[Text format, escaping, TYPE uniqueness, and grouping](https://github.com/prometheus/docs/blob/d606eaedccf61d4caf7d4185e6b7786537e8e287/docs/instrumenting/exposition_formats.md).

**S10 — Official package-only SDK action.**
[README](https://github.com/openwrt/gh-action-sdk/blob/f5813d30eeef3534b58ac7e79c5d8842b6035434/README.md),
[entrypoint](https://github.com/openwrt/gh-action-sdk/blob/f5813d30eeef3534b58ac7e79c5d8842b6035434/entrypoint.sh),
[25.12.6 SDK listing](https://downloads.openwrt.org/releases/25.12.6/targets/mediatek/filogic/).

**S11 — Current release and Lua ABI.**
[25.12.6 release](https://github.com/openwrt/openwrt/releases/tag/v25.12.6),
[Lua package](https://github.com/openwrt/openwrt/blob/v25.12.6/package/utils/lua/Makefile).

**S12 — Package-manager defaults.**
[24.10 configuration](https://github.com/openwrt/openwrt/blob/v24.10.4/config/Config-build.in),
[25.12.6 configuration](https://github.com/openwrt/openwrt/blob/v25.12.6/config/Config-build.in),
[pinned current snapshot configuration](https://github.com/openwrt/openwrt/blob/102f4eebca24a5205aad14c183401d7799862e4d/config/Config-build.in).

**S13 — Current package-format construction.**
[IPK/APK packaging implementation](https://github.com/openwrt/openwrt/blob/v25.12.6/include/package-pack.mk),
[apk package](https://github.com/openwrt/openwrt/blob/v25.12.6/package/system/apk/Makefile),
[25.12 package-feed listing](https://downloads.openwrt.org/releases/packages-25.12/aarch64_cortex-a53/packages/).

**S14 — Optional upstream ModemManager collector.**
[Collector](https://github.com/openwrt/packages/blob/70ac3a84406cab03865c55be84de960af815436a/utils/prometheus-node-exporter-lua/files/usr/lib/lua/prometheus-collectors/modemmanager.lua),
[package dependencies](https://github.com/openwrt/packages/blob/70ac3a84406cab03865c55be84de960af815436a/utils/prometheus-node-exporter-lua/Makefile).

**S15 — Pinned 3ginfo-lite producer, package, and license.**
[Makefile](https://github.com/4IceG/luci-app-3ginfo-lite/blob/b976ecb9d1b6a1683ccf672ef698a9526bdea390/luci-app-3ginfo-lite/Makefile),
[main script](https://github.com/4IceG/luci-app-3ginfo-lite/blob/b976ecb9d1b6a1683ccf672ef698a9526bdea390/luci-app-3ginfo-lite/root/usr/share/3ginfo-lite/3ginfo.sh),
[RM520N-family parser](https://github.com/4IceG/luci-app-3ginfo-lite/blob/b976ecb9d1b6a1683ccf672ef698a9526bdea390/luci-app-3ginfo-lite/root/usr/share/3ginfo-lite/modem/usb/2c7c0801),
[GPLv3 license](https://github.com/4IceG/luci-app-3ginfo-lite/blob/b976ecb9d1b6a1683ccf672ef698a9526bdea390/luci-app-3ginfo-lite/LICENSE).
Source inspection is not qualification of a separately released binary package.

**S16 — Apache License redistribution conditions.**
[Apache License 2.0, section 4](https://www.apache.org/licenses/LICENSE-2.0).

**S17 — GL public feature documentation.**
[Cellular interface guide](https://github.com/gl-inet/docs4.x/blob/master/docs/en/docs/interface_guide/internet_cellular_v4.7.md).
This documents UI features, not a stable passive backend ABI.

**S18 — Native ubus authorization.**
[ubusd ACL implementation](https://github.com/openwrt/ubus/blob/4fc532c8a55ba8217ad67d7fd47c5eb9a8aba044/ubusd_acl.c).
