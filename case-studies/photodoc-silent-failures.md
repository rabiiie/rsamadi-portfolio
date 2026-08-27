# Photo Documentation — OCR, Geocoding and Watermark Removal

> Sanitized: no proprietary code, credentials, client names or site locations. Measurements are real; places are generalized.

## Scope

Field crews photograph fibre installations. Each photo carries a stamped label — company, town, node, coordinates, address. The pipeline reads the label, verifies the location, writes it into the photo's EXIF, erases the stamp, and produces a documented set delivered to the client.

- Microservice: Python/FastAPI on an on-premise Windows Server, one module instance per pool thread.
- Cloud calls: OCR, geocoding, field extraction and inpainting. Database and files remain on premise.
- Consumers: Spring Boot backend, React UI.

The deliverable is the photograph itself. A coordinate that is wrong by 140 km and a correct one are indistinguishable in a file listing, and there is no downstream stage that rejects either. Every validation has to occur at the point of production.

---

## 1. OCR losing the last two lines of the label

**Problem.** On 3060×4080 photos, `DetectDocumentText` returned the upper label lines — company, town, node — and consistently omitted the final two: coordinates and address. Read rate on one 27-photo sample: 1/27.

**Root cause.** Relative resolution, not engine accuracy. At full-frame scale those lines occupy too few pixels.

**Fix.** Crop the lower band and submit it as a second call, merging the resulting lines into the full text (`ocr_text` in `aws_ocr.py`). Read rate on the same sample: 27/27.

Four constraints established by measurement while implementing it:

**Crop area governs the read, not the margin.** Measured on one photo: a 2017×1344 crop returned `748`; the same crop at 2017×1018 returned `50.00748`. Above approximately 1.5 Mpx the service downscales the image and resumes clipping the first character of each line. `_regiones_del_rotulo` computes band height against that budget.

**The crop requires its own margin.** `ImageOps.expand`, 60 px. A glyph touching the image border is clipped: `50.01774` arrived as `0.01774`, which fails the European-latitude plausibility check, and the photo was discarded as unreadable. A line that fails to parse is visible in the logs; a line that parses to a different valid-looking number is not.

**Both halves of the band are submitted.** The label is aligned to one margin, and which margin depends on the camera app. Inferring the side from the centre of mass of the first pass fails: printed text elsewhere in the frame (street-box markings) shifts the centre to the opposite side and the crop lands beside the label. Read rate on a second sample: 6/21 with inference, 16/21 with both halves, 21/21 after applying the area budget.

**Merge selects the longest read, not the last.** The right-hand crop captures only the tail of the left-aligned label's lines; its truncated `748` was overwriting the complete `50.00748` from the other crop.

**Related fix.** `extract_coords_and_date` used `DECIMAL_COORD_REGEX.search()` — first match only. OCR concatenates numbers from adjacent lines and forms plausible false pairs (`10.89201, 08.2026`) ahead of the real pair. Changed to `finditer` with a break on the first plausible match.

**Scope of the band.** The second pass runs inside `ocr_with_blocks`, not only over plain text. Its bounding boxes are translated to full-photo coordinates and merged with the first pass. Watermark localization and the erase mask both depend on those boxes.

**Cost.** Two OCR calls per photo instead of one: approximately €1.20 per 400 photos rather than €0.60. Erasing remains one call per photo.

---

## 2. Geocoder returning plausible results for unmatched queries

**Problem.** `geo-places.search_text` always returns a position. Measured against one town's data:

| Query | Response | Error |
|---|---|---|
| Real address | `PlaceType=PointAddress` | correct doorway |
| Real street, invented house number | `PlaceType=Street` | street midpoint, ~860 m |
| Invented street | `PlaceType=Locality` | town centre |
| No match | `BiasPosition` | country centre, ~140 km |

All four pass `is_plausible_lat_lon_for_europe`. A distance guard against the geocoded town centre rejects the bottom two only: a non-existent house number resolves inside the town radius and is accepted.

**Impact.** The coordinate is written to the photo's EXIF and from there into the client's report.

**Fix.** `aws_geocode.geocode_detail()` preserves `PlaceType` and `Address.Label` and translates them into an explicit precision value — doorway, street or town — surfaced through to the operator UI. The position alone is never accepted. The town-distance guard is retained; in one town it rejected 3 of 17 folders.

---

## 3. Per-photo rather than per-folder operator input

**Problem.** Photos without usable coordinates are placed in a folder named `sin_coordenadas`. Porting an operator dialog from a desktop tool to the browser made it convenient to collapse a per-photo question into one answer per folder.

**Root cause of the rejected approach.** Verified against real data: one such folder contained photos from three different streets. The address is in each file name with the node code as a suffix (`<street> <number> - <node>.jpeg`), not in the folder name, which is literally `sin_coordenadas`. Collapsing the dialog would stamp photos of different doorways with one coordinate and raise no error.

**Fix.** Operator input is modelled per photo (`coords_by_file`: path → value). Entries absent from the response are skipped, matching the behaviour of leaving a field blank in the original dialog.

---

## 4. Job state and batch reporting

**Problem A — batch results.** The service layer returned a fixed `"ocr_process completado"` string and `process_folder` did not return its `stats`. A run could complete, report success and have written zero coordinates, indistinguishable from a run that corrected every photo.

**Fix.** Batch operations return counts: examined, corrected, still pending and the reason. The UI renders one row per photo from `invoicing.photo_documentation`, linked to the job by `job_id` (migration V15).

**Problem B — state duplication.** Job state existed in three locations: microservice memory, the `photo_jobs` table, and the browser, synchronized by two loops. Five visible defects originated from that single duplication:

- `Running` displayed alongside `completed`
- correct jobs marked `Failed` on microservice restart ("Job perdido")
- progress fixed at 0 %
- stale `error_message` persisting on the row
- 401 on the SSE stream

**Fix (migration V16).** Single owner: the microservice writes `invoicing.photo_job_state` (status, progress, error) and `invoicing.photo_job_log` (one numbered row per line). Spring reads. `photo_jobs` retains context only — who launched it, which folder, which client/project/scope.

Removed: the `@Scheduled` Spring→FastAPI poll in `PhotoJobSyncService`, and the `/{jobId}/events` SSE relay. The browser polls `GET /{jobId}/log?desde=N` every 2 s and receives only new lines. The persisted log survives closing the tab.

---

## 5. Authorization

**Problem.** The prior check called `projectService.getProjectById(projectId)`, which belongs to a different product line and performs no check when `projectId` is null. In practice there was no authorization.

**Defect found while rebuilding it.** `public.clients` stores `name` and `code` as separate columns. `assertCanAccessCity` received the display name and compared it against the code literal. The comparison never matched, so the check fell through, leaving a `WARN` in the log and no enforcement.

**Fix.** The client code is resolved server-side via `ClientDAO.findCodeById(clientId)` and passed in `JobContext.clientCode`. It is never accepted from the browser.

**Model.** Module `<CLIENT>_PHOTODOC`, `RESOURCE_TYPE = "city"`, role `PHOTO_DOC`. The city travels in `JobContext.project_name` — the context has no `city` field, and for this client the project name is the city. Ownership is by scope, not `created_by`: any user holding the city can resume a review another user left incomplete. Access to all cities is granted by a `client`-mode scope, not by leaving the user without scopes.

**Two related decisions:**

- **File root failed open.** Without `app.photojobs.fs.allowed-roots`, the path logic fell back to the entire network share, which holds another product line's files. The policy now lives in `PhotoJobPathPolicy` and fails closed.
- **Legacy fallback is the permissive direction.** "No scope rows means full access" admits a user with nothing configured, while a user already holding scopes for that client under a different module is correctly refused. The permissive case generates no support ticket.

---

## 6. Inpainting region and mask

**Constraint.** Watermark removal is the highest-priority function and drives region selection. The current model (`us.stability.stable-image-erase-object-v1:0`) exists only in US regions. The European-hosted predecessor is in Legacy status with a fixed EOL date and closed to new customers; selecting it would repeat a failure already present in production, which is a pipeline built on a subsequently retired model. Cropping the tile before transmission does not mitigate the data-residency question: the crop is the region containing the address.

**Decision.** Recorded as a data-residency decision rather than a technical one, with the tradeoff stated — watermarks contain postal address, GPS, altitude and time — taken by the business owner, and carrying an explicit condition for revisiting it if the product is commercialized or opened to additional clients. Remaining services (OCR, geocoding, field extraction) stay in the EU region.

**Operational notes.** The model is invoked by inference profile (`us.` prefix) and does not appear in `list_foundation_models`, only in `list_inference_profiles`. Response shape: `{seeds, finish_reasons, images}`. One call with a mask covering all fragments replaces a per-bbox invocation loop. Approximately 32 s for a 12 MB PNG.

**Mask defect.** Erasing the label's enclosing rectangle covered 55 % of the tile. With that proportion masked the model lost surrounding context and generated replacement background: in one photo a sandstone wall and a set of steps were returned as plain concrete. Masking individual text lines within the bbox reduces coverage to approximately 34 % and the background is preserved. Line boxes rather than word boxes: the label is right-aligned, and word-level masks left line endings unerased. Verified on a 10-photo validation set: 10/10 clean.

---

## Measured ceiling

One town, 40 pending photos: 18 resolved.

The remaining 22 have no coordinates in the image and no GPS in EXIF — 0 of 40 carried EXIF GPS. Four camera-app label formats were encountered; two stamp coordinates, two stamp only a date or a technician name. OCR cannot recover a value that was never written.

For those, the folder name is an address. Geocoding it yields doorway precision rather than the position from which the photo was taken. Implemented and tagged with a distinct `invoicing.gps_source` value, `FOLDER` (migration V10), so it is never merged with a coordinate read from the photo. One geocoding call per folder, not per photo.

Two further format defects fixed in the same pass: `DEC_HEMI_REGEX` failed on one app's `49.877432°N` format due to the degree symbol between number and hemisphere letter and a separator class excluding the decimal point; `DATE_REGEX` accepted only `dd.mm.yyyy`, now also `dd/mm/yyyy`, with `apply_gps_and_datetime` splitting on `[./]`.

## Tech stack

Amazon Textract · Amazon Location Service · Amazon Bedrock (Stability image erase) · Python · FastAPI · Pillow · PostgreSQL · Spring Boot · React · EXIF/GPS metadata
