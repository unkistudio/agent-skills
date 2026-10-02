---
name: ekahau-review
description: Review an Ekahau WiFi site survey (.esx) and produce zone-by-zone verdicts, coverage maps and fix recommendations with a final report. Use this skill whenever the user mentions an Ekahau survey, an .esx file, WiFi coverage review, RTLS or location tracking validation, voice or data coverage check, channel/interference audit, AP placement review, or asks what to fix on their WiFi — even if they don't say the word Ekahau.
---

# Ekahau Survey Review

Turn an Ekahau `.esx` survey file into per-zone verdicts, Ekahau-style maps and a client-ready report with concrete WiFi fixes.

An `.esx` file is a zip archive. The JSON documents inside link to each other by ID. Read them — never guess. Key files:

| File | Gives you |
|---|---|
| `areas.json` | Zone polygons (the review scope) |
| `floorPlans.json` | Floors, scale (`metersPerUnit`), plan image IDs |
| `accessPoints.json` | Every AP heard, its position if placed, `mine` flag |
| `measuredRadios.json` | Links APs to their measurements |
| `accessPointMeasurements.json` | Per-radio SSID, channels, bands — the ground truth |
| `survey-*.json` | `routePoints` = real walk paths |
| `requirements.json` | Requirement profiles assigned to zones |
| `image-*` | Floor plan backgrounds |
| `track-*.bin` | Raw SUN samples, proprietary binary — do not attempt to decode |

## Phase 0 — Ask first, touch nothing

Ask these questions BEFORE reading the file. Never assume the answers. Use the question tool with multi-select where marked.

1. **Usage (single):** location tracking (RTLS), voice, data coverage, capacity, or general health check?
2. **Bands in scope (multi):** 2.4 GHz, 5 GHz, 6 GHz? (Tracking tags usually live on one band only; laptops and phones need the others.)
3. **Own network:** which SSIDs count as ours? Treat multi-SSID setups as one physical network — list every variant.
4. **Zones in scope (multi):** read the zone names from `areas.json` first, then have the user confirm which ones to judge. Everything outside the confirmed zones is out of scope and stays out of the report.
5. **Thresholds:** propose defaults from the usage and let the user adjust —
   - RTLS: primary signal at least -62 dBm, at least 3 APs at -75 dBm or better (target -72 dBm).
   - Voice: primary at least -65 to -67 dBm, clean overlap, 20 MHz channels on 2.4 GHz.
   - Data: agree explicit values with the user.
6. **Report language, audience and deliverables:** language (always ask), tech vs mixed vs management tone, markdown only or markdown plus styled HTML and PDF, logo or branding files if any.

Record the answers. They drive every later phase — bands filter every computation, zones bound every map and verdict, thresholds define pass and fail.

## Phase 1 — Parse (read-only)

1. Open the `.esx` as a zip and load the JSON graph above.
2. Build the network inventory on the **confirmed SSIDs**, not on the file's `mine` flag (it misclassifies neighbours). Count, per zone and per band: placed APs with positions, and heard-but-unplaced APs. Report both counts; only positioned APs go on maps.
3. Per positioned AP, resolve its measured channels from the radio→measurement links. Expect clean plans (1/6/11 on 2.4 GHz); flag anything else.
4. Note the requirement profile assigned to each zone and whether it matches the confirmed bands — a profile checking bands the clients never use is itself a finding.

## Phase 2 — Analyse

Work only inside confirmed zones, only on confirmed bands.

1. **Walls.** Extract wall geometry from the plan images by colour threshold and inspect the mask visually before using it. No wall data, no wall-aware claims.
2. **Propagation.** Predict per-AP signal with a simple calibrated model: transmit power, distance exponent, and a per-wall-crossing loss counted as wall transitions (one per wall, not one per drawn line — CAD double lines must not count double). Dilate the mask slightly and sample densely along each ray so thin walls are never skipped.
3. **Calibrate.** Tune the loss parameters against one Ekahau screenshot of a known zone supplied by the user (mostly passing with small grey patches is the typical anchor). One zone is enough — then freeze the parameters for all zones. If no screenshot is available, say so and mark verdicts as uncalibrated.
4. **Judge.** Per zone: share of area above the primary threshold, share hearing the required AP count at the secondary threshold, same-channel overlap. Grey means below the primary threshold only, so maps stay comparable with Ekahau's signal view.
5. **Propose (only if coverage work is needed).** Place new APs where the required AP count fails, spread apart (farthest-from-existing-AP selection, inset from zone borders), each on the least-used channel. Recompute the metrics with them included and report before/after shares. Describe each proposed AP by nearest existing AP, direction and channel — never by raw coordinates alone.

## Phase 3 — Maps

One signal map and one overlap map per zone, cropped to the zone, uniform display aspects across zones:

- Heat colours red (weak) to green (strong) with contour lines, legend showing the pass threshold, failing areas greyed out.
- Existing APs as small round dots with dark outline and a name-plus-channel label, labels alternated on both sides so dense corridors stay readable, always drawn above walk paths.
- Proposed APs as slightly larger dots in a contrasting colour with sequence labels.
- Real walk paths drawn thin underneath the AP layer.
- Zone outline dashed. Figure titles naming the zone and view.

## Phase 4 — Report

One section per zone: verdict in one plain sentence, then an action list with concrete items (which AP to add/move, where relative to named APs, which channel). Then a prioritised cross-zone action table, the thresholds used, a short conclusion, and a 6-line glossary for non-technical readers.

Writing rules: short sentences, plain words on first use, every number traceable to the file. Never discuss out-of-scope zones or bands, never dump file internals or methodology, never advise blanket power cuts (power comes down only after adds, only where overlap is proven excessive, only while checked areas stay above threshold), never present modelled values as measured.

For the styled HTML and PDF deliverables, reuse the report-publishing workflow (single markdown source of truth, embedded figures, verified page count and image count).

## Guardrails

- Questions before parsing, always. No defaults silently applied.
- Scope discipline: unconfirmed zones, bands and SSIDs do not appear in maps, verdicts or counts.
- Honesty: modelled coverage is labelled as such by its limits (no decoded samples, uniform wall loss); the final word belongs to the tool's own measured views.
- If the file contradicts an assumption (misplaced APs, unexpected SSIDs, missing bands), stop and ask instead of working around it.
