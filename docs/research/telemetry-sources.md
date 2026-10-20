# Cellular telemetry source qualification

Research for [Qualify complete cellular telemetry sources](https://github.com/jurij/openwrt-cellular-exporter/issues/2).

## Decision

Use the existing modem owner, not a second modem controller.
On the reference GL-X3000, passive sources support a useful exporter without new JSON files.
They do **not yet prove complete, current cellular telemetry**.

Recommend a GL-specific adapter with three separate source contracts:

1. Read the existing `modem.signal.get_signals` RAM buffer for generic signal, reported network type, selected slot, and collector timestamp.
2. Read kernel interface counters and netifd status for host-side traffic, interface state, and interface uptime.
3. Qualify the existing vendor AT cache as a separate, freshness-unknown source before using its LTE/NR fields.

Do not describe the first two sources as the complete product.
Keep distinct LTE/NR observations, active carriers, temperature, and modem-session coverage as explicit acceptance gaps.
Do not enable a direct AT fallback, invoke unproven modem RPC methods, or change modem polling to fill those gaps.

For stock OpenWrt, use a separate adapter for an **already installed and controlling** modem service.
ModemManager is the strongest documented structured-source candidate, but it is not installed on this reference device.
GL HTTP RPC and `modem.signal` are not generic OpenWrt interfaces.

This is a research recommendation, not implementation, installation, or release approval.

## Scope and evidence levels

Reference platform: GL-X3000, GL firmware 4.8.3, OpenWrt 21.02-SNAPSHOT, Quectel RM520N-GL.
The installed modem package is `gl-sdk4-modem git-2025.290.37125-7c3efcc-1`.
These identify the tested platform, not a universal compatibility promise.

The bounded follow-up inspection ran on 2026-10-20.
It reused earlier platform discovery and checked installed source files, existing caches, the proven RAM-buffer reader, and kernel/netifd state.
It did not install software, write router configuration, scan cells, send AT commands, execute candidate collectors, or call opaque modem HTTP RPC methods.
Raw cache responses were parsed in memory and were not retained or published.

Evidence labels:

- **L — Live:** a value, field, or structure was observed on the reference device.
- **C — Code/documented:** installed source code or an identified primary document supports the claim.
- **H — Hypothesis:** a plausible mapping or capability still lacks sufficient proof.
- **U — Unavailable in qualified sources:** no usable value exists in the inspected passive sources. This does not prove hardware inability.
- **X — Excluded:** sensitive information or a prohibited acquisition path, not missing hardware capability.

Live presence does not establish freshness, correct units, active use, or behavior in another radio mode.
A matching field in a manual establishes a candidate interpretation, not every firmware variant.

## Source inventory and safety

| Source | Evidence and contract | Boundary |
| --- | --- | --- |
| `modem.signal.get_signals` | **L+C.** Installed Lua implements a RAM-table reader. `time=0` returned one signal record. [S1] | No new modem query in this reader. The existing producer's opaque backend remains separate. Debug logging exists in the Lua code. |
| `/var/run/modem/at_cache.json` | **L.** Existing vendor cache has `AT_CACHE[]` entries containing only `cmd` and `result`. [S2] | Passive file read, but responses contain sensitive data. No per-command timestamp, success time, expiry, or generation field. |
| `/sys/class/net/<device>/statistics/*` | **L+C.** Eight standard RX/TX byte, packet, error, and drop counters were readable. [S4, S9] | Host interface counters, not per-radio-bearer counters or carrier billing. |
| netifd `network.interface` status/dump | **L+C.** Device-associated interfaces exposed state and numeric uptime. Upstream code defines interface uptime. [S4, S10] | A dump includes unrelated interfaces and addresses. Retain only the required device mapping and allowlisted status fields. |
| GL HTTP RPC `modem.get_info`, `modem.get_status`, `modem.get_cells_info` | **C.** Installed UI calls these methods. [S3] | Their native backend was not proven passive. No invocation during this research. UI polling is not a safety proof. |
| `/usr/bin/traffic_statistics` | **C.** Script maintains usage files and can change network configuration and reload networking. [S5] | Do not execute it as a telemetry helper, including superficially read-oriented entry points without full branch review. |
| `gl_modem`, `modem_AT`, serial/QMI tools | Existing control/query mechanisms, not qualified passive sources here. | No fallback use. A query can still contend with the production owner. |

The modem ubus namespace exposed `modem.signal`, not an object named `modem`.
The signal object also exposes `upload_cloud_signals`; that method is not part of the telemetry contract. [S1]

The earlier filesystem inspection established that the vendor cache resides on RAM-backed temporary storage.
Reading that existing cache is different from creating exporter-owned JSON files.
It does not make the upstream producer fileless or establish a cache freshness guarantee. [S2]

## Cellular coverage matrix

The matrix covers the relevant requested field families and additional fields found in those sources.
Rows marked unavailable remain visible requirements; they are not silently removed from scope.

| Field family | Reference-device evidence | Meaning, units, and remaining qualification |
| --- | --- | --- |
| Generic signal | **L+C:** RAM record has integer `rsrp`, `rssi`, `rsrq`, `sinr`, `strength`, and `mode`. [S1] | Preserve source identity. Do not turn proprietary `strength` or `mode` into a documented scale without a mapping. RAT attribution is not explicit. |
| Reported technology | **L:** RAM `network_type` reported `NR5G-NSA`. **C:** cache has explicit LTE and NR5G-NSA row types. [S1, S2] | NSA means non-standalone 5G with LTE anchoring. This badge is not evidence that generic quality fields measure NR. |
| Distinct LTE quality | **L+C:** cached LTE row contains numeric RSRP, RSRQ, RSSI, and SINR fields in the documented positions. [S2, S6] | RSRP/RSSI are power measurements; RSRQ/SINR are ratios. Manual maps LTE SINR rather than treating its raw integer as dB. Cache age remains unknown. |
| Distinct NR quality | **L+C:** cached NR5G-NSA row contains numeric RSRP, RSRQ, and SINR fields. [S2, S6] | NR SINR uses the documented NR interpretation, not the LTE conversion. No NR RSSI field exists in this response layout. |
| NR standalone quality | **C only:** manual defines an NR5G-SA layout. **U live:** not observed. [S6] | Do not claim live SA validation or reuse NSA offsets. No network-mode change was made to produce a sample. |
| Serving LTE/NR bands | **L+C:** both cached RAT rows contain numeric band fields. [S2, S6] | These are cached serving-row observations, not configured band lists. They are not proof of every currently active component carrier. |
| Channel and duplex information | **L+C:** row layouts support LTE EARFCN, NR ARFCN, and LTE duplex mode. [S2, S6] | Channel numbers are not frequencies in Hz. Live channel and cell values are not published. NR NSA lacks a separate duplex field here. |
| Serving bandwidth and NR subcarrier spacing | **L+C:** LTE UL/DL bandwidth and NR DL bandwidth/SCS fields were numeric. [S2, S6] | Decode enumerations using the module manual. NR DL bandwidth is only valid in RRC connected state. NR UL bandwidth is absent from the NSA layout. |
| Radio state | **L+C:** cache serving header reported `NOCONN`. [S2, S6] | Manual defines registered, camped, idle operation. It does not mean the host data interface is down. The cache cannot establish current RRC state. |
| Full carrier aggregation and carrier roles | **U qualified; C candidate:** UI exposes a cell collection. Cached `QCAINFO` was absent. [S2, S3, S6] | No verified complete primary/secondary carrier inventory, activation state, or per-carrier measurements. Serving LTE plus NR rows are not a full aggregation inventory. |
| Cell topology and identifiers | **L+C:** cached serving rows structurally contain cell/physical-cell fields. UI expects cell `type` and `mode`. [S2, S3, S6] | Cell ID, TAC, PCI, and channel combinations are location-sensitive. No identifier values are published. Count/role metrics still need complete source semantics. |
| Neighbor cells and scanned towers | **U passive; C UI feature.** The UI contains a separate tower-scan workflow. [S3] | Scanned tower fields are not evidence for a passive serving-cell source. Do not initiate scans or infer neighbor coverage from `get_cells_info` naming. |
| LTE CQI, transmit power, reception level | **L+C:** LTE row had numeric CQI and transmit-power fields; `srxlev` was invalid. [S2, S6] | CQI is a quality indicator. Transmit power uses tenths of dBm and is meaningful during traffic. Numeric idle fields are not automatically valid measurements. |
| Registration and roaming | **L+C:** existing CREG, CEREG, and C5GREG cache entries each had a numeric reporting mode and registration status. [S2, S6] | Preserve CS/EPS/5GS domains separately. Registered/roaming values require domain-aware decoding. Do not use CS denial alone to declare LTE data disconnected. Freshness is unknown. |
| Operator and network code | **L:** cached COPS response had mode, format, operator, and access-technology fields. **C:** UI consumes `simcard.carrier`. [S2, S3, S6] | Decode the operator using its format. Keep MCC/MNC as strings where required to preserve leading zeros. A selected/operator configuration is not a serving observation. |
| Selected SIM slot | **L+C:** RAM record contains integer `slot`. **C:** UI status contains `current_sim`. [S1, S3] | The live slot belongs to that signal sample. Preferred slot, failover policy, and supported slot count are separate configuration/capability data. |
| SIM readiness and switching state | **C only:** UI consumes `simcard.status`, `switch_status`, and `auto_switching`. [S3] | No safe complete live SIM-status reader was qualified. SIM presence, PIN readiness, registration, and data connection are different states. |
| Data-interface state | **L+C:** netifd device-associated interfaces were up and available, with `pending=false`. **C:** GL UI interprets `network.status=0` as connected. [S3, S4, S10] | Never coerce GL status zero to disconnected. Netifd state does not prove Internet reachability or current radio-bearer health. |
| Traffic bytes/packets/errors/drops | **L+C:** kernel RX/TX counters were present and nondecreasing across bounded samples; byte counters changed. [S4, S9] | Counters belong to the cellular interface. Deduplicate shared devices. Rates are downstream calculations, not measured radio capacity. |
| Per-SIM, billing-period, LTE/NR traffic split | **C only:** UI reads `network.traffic_total`; vendor script maintains SIM and network-type usage files. [S3, S5] | No validated counter reset, attribution, unit, or freshness contract. Do not relabel interface totals as billing or separate LTE/NR user-plane totals. |
| Interface/session uptime | **L+C:** netifd returned positive numeric uptime for associated qcm, DHCP, and DHCPv6 interfaces. [S4, S10] | Seconds since the interface entered its up state. These can share one device and have different lifecycles. This is not modem PDP/bearer session age. |
| Modem bearer/session history | **U qualified.** No passive bearer start/end time, reconnect count, or disconnect-cause history was proven. | Router uptime, signal-buffer history, RRC state, and interface uptime are not substitutes. Addresses and APN credentials are excluded. |
| Modem temperature | **U qualified; C hardware-query candidate:** no QTEMP cache entry or passive modem-temperature source was found in the inspected paths. [S2, S7] | The RM520N parser in 3ginfo-lite issues QTEMP, but was not run. A router MCU/SoC temperature is not modem temperature. |
| Model, firmware, capabilities | Reference model/firmware are established platform evidence. **L:** cache command inventory includes manufacturer/revision queries. **C:** GL `get_info` consumer exists. [S2, S3] | Do not export whole modem-info objects. Supported bands are capabilities, not active bands. No additional device-identification query is needed. |
| Preferences, locks, limits, protocol selection | **C:** UI supports LTE/NSA/SA band settings, SIM preference, operator/tower locking, and usage limits. [S3] | Settings are not observations. They can be documented separately; they cannot close active-band, active-carrier, or session gaps. |
| SIM/device identifiers, addresses, credentials, SMS | **X:** some exist in cache/backend responses. | Exclude IMEI, IMSI, ICCID, phone numbers, addresses, APN credentials, authentication/session tokens, and SMS data from metrics, fixtures, logs, and notes. |

No source in this matrix proves NR per-antenna measurements, complete modem thermal sensors, radio retransmissions, or a complete bearer error history.
Those remain unqualified extensions, not values to synthesize from interface statistics.

A bounded thermal/hwmon name inventory found only `cpu-thermal` and `pwmfan`.
It did not expose a modem sensor in those standard classes. [S4]

## LTE/NR attribution and decoding

### Stronger evidence for the generic signal

Two independently read comparisons matched the generic RAM RSRP, RSRQ, and RSSI to the cached LTE row.
The generic SINR also matched the manual's LTE conversion.
The simultaneous NR row's RSRP, RSRQ, and SINR did not match the generic fields. [S1, S2, S6]

This supports LTE-anchor attribution **for those observations**.
It does not prove an invariant inside the compiled `get_sim_signal` backend.
The RAM timestamp and cached command response are not a documented atomic snapshot.
Keep the generic stream unattributed until the backend contract or further mode-specific evidence establishes its selection rule.
Do not publish the same generic measurements under both LTE and NR labels.

### Format contract

The observed NSA cache had three QENG rows:

- A two-field `servingcell` header.
- An 18-field LTE row.
- An 11-field NR5G-NSA row.

Counts include the first row-type token and exclude the `+QENG:` prefix.
The following positions are **zero-based** and describe schema, not captured identifiers: [S2, S6]

| Layout | Non-identifier field positions |
| --- | --- |
| NSA LTE row | `0=LTE`, `1=duplex`, `7=band`, `8=UL bandwidth`, `9=DL bandwidth`, `11=RSRP`, `12=RSRQ`, `13=RSSI`, `14=SINR`, `15=CQI`, `16=TX power`, `17=srxlev` |
| NSA NR row | `0=NR5G-NSA`, `4=RSRP`, `5=SINR`, `6=RSRQ`, `8=band`, `9=NR DL bandwidth`, `10=SCS` |

LTE-only and NR5G-SA responses include different leading fields.
A single fixed-offset parser is unsafe across these layouts.
Future parsing must validate the RAT, row shape, numeric range, and invalid markers before normalization.

The Quectel manual defines LTE QENG SINR as `Y = (1/5) × X × 10 - 20`, with `Y` in dB.
Its NR SINR interpretation is different.
LTE bandwidth codes and NR bandwidth codes also use different tables.
The documented NR table includes codes beyond those handled by the inspected 3ginfo-lite parser.
`-` means invalid under the current condition. [S6, S7]

The observed header was idle while an NR row remained in the cache.
Therefore, a present NR row is not proof that an NR bearer is active now.
Do not interpret an absent NR row as a permanent lack of 5G capability either.

## Freshness contract

### RAM signal buffer

`/usr/bin/modem_manager.lua` periodically calls `modem.get_sim_signal` and adds `os.time()` as `timestamp`.
It inserts records at the front of its in-memory history.
`get_signals` reads that history without calling the modem backend again. [S1]

Bounded reads showed advancing timestamps.
The two attribution checks found collector ages of one and three seconds.
These observations prove that the collector loop advanced, not that the underlying modem answer was equally recent.
The earlier configuration inspection found a ten-second capture interval; it is a configuration observation, not a universal constant.

Treat `timestamp` as **vendor collector wall-clock time**, not original measurement time.
Clock changes, future timestamps, an empty buffer, missing fields, and an unavailable ubus object need explicit unavailable/error handling.
A successful call does not make every field valid.

### Existing AT cache

The observed cache held 13 entries.
Entries had only `cmd` and `result`; registration, COPS, and QENG values had no individual timestamp.
The inspected inventory did not contain QCAINFO, QTEMP, or a bearer/session statistics response. [S2]

File modification time cannot establish the freshness of every entry.
A changing QENG result can show that a response changed between reads; it cannot timestamp unchanged fields or other commands.
A constant value can be fresh or stale.
The RAM collector timestamp cannot be transferred to the cache merely because values happen to match.

**Recommendation:** treat cache-derived readings as freshness-unknown until the producer's update, failure, and invalidation behavior is proven.
If later exposed experimentally, keep them separate from freshness-qualified metrics and disclose the unknown age.
A normal Prometheus scrape timestamp must not masquerade as modem measurement time.
Do not manufacture a last-success timestamp from a successful file read.

### Kernel and netifd

Counter values are read from the current kernel interface, without a vendor sample timestamp.
Drivers determine the underlying statistics behavior. [S9]
Device removal, counter resets, and interface recreation need reset-aware handling.

Multiple netifd interfaces in the observation referred to the same cellular device.
Exporting its counters once per logical interface would double-count traffic.
Interface uptime is computed from the interface start time, not from modem registration or a carrier bearer start. [S4, S10]

## Native GL HTTP RPC: wider schema, unresolved safety

The installed Internet UI polls `modem.get_status` and `modem.get_cells_info`.
It reads `modem.get_info` separately.
The cell method is requested with a modem bus selector and has a `cells` collection in its consumer. [S3]

The modem-management UI separates `type=servingcell` entries and sorts LTE entries using `mode`.
The Internet details UI recognizes `band`, `ul_bandwidth`, `dl_bandwidth`, `RF`, quality fields, and `id`.
It also contains schemas for legacy RAT measurements such as RSCP and Ec/Io.
These are **consumer schemas**, not proof that this RM520N supplies every field.
`RF` and `id` need backend meaning before normalization.

Keep these distinctions explicit:

- Serving-cell details are not a proven full primary/secondary component-carrier list.
- Tower scan fields belong to an active scan workflow, not automatically to passive cell status.
- Another UI component uses `cells_info` for SIMO; that is not evidence for the RM520N modem path.
- A router-status `mcu.temperature` field is not modem-temperature evidence.
- The source spelling `verdor` is literal; do not silently repair a presumed backend field.

The backend is `/usr/lib/oui-httpd/rpc/modem.so`.
Method names, binary strings, and routine UI use do not establish read-only execution or cache freshness.
No automatic HTTP RPC fallback is recommended until its exact call path is qualified.
The available root/local-trust path also does not prove a restricted telemetry permission model. [S1, S3]

## Stock-OpenWrt paths and alternatives

### Existing ModemManager ownership

The official OpenWrt Lua exporter collector reads `mmcli -L -J`, modem JSON, and `--signal-get -J`.
It exports generic signal quality and numeric fields grouped by signal technology.
It does not implement the complete requested registration, cell, traffic, temperature, and session inventory. [S8]

ModemManager's primary API documents separate LTE and NR5G signal dictionaries.
Extended-signal updates can be enabled by polling or reporting thresholds; a zero polling rate disables periodic polling.
Reading a dictionary alone is not proof that updates are enabled or recent. [S11]

The 3GPP interface exposes registration state and operator code/name.
The bearer interface documents optional connection duration, start date, RX/TX bytes, and accumulated statistics.
Disconnected bearers can retain cached statistics. Availability depends on backend and version. [S11]

This supports a reusable source adapter when ModemManager already owns the modem.
Do not install or start it alongside GL's manager merely for monitoring.
No stock-OpenWrt device was live-tested in this research.

### Existing QMI/MBIM ownership

Pinned upstream `uqmi` source supports technology-specific signal output, including LTE and 5G NR.
For EN-DC, it can return an array with both technologies rather than one flat signal object.
It checks disconnected NR sentinels and exposes serving-system/registration information. [S12]

This is code evidence, not proof that the vendor's older package has that behavior.
QMI/MBIM device access still needs ownership and coordination qualification.
The presence of a query command does not authorize another device client on the reference modem.
Prefer an existing owner's structured status when available; do not make generic OpenWrt support depend on GL RPC.

### 3ginfo-lite

The project contains an RM520N-GL parser and can report bands, selected secondary-carrier details, signal, temperature, slot, and modem metadata.
That does not make it a safe dependency on this router. [S7]

Static findings at the pinned revision:

- The entry path sends `COPS=3,...`, `CREG=2`, and `CEREG=2` commands.
- The RM520N path can send `C5GREG=2` and issues direct modem commands, including QTEMP and QCAINFO.
- Reporting/format changes are writes even when followed by status queries.
- The project warns against ModemManager coexistence. Absence of ModemManager does not prove GL coexistence.
- JSON goes to stdout, but helpers can maintain temporary files. This is not a fileless producer guarantee.
- Generic quality fields are not a complete separate LTE/NR quality schema. Some secondary-carrier quality output is commented out.
- The parser can derive a display RSSI for SA; that must not become a measured NR RSSI metric.
- Its temperature selection and formatted output need sensor/unit validation rather than direct numeric forwarding.
- A source revision does not prove the contents of an older release package.

Do not execute or install it for this reference path.
Its parsing code is useful comparative evidence, not a safety qualification or license to copy code without license review.

### modemdata

`obsy/modemdata` is a standalone structured producer, not only a UI package.
It offers serial, ModemManager, and QMI/MBIM entry points. [S13]

The inspected serial path sends reporting/format-setting commands.
The ModemManager path runs `--signal-setup=3` and can add proxied QMI requests.
The QMI path invokes `uqmi`, and its network helper reads netifd status.
Thus, these helpers need separate side-effect and ownership reviews; none is automatically a passive GL adapter.

Retain modemdata and ModemManager as alternative source families for reusable OpenWrt support.
Neither resolves the reference device's current full-coverage gaps without further qualification.

## Gaps, blockers, and next decision gates

### Blocks a claim of complete, current reference telemetry

1. **Distinct radio freshness:** cache LTE/NR fields have no per-response measurement or success timestamp.
2. **Complete active carriers:** no safe, current, complete aggregation inventory was proven.
3. **Modem temperature:** no passive live modem-temperature source was qualified.
4. **Modem session semantics:** interface uptime exists, but modem bearer/session timing and reconnect history remain unproven.
5. **Wider native status safety:** the GL HTTP RPC backend remains opaque for the required read-only proof.

These block a **full-coverage claim**, not all exporter implementation.
There is enough evidence to plan generic signal, selected-slot, source-health, interface-state, and traffic collection.
That narrower implementation must not silently replace the full product requirement.

### Known limitations, not demands for more hardware

- SA, SIM failover, disconnection, and additional modem models were not exercised.
- Native cache rows cannot provide NR RSSI or NR UL bandwidth when the source format lacks them.
- Sensitive identifiers are deliberately excluded, not missing telemetry work.
- Stock OpenWrt support remains source-level research until that adapter is validated.
- Root was used for qualified live reads; least-privilege ubus/file access still needs package-level validation.

No second physical router, firmware replacement, active scan, forced radio-mode change, or direct AT probe is required to accept this research result.

### Recommended follow-up

Resolve the reference-specific backend/cache contract through vendor source, vendor documentation, or static analysis before enabling wider collection.
Keep source timestamp, read time, validity, and freshness status separate in the eventual schema.
Maintain independent capability flags for generic signal, LTE, NR, carriers, temperature, and session data.
Unsupported, stale, invalid, and failed must not collapse into numeric zero.

Any later live call outside the proven passive sources needs a new safety qualification.
The package should fail closed rather than use direct AT commands automatically.

## Primary evidence references

Installed firmware paths below identify inspectable primary artifacts.
They are not links to private captures, and this note does not redistribute proprietary binaries or raw telemetry.

- **[S1] Installed signal producer:** `/usr/bin/modem_manager.lua`, especially `Signal:collects`, `Signal:get_signals`, and the `modem.signal` ubus registration; `/etc/init.d/modem`. Live qualified reader: `ubus call modem.signal get_signals '{"time":0}'`. The package/version is recorded under Scope.
- **[S2] Existing cache:** `/var/run/modem/at_cache.json`, schema `AT_CACHE[] -> {cmd, result}`; `/etc/init.d/at_cache_init`; `/usr/bin/at_cache`. Evidence is limited to sanitized structure, selected field validity, and in-memory comparisons.
- **[S3] Installed UI/backend:** `/www/views/gl-sdk4-ui-internet.common.js.gz`, `/www/views/gl-sdk4-ui-modemmanage.common.js.gz`, `/usr/share/gl-ngx/oui-rpc.lua`, `/usr/lib/lua/oui/rpc.lua`, and `/usr/lib/oui-httpd/rpc/modem.so`. UI consumers are not backend safety proofs.
- **[S4] Live host state:** `/sys/class/net/rmnet_mhi0/statistics/{rx_bytes,tx_bytes,rx_packets,tx_packets,rx_errors,tx_errors,rx_dropped,tx_dropped}` and the device-matched subset of netifd interface status. Thermal inventory used `/sys/class/thermal/thermal_zone*/type` and `/sys/class/hwmon/hwmon*/name`. The interface device name identifies this observation, not a package-wide constant.
- **[S5] Installed traffic implementation:** `/usr/bin/traffic_statistics`; source contains temporary and optional persistent traffic files, configuration writes, and network reload paths. Inspected, not executed.
- **[S6] Quectel primary manual, distributor-hosted:** [RG520N/RG525F/RG5x0F/RM5x0N/RM521F Series AT Commands Manual, V1.0, 2024-02-07](https://mm.digikey.com/Volume0/opasdata/d220001/medias/docus/6382/Quectel_RG520NRG525FRG5x0FRM5x0NRM521F_Series_AT_Commands_Manual_V1.0.pdf), especially section 5.20, printed pages 103–108. Command names describe cached formats; no commands were sent.
- **[S7] 3ginfo-lite pinned source:** [revision b976ecb9d1b6a1683ccf672ef698a9526bdea390](https://github.com/4IceG/luci-app-3ginfo-lite/tree/b976ecb9d1b6a1683ccf672ef698a9526bdea390), `README.md`, `luci-app-3ginfo-lite/root/usr/share/3ginfo-lite/3ginfo.sh`, `info.gcom`, and `modem/usb/2c7c0801`.
- **[S8] Official OpenWrt ModemManager collector:** [modemmanager.lua at 70ac3a84406cab03865c55be84de960af815436a](https://github.com/openwrt/packages/blob/70ac3a84406cab03865c55be84de960af815436a/utils/prometheus-node-exporter-lua/files/usr/lib/lua/prometheus-collectors/modemmanager.lua).
- **[S9] Linux kernel:** [Interface statistics](https://docs.kernel.org/networking/statistics.html), especially standard interface counters and sysfs access.
- **[S10] netifd primary source:** [ubus.c at 2fec6e82409b029c36be6eb3c07938d77c52263a](https://github.com/openwrt/netifd/blob/2fec6e82409b029c36be6eb3c07938d77c52263a/ubus.c), interface status and `uptime` calculation. Upstream semantics corroborate live fields; this is not a claim of binary identity with the vendor build.
- **[S11] ModemManager primary API:** [Signal](https://www.freedesktop.org/software/ModemManager/api/latest/gdbus-org.freedesktop.ModemManager1.Modem.Signal.html), [3GPP](https://www.freedesktop.org/software/ModemManager/api/latest/gdbus-org.freedesktop.ModemManager1.Modem.Modem3gpp.html), and [Bearer](https://www.freedesktop.org/software/ModemManager/api/latest/gdbus-org.freedesktop.ModemManager1.Bearer.html). These describe current documented APIs, not the reference device's installed capabilities.
- **[S12] uqmi primary source:** [commands-nas.c at 7914da43cddaaf6cfba116260c81e6e9adffd5ab](https://github.com/openwrt/uqmi/blob/7914da43cddaaf6cfba116260c81e6e9adffd5ab/uqmi/commands-nas.c), signal and serving-system handlers.
- **[S13] modemdata pinned source:** [revision fbdc0b6a720647642e99c6fc8db933a767c9ead0](https://github.com/obsy/modemdata/tree/fbdc0b6a720647642e99c6fc8db933a767c9ead0), `README.md` and `modemdata/files/usr/share/modemdata/{params,params_modemmanager,params_qmi,network}.sh`.
