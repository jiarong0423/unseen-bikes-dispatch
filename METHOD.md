# Method

This document writes the approach behind the slides as a readable specification. All numbers are **algorithmic replay estimates** from official public station data, not operating results. The citywide off-dock bike count is an **assumed maximum (a scenario value) derived from official data**; the only direct counts are two visits to one station. The document discloses method and results; it does not contain code, raw snapshots, keys or the packet format.

## 1. The problem

When a station has no free dock, the bike is parked outside the docks and staff are called; billing is extended meanwhile. Official station data records only docked bikes, so off-dock bikes do not exist in the data and the data can disagree with the field.

A field example (one station, one moment, counted by eye; not a distribution and not a city average): a station in Taoyuan on 2026-09-17, 25 docks, with another 20 bikes standing outside the rails (photographed). At 17:15 the official feed showed 2 bikes available while about 22 were on site; on 2026-09-22 the author again reported about 20 outside while the docks were nearly full. It shows that the phenomenon exists, can be counted, and cannot be expressed by the official station fields. These two visits are the only direct observations; the station is outside the New Taipei data scope, and the full-resolution originals are not published.

## 2. Inputs

- Official public live station data (bikes available, docks free), plus the author's own daily snapshots.
- A station-to-circle mapping (2 km shared-reservoir circles: 72 main circles plus island groups, 96 in total).
- Truck parameters from public reports: New Taipei dispatch fleet of 41 trucks, about 14 bikes per truck (the operator's statement).
- No rider identity and no ride tracks are used.

## 3. Algorithm

1. **Each station's target level is its own usual level**, not a city-wide ratio. "Usual" comes from a base table: the median docked level per station, per window and per day type (holidays follow the official calendar); special-event days and typhoon-closure days are excluded.
2. **The unit is a 2 km shared-reservoir circle**: a station belongs to a circle if it is within 2,000 m of the circle centre. Circles may overlap (shared stations are the channels through which bikes move between circles).
3. **Four evaluation windows**: 05:30, 10:00, 15:30, 23:00. 05:30 and 15:30 are completion deadlines for trucks; the one-button batch applies a 120-minute deadline to every window (for example the 23:00 window needs the latest departure at 21:00).
4. **Shortage** = target total level minus current docked level. The estimated off-dock (reserve) bikes are then counted against it: **remaining shortage = shortage − truck moves − reserve**. The off-dock count uses a scenario value of 4,262 bikes: the gap between the median docked count at 03:00–04:00 (17,628) and an estimated fleet of 21,890 (peak simultaneous docked 19,450 plus an assumed 1,830–3,660 bikes on the road). It is a pool of all non-docked bikes, so it may include bikes that are rented, broken, in repair or in trucks, not only bikes parked beside full stations. As a cross-check, the official end-2025 fleet of 23,479 bikes minus about 17,500 docked at 03:00 leaves a pool of about 5,950 non-docked bikes of every kind (the official figure may not be current), so 4,262 is the same order of magnitude. See the sensitivity table in section 5.
5. **Trucks only handle what can be moved directly within 2 km**; the rest relies on each circle's reserve bikes being re-docked (motorbike staff re-dock bikes: layer change, total unchanged).
6. **Forward-route rules**: at most 14 bikes per truck; each leg at most 2 km; no backtracking (all legs inside one 120° sector).
7. **Number of trucks**: trucks leave from depots (three approximate Taipei-side locations: Wanhua, Neihu, Gongguan/Da'an; coordinates are landmark approximations, not street addresses), at 40 km/h, detour factor 1.3, no stop time; the minimum number of trucks needed to finish within the deadline is reported under three loading-time assumptions (optimistic / middle / conservative).
8. The computation is rerun automatically every 30 minutes.

## 4. The edge device (the off-dock layer)

- **Bike unit**: the concept is a unit mounted on the bike (the official YouBike 2.0 unit is also on the bike, the bike itself is the signal source). On the bench: a development board plus a button module. The rider presses the button to trigger a temporary return; there is also a periodic heartbeat.
- **Wireless packet**: short Bluetooth advertisement with an authentication code; keys and packet format are not published.
- **Virtual dock (gateway)**: near the station, pairs with the bike unit, with de-duplication, cooldown and a freshness window, and records one temporary return in this study's own log. **Nothing is sent into any operator system and no operator record is changed.**
- **A temporary return is not an official return. Billing does not stop. Do not leave a bike outside the docks because of it. Reports are unverified signals and never drive dispatch on their own.**
- **Report fields**: which station, how many, when, who reported (a reporter code). The code does not correspond to a name; no rider identity or bike number is collected; retention and deletion rules for the code are not yet defined.
- **Roles**: the rider pairs with the virtual dock over Bluetooth and makes the temporary return; the physical lock and payment stay with the operator; nothing here unlocks, controls a lock or changes billing. Field work by peak-hour motorbike staff is outside this proposal.
- **Verified so far**: bike unit → advertisement → gateway (bench, short range); gateway → console (local event drill). **Not done**: uploading over the network, multiple boards, pairing with real bikes, comparison with operator data. **Bench only; not to be installed on operator vehicles without authorization.**

## 5. Results and sensitivity (replay estimates)

Data: 280 clean slots from 2026-07-21 to 10-15, main 72-circle basis.

| Assumed off-dock bikes (citywide; 4,262 = assumed maximum) | 0 | 1,000 | 2,000 | 4,262 | 6,000 | 8,000 |
|---|---:|---:|---:|---:|---:|---:|
| Average shortage remaining, all slots (bikes) | 314.9 | 129.5 | 46.5 | 4.75 | 0.93 | 0.16 |
| Weekday 15:30 | 522.7 | 218.3 | 76.0 | 9.11 | 2.28 | 0.34 |
| Weekday 10:00 | 146.9 | 62.1 | 29.0 | 3.72 | 0.76 | 0.28 |

Trucks versus completion deadline (96 circles citywide replay; 40 km/h, detour 1.3, no stop time):

| Deadline | 60 min | 120 min | 180 min |
|---|---:|---:|---:|
| Trucks per slot | 7.9 | 3.7 | 2.6 |

Others: weekday remaining shortage 0.6–3.6 bikes (weekday slots of the 10 out-of-sample days, 2026-10-06 to 10-15, main 72 circles); trucks directly resolve about 8%–26% of the shortage; the National Day holiday at 15:30 has the largest shortage (per-slot average over the three days, 96-circle basis: shortage 2,628 bikes, 253 remaining, under the 4,262 scenario). Mid-Autumn 15:30 (four days) averages 493 and 9.8.

## 6. Limitations (stated up front)

- "Remaining shortage" in this repo is what the Traditional Chinese console UI calls "真實缺車": the same quantity.
- "Shortage" here means: the circles' target total level minus the current docked level (reserve bikes are counted afterwards). Denominators differ between pages (72 vs 96 circles; one batch vs a 280-slot replay), so figures are not directly comparable. "Trucks directly resolve 8%–26%" is a replay average; a single batch (for example the one-button batch) can differ.
- A longer deadline reduces the number of trucks running at the same time, but total workload is about the same (about 7.4–7.9 truck-hours), so it is not a saving in labour.
- "Weekday remaining shortage 0.6–3.6" is from the weekday slots of the 10 out-of-sample days (2026-10-06 to 10-15); the sensitivity table in section 5 (weekday 15:30: 9.11, 10:00: 3.72) is the 280-slot average. Sample and basis differ; do not compare directly.
- "Trucks directly resolve 8%–26%" is the range of averages over the 280-slot replay; both single batches published here (2026-10-21 18:59, about 54%, and 22:25, about 49%, both for the 23:00 window) are above this range, for reasons not yet investigated (denominator, window, day type). This replay-versus-live gap is an open validation issue.
- "Remaining shortage" is the value under the estimated reserve (4,262): with fewer reserve bikes the shortage is larger, with more it is smaller. It is a lower bound only if no more than 4,262 bikes are really off-dock, which the data do not establish.
- The truck count (3.7 per slot) was computed only for the 4,262 assumption, on the 96-circle basis, with no stop time and straight-line distances; it was not recomputed for other off-dock counts. The sensitivity table's 6,000 and 8,000 columns are stress tests beyond the stated bound.
- "Remaining shortage" measures distance from each station's usual level, not service outcome: a value near zero does not by itself mean fewer empty or full stations.
- Details not published: how the minimum truck count is solved, the weights used to spread reserve bikes over circles, the criteria for clean days.
- There is no citywide measurement of off-dock bikes; the field count is one station at one moment. Results are highly sensitive to this assumption, which is exactly the value of edge reporting.
- The replay has no counterfactual and no field validation.
- Distances are straight lines between circle centres, with no real road network and no stop time.
- Patent: the author has a separate Taiwanese patent application whose subject differs from this method; this document grants no patent rights.
- Source terms are reviewed per source; this repo does not redistribute raw snapshots and contains no code.

## 7. What this repo does not contain

- Raw station snapshots and per-station data (licences not checked item by item).
- Code, keys, concrete communication parameters.
- Contact phone numbers, rider identities or bike numbers.

## 8. Licence and trademarks

Documents, slides, diagrams and console screenshots in this repo are provided under CC BY-NC-ND 4.0 (see LICENSE and NOTICE.md). The slides use no third-party photo or logo: the bike unit is the author's own sketch and the bench photo is the author's own. Names such as YouBike belong to their owners; this is independent research with no affiliation or endorsement.

Contact: Sun Jia-Rong (孫嘉嶸), jiarong0423@gmail.com
