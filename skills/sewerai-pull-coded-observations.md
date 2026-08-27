---
generated: '2026-08-27'
method: generated
name: Pull NASSCO-coded observations for a program
description: Read coded inspection results out of SewerAI PIONEER into your own system — list inspections, pull their PACP/MACP/LACP observations and ratings, and fetch video playback URLs — without pagination, without rate-limit signal, and without over-fetching.
api: openapi/sewerai-swagger.json
operations: [v1_inspections_list, v1_inspections_read, v1_inspections_observations_list, v1_observations_list, v1_observations_read, v1_videos_list, v1_videos_url, v1_assets_list]
source: >-
  Grounded in openapi/sewerai-swagger.json, harvested verbatim 2026-08-27 from
  https://api.sewerai.com/swagger.json. Every operationId verified verbatim in that contract. Field
  semantics per data-model/sewerai-data-model.yml and the schema reference at
  https://docs.sewerai.com/#schemas. NASSCO field mapping per conformance/sewerai-conformance.yml.
---

# Pull NASSCO-coded observations for a program

The read side of SewerAI: getting inspection results out of PIONEER and into a GIS, CMMS, capital-planning model or report. This is the flow behind SewerAI's own Cityworks, ArcGIS and Cartegraph integrations.

## Auth

- Base URL: `https://api.sewerai.com/v1`
- `Authorization: X-SAI {API_KEY}` or `Authorization: Bearer {JWT}`. See `authentication/sewerai-authentication.yml`.

## Steps

1. **Scope the pull with filters, because you cannot page.** `v1_inspections_list` (`GET /v1/inspections/`) accepts date-window and type filters. Use `created_after` / `created_before` / `updated_after` / `updated_before` to slice the program into windows small enough to return in one response. There is no `limit`, `offset`, `page` or `cursor`.
2. **Read the inspection header** — `v1_inspections_read` (`GET /v1/inspections/{sid}/`). This is the widest object in the API: the PACP/LACP/MACP header record with pipe geometry, manhole component condition, coordinates, survey and reviewer certification numbers, the ten NASSCO custom fields (`Custom_Fields` with `Custom_Labels`), and the `ratings` block.
3. **Read the ratings** — `ratings` on the inspection is the NASSCO rating block: `ST1`–`ST5` and `OM1`–`OM5` grade counts, `STGradeScore1`–`5` / `OMGradeScore1`–`5`, `PACPQuickRating`, `MACPQuickRating`, `OverallQuickRating`, and the likelihood-of-failure indices `LoFPACP` / `LoFMACP`. If you already consume PACP, this maps straight across with no bespoke translation.
4. **Read the observations** — `v1_inspections_observations_list` (`GET /v1/inspections/{inspection_sid}/observations/`) for one inspection, or `v1_observations_list` (`GET /v1/observations/`) across inspections. Each observation is a PACP defect record: `code`, `Distance` (chainage), `Grade`, `Continuous`, `Joint`, `Clock_At_From` / `Clock_To`, `Value_1st_Dimension`, `Value_2nd_Dimension`, `Value_Percent`, `Remarks`, plus `video_frame`, `snapshot_url` and `bounding_boxes` linking the code back to the frame the AI found it in.
5. **Resolve the asset** — the inspection's `asset` field is an absolute URL. Take the trailing path segment for the `sid` and call `v1_assets_list` or read the asset directly for `geojson`, `shape`, `host_material`, `length`, `rim_to_invert` and the rest of the physical record.
6. **Get video playback** — `v1_videos_url` (`GET /v1/videos/{sid}/url/`) returns a playback URL for the footage behind an observation. Treat it as short-lived; nothing in the contract states its lifetime.

## Rules an agent must follow

- **Slice by time, never by page.** With no pagination parameters, an unbounded `GET /v1/observations/` over a large program will return everything in one array. Drive the pull off `updated_after` / `updated_before` windows and keep a high-water mark; that is also how you do incremental sync.
- **Pace conservatively.** No published rate limit, no `RateLimit-*` headers, no documented 429 behaviour. Serialize the pull and leave gaps between requests rather than fanning out. See `rate-limits/sewerai-rate-limits.yml`.
- **Deleted rows are still returned as rows.** `deleted` and `deleted_by` are read-only fields on assets, inspections and observations. A non-null `deleted` means the record is soft-deleted — filter it out yourself; the API does not do it for you.
- **`autocode_complete` is the readiness flag.** An inspection whose `autocode_complete` is false has not finished AI coding. Its observation list may be empty or partial. Check the flag before treating an empty list as "no defects found".
- **Unreviewed AutoCode observations may not count.** As of the 2026-07-14 release, unreviewed AutoCode observations are excluded from NASSCO ratings. An observation list and a rating block can therefore disagree, legitimately. See `changelog/sewerai-changelog.yml`.
- **Units are not implicit.** The inspection carries `IsImperial`. Metric conversion on the wire is a v2 feature announced on 2026-07-14; on v1 you convert yourself. Read `IsImperial` before doing arithmetic on any dimension.
- **Errors are three shapes and 403 means unauthenticated.** See `errors/sewerai-problem-types.yml`. Capture `x-amzn-requestid` for support.
- **This is read-only and therefore safe.** Every operation in this skill is a `GET`; idempotency and reversibility do not apply. Keep it that way — do not mix write calls into a sync job.
