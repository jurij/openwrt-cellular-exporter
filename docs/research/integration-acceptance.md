# Scrape integration and single-X3000 acceptance

## Result and scope

The existing Prometheus and Grafana services can remain the monitoring backend.
The reference router does not yet have an exporter installed or listening on TCP 9100.
End-to-end cellular scraping is therefore **not qualified**.

Use a package-only installation, an explicitly scoped listener, and one additional scrape job.
Do not add a database, settings page, firmware image, or per-scrape JSON files.
Keep intermediate structured data in memory.

This note answers [Qualify scrape integration and X3000 acceptance](https://github.com/jurij/openwrt-cellular-exporter/issues/4).
It proposes later tests, not permission to install or change production systems.
Only one physical X3000 is required.
Other firmware, modem combinations, and adapters remain untested until separately qualified.

## Evidence already obtained

The following observations came from bounded, read-only checks of the reference router and monitoring host.
Private addresses, hostnames, identifiers, topology, and raw responses are deliberately omitted.

| Check | Observed result | Boundary |
| --- | --- | --- |
| Administration | Existing, host-key-verified SSH access worked. | This is not a recovery or reboot test. |
| Exporter installation | No installed `prometheus-node-exporter-lua` package appeared in the installed-package inventory. | Availability in a feed does not prove installation. |
| Runtime dependencies | Lua 5.1.5, `libubus-lua`, and uhttpd were installed. `luasocket` and `uhttpd-mod-lua` were absent from that inventory. | Dependency installation and coexistence remain untested. `uhttpd-mod-ubus` is not `uhttpd-mod-lua`. |
| Installed webserver | uhttpd identified as `2021-03-21-15346de8-2`. Its help reported a 60-second script timeout. | No exporter-specific live process exists. Package version and help do not exclude vendor patches. |
| Candidate data service | The `modem.signal` ubus object existed. | Existence alone proves neither complete fields nor freshness. No modem RPC was invoked during this integration review. |
| Listener | Router listener inspection found no TCP 9100 listener. | No HTTP metrics endpoint was available to test. |
| Monitoring-origin path | A TCP connection from the monitoring host to the router's existing SSH service succeeded and returned an SSH banner. | This proves that service's path, not permission for another port. |
| Monitoring-origin port 9100 | The bounded TCP connection failed with connection refused. | This is consistent with the missing listener. It does not prove a firewall allow or deny decision. |
| Router input policy | Configured LAN input was ACCEPT. The inspected runtime LAN input chain ended in the LAN accept chain after earlier rules. | No complete firewall audit or non-monitoring-client denial test occurred. |
| Prometheus | Service active; runtime build reported `2.53.3+ds1`; listener bound to loopback. | No Prometheus listener change is needed for an outbound scrape. |
| Actual scrape timing | Runtime configuration reported global interval 5s, timeout 5s, and evaluation interval 5s. Active jobs used intervals 5s, 15s, 30s, and 1m, with timeouts 4s or 5s. | These are existing-job settings, not cellular-source sampling rates or recommended cellular defaults. |
| Existing targets | The inspected active targets were up. No cellular job was present. | Existing healthy targets do not validate a future exporter. |
| Grafana | Health endpoint reported version 13.2.3 and database `ok`. A provisioned Prometheus datasource used server-side proxy access to local Prometheus. | No authenticated datasource query or browser rendering test occurred. |
| Datasource timing | Provisioned datasource minimum interval was 5s. | This does not cause Prometheus to scrape every 5s. |
| Resources | A router snapshot showed available RAM and overlay space, with no swap. | No exporter CPU, memory, storage growth, or load comparison was possible. |

No packages, services, firewall rules, monitoring configuration, or remote files were changed.
No stopped workload was started.
No WAN test, reconnect, modem-control command, or reboot was performed.

A configuration check through `promtool check config /dev/stdin` failed because opening `/dev/stdin` was denied.
No candidate configuration was loaded, and no successful `promtool` validation is claimed.
The later validation gate remains open.

## Reproducing the read-only preflight

These examples require existing administrative access; they do not install tools or change configuration.
Keep their output private unless it has been reviewed for publication.

1. On the reference router, inspect the selected package records and listeners:

```sh
opkg status prometheus-node-exporter-lua lua luasocket libubus-lua uhttpd uhttpd-mod-lua
netstat -lnt
/usr/sbin/uhttpd -h
```

2. On the monitoring host, set `REFERENCE_ROUTER` to the intended private destination in the shell environment.
3. Run this bounded port check from the actual Prometheus network context:

```python
import os
import socket

host = os.environ["REFERENCE_ROUTER"]
for port in (22, 9100):
    try:
        with socket.create_connection((host, port), timeout=3):
            print(port, "connected")
    except OSError as error:
        print(port, type(error).__name__)
```

Use the existing management port instead of 22 if the deployment uses a different port.
This tests only two specified services; it is not a network scan or an HTTP metrics test.
A successful management connection does not establish exporter access.

4. Read Prometheus `/api/v1/targets?state=active` through its existing authorized local access path.
5. Retain only each relevant target's `health`, `scrapeInterval`, and `scrapeTimeout` for timing verification.
6. Read only the global timing fields from `/api/v1/status/config`; never publish the complete configuration response.
7. Inspect the exact provisioned Prometheus datasource's type, proxy access, and minimum interval without printing credentials.

If Prometheus runs in a container, probing from its host is not an equivalent network-context test.
Inspect runtime listeners and the applicable input rules separately; a refused connection alone does not identify the rejecting layer.

## Proven package and listener prerequisites

The inspected vendor feed offers `prometheus-node-exporter-lua` version `2024.06.16-2`.
Prior package inspection matched its exporter, init script, and UCI configuration to the pinned upstream revision below.
This review independently read those pinned source files. [1–3]

The exporter requires `libc`, `luasocket`, `lua`, `uhttpd`, `uhttpd-mod-lua`, and `libubus-lua`. [1]
The proposed cellular adapter must declare any additional parser or source dependencies explicitly.
An architecture-independent script package is not automatically compatible with every firmware runtime.

Before installation:

1. Verify the firmware, Lua version, package architecture, and existing package versions.
2. Resolve dependencies against the selected vendor feeds without mixing incompatible release feeds.
3. Inspect package file ownership, maintainer scripts, and dependency changes before applying them.
4. Reject plans that replace modem management, shared webserver packages, or unrelated configuration without separate approval.
5. Keep the existing modem owner; do not install ModemManager solely for telemetry.
6. Verify the complete approved field inventory and each adapter's read-only source contract.
7. Report missing source fields instead of silently adding direct AT access.
8. Verify free storage against the actual package and dependency sizes.

The default exporter configuration selects logical interface `loopback` and port `9100`. [2]
Its init script starts a separate uhttpd process, not the management webserver's configured instance. [3]
Changing the management server's configuration will not automatically configure this exporter.

`listen_interface` selects an OpenWrt logical network interface.
The script resolves both its IPv4 and IPv6 addresses; it is not an arbitrary literal-address field.
The special value `*` requests a wildcard bind.
An unavailable selected interface defers startup. [3]
Verify actual listeners after startup and after any later address change.

## Least-exposure access design

**Preferred pilot:** bind only the required private interface and permit the monitoring source explicitly.
Keep loopback binding until that exposure change has separate approval.
Do not choose `*` merely to make an initial scrape succeed.

A listener bind limits destination addresses; it does not authenticate clients.
The inspected exporter service does not configure TLS or HTTP authentication. [3]
A Prometheus `collect[]` parameter selects collectors; it is not an authorization boundary. [4]

A router LAN can include clients that bypass a downstream firewall.
A downstream allow rule therefore does not establish router-local isolation.
The observed accepting LAN policy makes this distinction important.

Before exposing the endpoint:

1. Identify the source address that the router actually sees, including any address translation.
2. Inspect the router input path for that source and destination port.
3. Inspect any intervening filtering path without changing unrelated routing or failover.
4. Account for both address families when the selected interface has IPv4 and IPv6 addresses.
5. Define one permitted monitoring source and explicitly exclude other reachable sources.
6. Verify the policy with allowed-source and disallowed-source tests after approval.

An allow rule alone does not restrict other clients when the surrounding input policy already accepts them.
Do not publish the endpoint through WAN forwarding or a public reverse proxy.
Treat cell identifiers as location-sensitive even on a private metrics endpoint.
Keep SIM identifiers, device identifiers, credentials, SMS, and raw modem responses out of metrics and logs by default.

If router-local isolation is impractical, loopback plus an existing authenticated private transport is an alternative.
A new tunnel adds service ownership and recovery work; it is not the default recommendation.
No tunnel, ACL, source-address translation behavior, or end-to-end isolation was qualified here.

## Minimal scrape and failure contract

Keep existing jobs, their intervals, the Prometheus store, and the Grafana datasource unchanged.
Add one explicit job only after installation and endpoint qualification.

The following **unvalidated example** starts with a 15-second interval and a 5-second scrape timeout.
These are proposed pilot settings, not measured collector performance or source-update guarantees.
The collector name `cellular` is provisional until implementation fixes its name.

```yaml
scrape_configs:
  - job_name: cellular
    scrape_interval: 15s
    scrape_timeout: 5s
    metrics_path: /metrics
    params:
      collect[]: [cellular]
    static_configs:
      - targets: [reference-router.invalid:9100]
```

Merge the job into the existing configuration; do not replace its `scrape_configs` list.
The reserved `.invalid` hostname is a placeholder, not a deployment target.
Use an explicit job interval instead of inheriting the observed 5-second global interval. [5]
A shorter interval cannot create source measurements that the native producer has not made.

The producer deadline, cleanup, encoding, and transport must fit below the scrape timeout with measured margin.
The scrape timeout must not exceed the interval.
If the producer cannot meet this budget, resolve the source or process design before enabling the job.
Do not silently extend timeouts or introduce a sampler to hide an unbounded call.

The inspected exporter catches Lua exceptions but has no collector execution deadline or cross-request lock. [4]
It sends HTTP 200 before collection.
`node_scrape_collector_success` records exception handling, not semantic validity or freshness.
Unknown collector names are ignored, so a successful empty scrape must not pass acceptance.

Required implementation behavior:

- Bound source execution and response size before parsing or buffering untrusted source data.
- Finish parsing and validation before emitting a field group, preventing misleading partial success.
- Serialize any unsafe shared access across requests, not only within one Lua interpreter.
- Bound lock waiting and recover owned locks after failure.
- Terminate owned subprocess descendants and release locks after timeouts or client cancellation.
- Do not assume a package-local lock coordinates with the native modem manager.
- Load modules without source I/O; the exporter loads all installed collector modules at startup.
- Escape labels and keep label values bounded; never label by timestamps or complete source responses.
- Omit unavailable measurement samples rather than substituting zero or repeating cached values as fresh.
- Export source availability, collection success, and source observation time separately from signal values.
- Treat unknown source time as unknown freshness; a successful cache read does not establish measurement age.
- Use normal scrape timestamps for samples and separate timestamp gauges for source age.

uhttpd script timeouts and HTTP keepalive are not source deadlines. [3, 7]
A client timeout does not prove cleanup of descendant processes.
Forked HTTP handlers do not share a Lua-table cache. [7]
A background sampler is conditional on demonstrated need, not part of the minimal installation.

Keep transport health (`up`), collector success, and source freshness separate in queries and dashboards.
Prometheus staleness removes absent series; it cannot detect old values repeatedly emitted as current. [6]
Grafana should show gaps or unknown states, not zero or green replacements.
Set a cellular-specific panel minimum interval if needed; preserve the existing datasource's shared setting.
Choose freshness thresholds only after measuring the native update cadence and qualifying its timestamp semantics.

## Resource and rollback boundaries

Measure exporter process memory, owned child processes, scrape duration, sample count, and response bytes during the pilot.
Compare router CPU/load and available RAM before and during the bounded test.
Inspect exporter-owned storage for unexpected growth.
Set response and series limits from the approved field inventory, not arbitrary fleet-scale budgets.
Check limits against the deployed Prometheus version before configuring them. [5]

Reject unexplained process accumulation, retained locks, repeated deadline failures, or new persistent per-scrape files.
Do not promise a resource ceiling from the router's current free-memory snapshot.
Keep operational logs bounded and free of source payloads.

Before the authorized pilot, preserve exact owned configuration and prior package versions privately.
Record which dependencies were newly installed and which services already existed.
Record whether the base exporter is shared with other collectors.

Rollback sequence, after approval:

1. Remove only the new cellular scrape job and its owned alerts or dashboard objects.
2. Validate the resulting Prometheus configuration before its approved reload.
3. Stop the owned exporter only if no existing collectors depend on it.
4. Remove or downgrade only the owned cellular packages using the recorded package inventory.
5. Restore only the listener and access rules introduced by this pilot.
6. Remove newly installed dependencies only when no remaining package or service needs them.
7. Verify prior collectors, management access, modem service, and existing connectivity observations.

Collector removal or replacement requires the appropriate exporter restart because modules load at startup. [4]
Do not restart the router, modem manager, management webserver, or network merely to reload a collector.
Preserve Prometheus history; rollback does not require deleting its database.
An unexpected dependency replacement or loss of the known access path stops the pilot.

## Later acceptance gates — none executed here

Run offline failure tests before the live pilot.
Use synthetic fixtures and a controllable fake source, not deliberate production WAN interruption.
Fixture results establish code behavior, not unobserved radio modes.

| Gate | Practical test | Required evidence |
| --- | --- | --- |
| Package build | Build only the package with a pinned compatible SDK; inspect its archive and dependency plan. | Installable artifact, checksum, owned-file manifest, and no firmware build requirement. |
| Parser and field coverage | Test every approved field, units, boundaries, escaping, missing fields, malformed JSON, error objects, and invalid numbers. | Field-by-field contract coverage; no missing value becomes zero. Unresolved required fields remain blockers. |
| LTE/NR attribution | Test LTE, NR, and NSA fixtures; compare the currently available live mode with a qualified native source. | LTE anchor and NR measurements remain distinct. Unobserved live modes are labelled untested. |
| Freshness | Freeze, omit, and skew source timestamps in an isolated test source. | Old or unknown-age data cannot appear as a fresh valid measurement. |
| Exporter failure | Fail module loading and source reads in an isolated instance; stop only that instance. | Distinct endpoint, collector, and source failures; expected recovery without modem changes. |
| Timeout cleanup | Delay or hang the fake producer, then disconnect the client. | Bounded return, no surviving owned descendants, and released owned locks. |
| Overlapping scrapes | Send two concurrent requests and one subsequent request against the isolated source. | Safe bounded coordination, no corrupt output, and successful next collection. |
| Install and permissions | Install the reviewed package under separate approval; inspect actual processes, listeners, and source access. | Correct dependency state and read permissions without a replacement modem manager. |
| Endpoint security | Test from the approved monitoring origin and an existing disallowed origin. | Successful permitted scrape and denied unapproved access for every exposed address family. |
| Source-to-metric agreement | Compare close-in-time source observations and exporter output for every available field group. | Values, units, labels, source times, and availability agree. No raw captures are published. |
| Prometheus ingestion | Validate with `promtool check config`; lint actual endpoint output with `promtool check metrics`; enable the approved job. | Expected cellular series, correct runtime interval/timeout, advancing scrapes, collector health, and source age. HTTP 200 alone fails this gate. |
| Grafana | Query through the existing datasource and render the cellular panels. | Real signal/connection data, clear missing states, correct units, and no new database or datasource dependency. |
| Resource behavior | Observe several normal scrapes plus bounded overlap and timeout tests. | Recorded duration, response size, process memory/count, CPU/load, and storage behavior without accumulation. |
| Upgrade and removal | Upgrade or reinstall the owned package, then exercise the scoped rollback and restoration. | Correct loaded collector version, preserved settings, clean removal, and surviving unrelated services. |
| Preserved operation | Compare native modem/service status and existing connectivity observations before, during, and after the pilot. | No new interruption correlated with the exporter. Do not trigger a WAN outage, reconnect, or reboot. |

A missing live field is not automatically unsupported hardware.
Resolve whether its cause is the current radio state, producer limitations, permissions, parsing, or implementation.
Do not reduce the required modem-field inventory to make a release pass.

A prerelease needs a built artifact and successful basic live collection, ingestion, safe runtime behavior, and scoped rollback evidence.
A regular release needs the complete agreed field coverage and acceptance gates.
Record outstanding modes and gates explicitly instead of inventing availability percentages or requiring a full-day observation window.

## Public primary sources

1. [Pinned package definition](https://github.com/openwrt/packages/blob/c279efb760ca516db7cdaee32216344df47f2203/utils/prometheus-node-exporter-lua/Makefile) and [vendor package index](https://fw.gl-inet.com/releases/v21.02.3/packages-4.0/aarch64_cortex-a53/packages/Packages.gz).
2. [Pinned default UCI configuration](https://github.com/openwrt/packages/blob/c279efb760ca516db7cdaee32216344df47f2203/utils/prometheus-node-exporter-lua/files/etc/config/prometheus-node-exporter-lua).
3. [Pinned exporter service](https://github.com/openwrt/packages/blob/c279efb760ca516db7cdaee32216344df47f2203/utils/prometheus-node-exporter-lua/files/etc/init.d/prometheus-node-exporter-lua).
4. [Pinned exporter implementation](https://github.com/openwrt/packages/blob/c279efb760ca516db7cdaee32216344df47f2203/utils/prometheus-node-exporter-lua/files/usr/bin/prometheus-node-exporter-lua).
5. [Prometheus 2.53.3 configuration](https://github.com/prometheus/prometheus/blob/v2.53.3/docs/configuration/configuration.md) and [promtool commands](https://github.com/prometheus/prometheus/blob/v2.53.3/docs/command-line/promtool.md).
6. [Prometheus 2.53.3 staleness semantics](https://github.com/prometheus/prometheus/blob/v2.53.3/docs/querying/basics.md#staleness).
7. Pinned uhttpd source: [defaults](https://github.com/openwrt/uhttpd/blob/15346de8d3ba422002496526ee24c62a3601ab8c/main.c), [Lua handler](https://github.com/openwrt/uhttpd/blob/15346de8d3ba422002496526ee24c62a3601ab8c/lua.c), and [process handling](https://github.com/openwrt/uhttpd/blob/15346de8d3ba422002496526ee24c62a3601ab8c/proc.c).

Runtime observations above are deployment evidence, not properties guaranteed by these upstream sources.
Public reproductions should use their own reference router and monitoring host.
