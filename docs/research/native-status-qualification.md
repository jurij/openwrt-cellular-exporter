# Native vendor status qualification

Research for [issue #6](https://github.com/jurij/openwrt-cellular-exporter/issues/6), observed on 2026-10-20.
Reference: GL-X3000, GL firmware 4.8.3, OpenWrt 21.02-SNAPSHOT, RM520N-GL.
The prior inventory identifies `gl-sdk4-modem git-2025.290.37125-7c3efcc-1`. [S0]

## Result and architecture impact

**Use the native vendor cell-status response for separate LTE and NR measurements.**
The qualified `modem.get_cells_info` read returned two explicitly labelled serving rows: `LTE FDD` and `NR5G-NSA`.
Together they contained seven quality values, two bands, and three bandwidth fields. [S4]
A custom AT-cache parser is therefore **not required for this observed UI field set**.

Recommend these distinct GL adapter inputs:

- Native `get_cells_info`: technology-labelled serving-cell observations and band information.
- Native `get_status`: SIM/connection state, operator metadata, and vendor-reported usage.
- Native `get_info`: discovery and capability metadata, not a per-scrape requirement.
- Existing `modem.signal.get_signals`: optional generic signal history and vendor collector timestamp.
- Kernel/netifd: interface counters, interface state, and explicitly named interface uptime. [S0–S4]

This updates the earlier recommendation to leave every compiled status method uncalled.
Normal supported reads through the vendor owner do not require a second serial reader.
They also do not prove measurement freshness, complete carrier aggregation, or every possible modem capability.
Temperature and bearer-session history remain **unproven extensions**, not demonstrated missing UI values.
Keep the generic OpenWrt package independent of the GL adapter and its firmware-specific RPC transport.
This note authorizes no implementation, installation, polling change, or release.

## Why these exact reads qualified

The vendor's v4.8–v4.9 guide separates **View More Information**, **Traffic Statistics**, and **Historical Signal Record** from settings, scans, and AT commands. [S1]
The installed UI supplies the exact call contracts, not merely suggestive method names. [S2]

| Method and arguments | Normal installed UI behavior | Bounded live result |
| --- | --- | --- |
| `modem.get_info`, `{}` | Discovers `modems[]`; selects a modem using its returned `bus`. | One modem metadata object. |
| `modem.get_status`, `{}` | Reads `modems[]`; normal status refresh uses a 30-second timer. | One modem status object. |
| `modem.get_cells_info`, `{"bus": <discovered bus>}` | Internet card reads `cells`; runs immediately when a SIM is present, then normally every 30 seconds. Modem-management Refresh uses the same method. | Two serving-cell rows. |

The dispatcher accepts JSON-RPC `call` with `[sid, object, method, args]`.
It validates the request, checks access, and routes native methods through `/cgi-bin/glc` to `/usr/lib/oui-httpd/rpc/modem.so`.
The inspected configuration maps `/rpc` to `/usr/share/gl-ngx/oui-rpc.lua`.
No alternate Lua modem handler was present. [S3]

These facts qualify ordinary read-oriented **operations**, not a claim that proprietary internals never update their own caches or logs.
The exact UI read contracts and dispatch provide sufficient operational evidence; complete native-code proof is not required.
They do not authorize `send_at_command`, arbitrary `get_*` methods, debug export, scans, or setters.

The live test used authorized SSH and the dispatcher's existing loopback trust path.
It made **one call per method**, with a 30-second HTTP deadline, no retries, and the discovered bus selector.
All three returned valid result objects in less than one second each. [S4]
No raw AT command, serial/QMI client, scan, setting change, restart, installation, or remote file write was performed.
Responses stayed in memory; only schema, controlled radio-mode names, comparisons, and coverage counts were retained.
No browser page was opened, so no UI background polling was started.
Root/local-trust access is not evidence of a least-privilege telemetry permission model.

## Actual UI and live field inventory

**Live** means present in this one native response set.
**UI-only** means an installed consumer or vendor guide supports it, but this test did not obtain a corresponding value.
Neither label establishes original measurement time or behavior in another radio mode.

| Field family | Evidence and exact shape | Meaning and export constraint |
| --- | --- | --- |
| LTE serving row | **Live:** `type="servingcell"`, `mode="LTE FDD"`; integer `rsrp`, `rssi`, `rsrq`, `sinr`; integer `band`; string `ul_bandwidth`, `dl_bandwidth`. | Explicit LTE identity, including duplex in the mode string. Match the observed mode shape; equality with literal `LTE` alone would miss it. |
| NR serving row | **Live:** `type="servingcell"`, `mode="NR5G-NSA"`; integer `rsrp`, `rsrq`, `sinr`; integer `band`; string `dl_bandwidth`. | Separate NR values. No NR `rssi`, `ul_bandwidth`, `id`, or `tx_channel` in this response. Absence is not zero. SA remains untested. |
| Signal presentation | **Live:** LTE has `rsrp_level`, `rssi_level`, `rsrq_level`, `sinr_level`; NR has the three applicable level fields. All are integers. **Code:** UI displays vendor badges and signal-quality categories. | Preserve vendor scale separately from physical quantities. Native quality values are display-ready, not raw QENG positional codes. Do not apply the raw LTE SINR conversion again. [S0, S2, S4] |
| Physical units | RSRP/RSSI are power quantities; RSRQ/SINR are ratios. Prior module-format research establishes dBm/dB context. Native UI displays the returned values directly. [S0, S2] | Use dBm/dB only for the qualified fields. This test did not independently prove every proprietary conversion, range, sentinel, or validity condition. |
| Bandwidth representation | **Live:** all three bandwidth strings use a numeric value followed by `M`, not raw integer enumeration codes. | Preserve this source format in parsing requirements. The inspected consumer adds no unit conversion; confirm the native `M`→MHz mapping before promising normalized bandwidth units. |
| Cell identifiers | **Live:** LTE has string `id` and `tx_channel`; NR does not. **Code:** Internet details explicitly shows `id`; modem-management displays every returned cell key. | These names do not prove PCI, TAC, or channel semantics. Do not reinterpret `tx_channel` as duplex or frequency. Exclude values from public metrics, logs, and fixtures pending an explicit privacy/semantic decision. |
| Active carriers and neighbors | Two returned rows, both `servingcell`. No primary/secondary carrier role, carrier activation state, or neighbor row appeared. | A returned-row count is not an active component-carrier count. This endpoint closes the observed serving-band gap, not proof of all aggregated carriers. |
| Other UI cell fields | **UI-only:** Internet details recognizes `RF`, `rscp`, `snr`, and `ecio`; none appeared in these live rows. | These are cross-modem consumer capabilities, not missing RM520N values. `RF` has no qualified meaning here. |
| Generic signal | **Live:** `simcard.signal` contains integer `mode`, `strength`, `rsrp`, `rssi`, `rsrq`, `sinr`, and string `network_type`. | Network label reported NSA. All four quality values matched the LTE row, while the three NR quality values differed. This supports LTE attribution for this response set, not a universal selection rule. Keep generic values distinct from technology-labelled cells. |
| SIM/operator state | **Live:** string `current_sim`; integer `simcard.status`, `switch_status`, `auto_switching`; string `carrier`, `mcc`, `mnc`. | Preserve MCC/MNC strings. UI treats `switch_status=1` as switching and uses `auto_switching` for initialization display. Do not equate these fields with independent 3GPP registration-domain states. |
| Connection state | **Live:** integer `network.status`; the observed value maps to connected. **Code:** `0=CONNECTED`, `1=CONNECTING`. | Never coerce zero to disconnected. Other values need explicit unknown handling. Connection state is not Internet reachability or an active NR bearer. |
| Vendor usage | **Live:** `network.traffic_total` is a decimal numeric string. UI displays it as Data Used. **Code/docs:** per-SIM totals are user-editable and resettable; UI converts edited totals to bytes. | Model as vendor-reported usage, not a permanently monotonic lifetime counter or carrier billing truth. Do not infer LTE/NR traffic split or transfer kernel counter lifecycle semantics. |
| Per-SIM traffic settings | **UI-only:** `get_traffic_config({bus})` consumer has `sim1_traffic_total`, `sim2_traffic_total`, limits and `save_to_flash`. This method was not called. | These are documented UI data/settings, not live-qualified per-SIM totals in this test. Reading the current status requires no execution of the traffic-maintenance script. |
| Model and capabilities | **Live:** `get_info.modems[]` has strings `name`, `version`, `vendor`, `bus`, `at_port`; arrays `devices`, `protocols`; capability booleans including `signal_support`. | Allowlist metadata. The actual native field is `vendor`, not the earlier consumer spelling `verdor`. Bus and device paths are internal selectors, not public identity labels. |
| History, interface state and counters | Existing RAM reader provides generic signal history with `timestamp`; kernel/netifd sources provide interface counters and uptime. [S0] | Existing evidence remains valid. These three native responses add no observation timestamp, interface counter set, or bearer uptime. Deduplicate shared physical devices. |
| Modem temperature and session history | No temperature, uptime, session start/duration, or reconnect field appeared. The inspected cellular consumers and v4.8–v4.9 guide establish no such modem-status display. | Do not make every hardware-query capability a product acceptance requirement. Router `mcu.temperature` is not modem temperature. Interface uptime is not modem bearer age. |
| Sensitive data | Metadata/status include SIM/device identifiers, phone number, APN, addresses and DNS; status also has an SMS count. | These are deliberately excluded, not telemetry acquisition gaps. Never retain whole responses or use them as public fixtures. |

The Internet details component filters `type="servingcell"` and shows a fixed field list.
The modem-management cell component instead renders all keys of each returned row. [S2]
Thus `tx_channel` is observable in that generic consumer even though absent from the Internet details field list.
Neither consumer establishes an exhaustive active-carrier inventory.
The separate AT-command UI offers a CA-information shortcut; invoking that shortcut would cross this research boundary.
Its existence is not proof of an automatically available CA-status table.

## Freshness and validity contract

The native methods returned **current request responses**, not proven fresh modem measurements.
None of their observed structures contained a source timestamp, last-success time, generation, expiry, or measurement-age field. [S4]
The calls were sequential, not an atomic LTE/NR/status snapshot.
The existing UI refresh interval describes request scheduling, not a backend freshness guarantee.

The adapter must distinguish:

1. Request completion and parse success.
2. Source-provided collection time, when available.
3. Original measurement age, which remains unknown for native cells/status.
4. Field presence and validity, including idle-state or invalid-value behavior not exercised here.

A scrape time or successful native call must not become a modem measurement timestamp.
The RAM history timestamp is assigned by the vendor collector; it cannot timestamp native cell rows. [S0]
Do not refresh an invented last-measurement time merely because unchanged values were returned.
Do not infer active NR transmission from the presence of a cached or otherwise untimestamped NR serving row.
Successful reads also do not prove native cache invalidation on disconnect or modem failure.

## Remaining boundary and smallest blockers

**No further approval is needed merely because these three qualified status handlers are compiled.**
The research already establishes a usable existing-owner data path for the observed UI telemetry.
Wayfinder and the project's implementation/deployment approval gates still apply.

The smallest remaining technical gaps are source-age/failure semantics and field normalization, not permission for an ordinary status read.
A design can expose explicitly freshness-unknown native observations with separate reader health.
It cannot claim complete, freshness-qualified modem measurements without further producer evidence.
Native bandwidth units, opaque identifier meanings, and invalid/sentinel behavior need targeted qualification before those specific normalized fields are promised.

Complete CA roles, modem temperature and bearer-session history have not been established as routine UI telemetry.
Keep them as optional capabilities unless a later source proves otherwise or the user explicitly expands scope.
If obtaining them requires direct modem access, reporting-mode changes, scans, or enabling a new producer, explain the gap and obtain a separate decision.
Never add an automatic direct-AT fallback.
A future repeated-polling design still needs bounded execution, concurrency handling and firmware compatibility tests; this one-shot result is not a load test.

## Primary sources and reproducibility

Installed paths identify primary firmware artifacts, not published private captures.
No proprietary binary, raw response, network address, cell identifier value, or authentication token accompanies this note.

- **[S0] Prior research:** [telemetry-sources.md at 4f32042](https://github.com/jurij/openwrt-cellular-exporter/blob/4f32042/docs/research/telemetry-sources.md), including module manual, RAM producer, kernel/netifd and traffic-script evidence. This follow-up supersedes its blanket unresolved native-status safety conclusion and its assumption that every thermal/session/CA extension is a demonstrated UI coverage gap.
- **[S1] Vendor documentation:** [Cellular v4.8–v4.9](https://docs.gl-inet.com/router/en/4/interface_guide/internet_cellular/), read from the [official documentation source at 644bd2f0e5e87736e245fa88566758489c244211](https://github.com/gl-inet/docs4.x/blob/644bd2f0e5e87736e245fa88566758489c244211/docs/en/docs/interface_guide/internet_cellular.md). Relevant sections: Auto Setup, Traffic Statistics, Historical Signal Record, Lock Tower and Modem AT Command. Direct rendered-page retrieval was blocked; the official source supplied the text.
- **[S2] Installed UI:** `/www/views/gl-sdk4-ui-internet.common.js.gz`: `getModemInfo`, `getModemStatus`, `getModemStatusTimeout`, `getCellsTimeout`, `ModemDetails.cellList`, `cellInfoKeys`, traffic editing and state constants. Compressed SHA-256: `5312562b1a99e6332e9d9d56796195033c7d1e614f366bfe9d522cdcce763aa4`. `/www/views/gl-sdk4-ui-modemmanage.common.js.gz`: `getCellInfo`, cell rendering and explicit AT-command controls. Compressed SHA-256: `adbaa5b65321080fd67a2899d669a34a00660c67b6a8ab9b5af0a9d2b08cee1d`.
- **[S3] Installed dispatch:** `/usr/share/gl-ngx/oui-rpc.lua` (`rpc_method_call`), `/usr/lib/lua/oui/rpc.lua` (`M.access`, `valid_rpc`, `M.call`, `glc_call`), and `/etc/nginx/conf.d/*.conf`. Native backend `/usr/lib/oui-httpd/rpc/modem.so` SHA-256: `5105747edee7bcbac9f67db25e5cab6849df9ec7324c2574003672485637a70d`. `/usr/share/gl-validator.d/modem.lua` is Lua bytecode; this assessment does not claim a source-level audit of it or the native backend.
- **[S4] Bounded native observation:** one successful `get_info({})`, one `get_status({})`, and one `get_cells_info({bus})`, completed 2026-10-20 around 11:39 UTC. Durations were approximately 0.20, 0.31 and 0.15 seconds. Schema and coverage were computed in memory. No response bodies were saved. This note is the sanitized observation record, not independently reproducible historical raw data.
