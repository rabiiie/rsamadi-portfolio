# Photo Documentation — A Pipeline Where Nothing Fails Loudly

> Sanitized write-up — no proprietary code, credentials, client names, or site locations. Measurements are real; places are generalized.

## The short version

Field crews photograph fibre installations. Each photo carries a stamped label — company, town, node, coordinates, address — and the job is to read it, verify the location, write it into the photo's metadata, erase the stamp, and hand the client a documented set.

Every stage of that pipeline **degrades instead of failing**. The OCR returns most of the label. The geocoder returns a plausible point for an address that doesn't exist. The batch job returns green having written nothing. The permission check compared two values that never matched and logged a warning. Not one of them raises an error, and the output of the whole thing is a photograph that *is* the evidence sent to the client.

This is a write-up about building guards for failures that don't announce themselves.

## Why silence is the whole problem here

In most systems a wrong value gets caught downstream: it fails a constraint, breaks a join, shows up as an outlier in a report. Here the artifact is a JPEG in a folder handed to a client as documentation of where a technician stood.

A coordinate that is wrong by 140 km and a coordinate that is right look exactly the same in a file listing. There is no downstream. The guard has to exist at the point of production or it doesn't exist.

---

## 1. The OCR read the label — just not the two lines that mattered

On a 3060×4080 photo, text detection returned the top of the label — company, town, node — and consistently dropped the last two lines: **the coordinates and the address**, the only two anybody needed.

It isn't that the engine reads badly. At that scale those lines are a few pixels tall. It's relative resolution.

The fix is to send the bottom strip as a second call and merge the lines into the full text. That took one sample from **1 of 27 photos read to 27 of 27**. Then three things turned out to matter more than the idea:

**What governs the read is the area of the crop, not the margin.** Measured on one photo: a 2017×1344 crop returned `748`. The same crop at 2017×1018 returned `50.00748`. Past roughly 1.5 megapixels the service downscales the image and starts eating the first character of every line again. The strip height is now computed against that budget rather than picked as a percentage.

**A missing first character is worse than a missing line.** `50.01774` arriving as `0.01774` is not a latitude anywhere in the country — the photo was silently discarded as unreadable. The crop needs its own 60-pixel margin, because a glyph touching the image border gets clipped. A line that fails to parse is visible; a line that parses into a different valid-looking number is not.

**Deducing which side the label sits on doesn't work.** Different camera apps stamp it left or right. The obvious inference — take the centre of mass of the first pass — *fails*, because printed text elsewhere in the frame drags the centre to the opposite side and the crop lands beside the label instead of on it. Sending both halves took another sample from 6/21 to 16/21; the area budget took it to 21/21.

**And in the merge, the longest read wins — not the last one.** The right-hand crop catches only the tails of the left-hand label's lines, and its truncated `748` was overwriting the other crop's complete `50.00748`. Last-write-wins is the default nobody chooses deliberately, and here it silently preferred the worse answer.

One more of the same family: the coordinate regex used a "find first match" call. The OCR concatenates numbers from adjacent lines and forms **plausible false pairs before the real one** — a longitude joined to a date fragment reads like a coordinate pair. It now iterates and stops at the first *plausible* match, which is a different thing from the first match.

Cost: two OCR calls per photo instead of one. At the current volume that's roughly €1.20 a month instead of €0.60, which is not a trade worth thinking about.

---

## 2. The geocoder never says "I couldn't find that"

Addresses that can't be read from the label are geocoded from other sources. The geocoding service always returns something, and it always looks like a hit. Measured against one town's data:

| Query | What comes back | Error |
|---|---|---|
| A real address | `PointAddress` | the actual doorway |
| Real street, **invented house number** | `Street` | midpoint of the street, ~860 m |
| **Invented street** | `Locality` | centre of the town |
| Nothing matches at all | the **bias position** | centre of the country, ~140 km |

All four pass a "is this plausibly in Europe" check without effort. A distance guard against the town centre catches the bottom two — and misses the one that matters most, because **a house number that doesn't exist lands comfortably inside the town's radius** and gets accepted as correct.

The mistake was asking the service for a position. It also returns a **place type**, which is the service telling you how much of your query it actually matched. That is now preserved and translated into an explicit precision — doorway, street, or town — carried all the way to the operator's screen. A street-level answer is still useful; presenting it as a doorway is the defect.

The consequence, and the reason this got the attention it did: that coordinate is written into the photo's EXIF and travels into the client's report. **A typo in an address becomes documented reality.**

---

## 3. The unit of work is the photo, not the folder

Photos without usable coordinates land in a folder literally named "no coordinates". It's tempting to read that as *one folder, one location* — and while porting an operator dialog from a desktop tool to the browser, it was tempting to collapse a question asked per photo into one answer per folder.

Checked against real data: one such folder held photos from **three different streets**. The address was never in the folder name. It was in each file's name, where it had been all along.

Collapsing that dialog would have stamped photos of different doorways with the same coordinate, and produced no error of any kind. Every operator input is now modelled per photo, and anything the operator leaves unanswered is skipped rather than defaulted.

---

## 4. Green is not evidence

The batch endpoint returned a fixed success string. The function underneath it didn't return its statistics at all. So a run could complete, report success, and have written **zero** coordinates — and nothing in the interface could tell that apart from a run that fixed everything.

A status without numbers behind it is worse than no status, because it closes the investigation.

Every batch operation now returns real counts — how many it examined, how many it corrected, how many are still pending and *why* — and the interface shows a row per photo rather than a summary.

**Underneath that was a structural fault worth naming.** Job state lived in three places at once: the microservice's memory, a jobs table, and the browser — with two synchronization loops between them. That single duplication produced five different visible bugs: "Running" displayed beside "completed", correct jobs marked *Failed* whenever the microservice restarted, progress frozen at 0 %, a stale error message stuck to a row, and an authentication failure on the event stream.

One fault, five faces, and each face had been looked at as its own bug.

Now there is exactly one owner: the service doing the work writes its state and a numbered log line per event; the other service reads. No polling between services and no event-stream relay — the browser asks for log lines after the one it already has. As a side effect the log survives closing the tab, which was another quiet loss nobody had listed as a bug.

---

## 5. The authorization that logged a warning instead of denying

The original check called a project lookup belonging to a different product line, which doesn't apply to this client — and which checks nothing at all when the identifier arrives null. In practice there was no authorization.

Rebuilding it turned up the sharper version of the same thing. The clients table stores a display name and a short code. The check received the **display name** and compared it against the **code**. They never matched, so the comparison fell through, and what remained was a log warning. Green tests, working feature, no enforcement. The code is now resolved server-side from the identifier and never accepted from the browser.

Two related decisions:

- **The file root failed open.** Without an explicit allow-list property, the path logic fell back to the entire network share — including another client's files. That policy now lives in one place and fails closed.
- **The legacy fallback is the dangerous direction.** "No scope rows means full access" means a user with *nothing* configured gets in, while a user who already has scopes for that client through a different module is correctly locked out. The permissive case is the one that doesn't generate a support ticket, so it's the one that survives unnoticed.

---

## 6. The decision I couldn't engineer around

Erasing the stamp is the feature the business cares about most, and the only region where the current model for it exists is outside the EU. The European-hosted predecessor is end-of-life and closed to new customers — and picking it would have repeated exactly the failure already sitting in production, which is a pipeline built on a model that was later retired.

Cropping the tile before sending it doesn't mitigate anything: **the crop is precisely the region containing the address.**

So this isn't a technical problem with a clever answer. It's a data-residency decision, and it was written down as one — the tradeoff stated, the decision taken by the person who owns it, and a recorded condition for revisiting it if the product is ever sold outside the company. An unexamined default would have made the same choice silently.

The engineering that *was* available was in the mask. Erasing the label's bounding rectangle covered 55 % of the tile — enough that the model lost its surrounding context and **invented** the background: in one photo a sandstone wall and a set of steps came back as plain concrete. Masking the individual text lines instead drops it to about 34 %, and the real background survives. Line boxes, not word boxes: the label is right-aligned, so word-level masks left the ends of lines unerased. Verified clean on 10 of 10 photos in the validation set.

---

## The ceiling, stated

Not every photo is solvable, and the honest number matters more than the good one. In one town, **18 of 40**.

The other 22 have no coordinates in the image *and no GPS in the EXIF* — zero of the forty carried EXIF GPS. Several camera apps are in circulation among the crews and only some stamp a position at all: two of the four formats encountered carry coordinates, two carry only a date or a technician name. No amount of OCR work recovers a number that was never written down.

What remains for those is the folder name, which *is* an address. Geocoding it yields doorway precision rather than the precision of the spot where the technician actually stood — a different measurement wearing the same units. It's implemented, and it's tagged with a distinct provenance value so it can never be silently mixed with a coordinate that was really read off the photo.

## Takeaway

Every defect in this write-up produced valid-looking output. A truncated number that was still a number, a coordinate for an address that doesn't exist, a green batch that did nothing, a permission check that compared two fields that could never be equal.

**A system whose failures are silent doesn't need better error handling. It needs the thing it produces to carry how it was produced** — the place type, the precision tag, the per-photo counts, the provenance value — so that "wrong" and "right" stop looking identical.

## Tools

Amazon Textract · Amazon Location Service · Amazon Bedrock (Stability image erase) · Python · FastAPI · Pillow · PostgreSQL · Spring Boot · React · EXIF/GPS metadata handling
