# Geo-LocalizationHITL

A model-agnostic, human-in-the-loop (HITL) spatial decision support system (SDSS) that helps investigative and OSINT journalists **verify a claimed image location**. MSc thesis project (GIMA). The goal is transparent, deterministic and defensible verification, not maximum benchmark accuracy.

Status: design settled (Nov 2026), implementation starting. The tool must be done by early January 2027. Build one component at a time when asked; don't scaffold the whole structure at once. Prefer a simple working version over a complete one.

## How we work together
- I'm a student and want to understand every line and every structural choice. Prefer simple, readable code over clever code.
- Before changes across several files, explain the plan and wait for my go.
- When I ask how to do something, explain and give hints instead of editing files, unless I explicitly ask you to implement it.
- When you introduce a concept, library or pattern I may not know, explain it in 1–2 sentences.
- Comments explain *why*, not *what*.
- Challenge my design: if you see a better approach or a problem, say so before implementing.
- Don't commit or push unless asked.

## Core workflow
1. **Input**: an image + a claimed location (place name → claim area by place type: country polygon / city ~20 km / street ~1 km) + optional claimed time. Free-text context (source, story) is stored for the journalist's final judgement only; no component computes with it.
2. **AI Stream 1 – location distribution**: a probability density over the globe.
3. **AI Stream 2 – location + clues**: a location prediction plus structured clues (fixed schema, each with a crop, source and confidence).
4. **Human Review**: the journalist confirms, corrects or dismisses each AI clue. Clues feed the geospatial checks; unconfirmed clues count with reduced weight.
5. **Geospatial evidence**: rule-based checks that test clues against open geodata around the claim (see below). The claim defines the search area; the claimed time enables the sun check.
6. **Human Evidence**: the journalist can accept or dismiss any layer and add external evidence from tools the system can't reach (Street View, Google Earth, …) via a structured form: type, geometry, source tool, link, strength, note.
7. **SDSS fusion**: every stream and check becomes an evidence layer on a common grid; layers are fused into one likelihood surface.
8. **Verdict**: a rule-based decision tree assigns **supported / contradicted / inconclusive**, with reason codes. Explanations and check requests come from fixed templates per reason code. **No LLM orchestrator.**
9. **Final verdict**: the journalist confirms or overrides, with a note. Only then can the case be exported.
10. **Audit trail + case database**: every step is logged (W3C PROV logic) and every case is stored.

## Geospatial checks
- Core: script on signs (constraint), driving side (constraint), brand names (factor), land cover / climate (factor), sun position if a time is given (constraint), EXIF GPS/time (low-weight factor).
- If time allows: OSM features near the claim, relief (DEM), licence plates.
- Not a layer: a data-quality flag (OSM density around the claim); poor mapping means missing features don't count against the claim.
- Left to the human: Street View / satellite comparison, skyline matching, camera position.
- Data: GeoINT lookup tables (marouane53/GeoINT: script, driving side, plates; spot-check against Wikidata), OSM name-suggestion-index, ESA WorldCover, Köppen-Geiger, Copernicus DEM, Natural Earth / GeoNames.

## SDSS and decision tree
- **Grid**: H3, coarse (res 3–4) globally, fine (res 6–7) in the case region.
- **Layer values**: rescale each layer so its best cell = 1, floor at ε (default 0.05), so every layer has the same maximum influence. Constraints: 1 = compatible, ε = not. A check without information is left out (neutral), never scored 0.
- **Fusion**: weighted log-linear pooling (sum of w · log layer per cell, then normalise).
- **Verdict rules, in order** (inconclusive is the default):
  - R0 fewer than two informative layers → inconclusive
  - R1 human-confirmed constraint violated at the claim → contradicted
  - R2 two or more independent AI-extracted constraints violated → contradicted
  - R9 strong human location match → supported inside the claim area / contradicted far outside it
  - R3 claim rank q ≥ τ_contra (start 0.90) → contradicted
  - R4 q ≤ τ_support (start 0.50) → candidate supported, only if R5–R8 all hold: no constraint violated (R5), no strong local contradiction unless explained by poor mapping (R6), streams don't strongly disagree (R7), surface not flat (R8)
- Every verdict stores its reason codes, the layer contributions at the claim, the country breakdown and "points instead to".
- Thresholds are tuned on a validation split only and fixed before the test run.

## Hard constraints
- **Sensitive content stays local**: the image, crops, clues, claim, context, all AI inference and the audit trail run on the user's machine or a controlled self-hosted environment (newsroom GPU node, university HPC, EU cloud). No AI APIs or cloud models; nothing derived from the image ever leaves (no reverse image search, no embeddings).
- **Public geodata lookups may go online**: basemap tiles, geocoding place names, Overpass queries, downloading geodata tiles. Log every outbound call (what, where, when) and cache results for reproducibility.
- **Open source**: open-source dependencies and open-weight models only. Ask before adding a dependency. Prefer MIT/BSD/Apache; GPL tools only as separate processes or services.
- **Model-agnostic**: model-specific code stays inside its stream adapter, behind one shared interface. Nothing else imports a model directly.
- **Deterministic**: verdicts come from code, never from a language model. Model inference with fixed seeds and settings.
- **Human has the final say**: the tool never finalises a verdict; all AI output can be overridden or ignored. "Inconclusive" is a valid outcome, not a failure.
- **Traceable**: every verdict links back to its evidence (image crop, stream, clue, check, data source) via the audit trail.
- **Explicit uncertainty**: show spatial uncertainty as an area or surface, never only a single point.
- **Sensitive data stays out of the repo**: case images, claims, sources and the case database live outside the repo (path in `config.local.yaml`, gitignored). Tests and examples use public or synthetic data only.

## Audit trail and cases
- Per case a folder: `events.jsonl` (append-only, hash-chained), `case.json`, `layers.gpkg`, `crops/`, report.
- Each event: timestamp (UTC), activity, agent (`human` / `ai` / `code`), used and generated entities (IDs with hashes), details (model version, seed, data source and version, outbound query).
- AI verdict and final verdict are separate entities with different agents.
- Case database (SQLite/GeoPackage): `cases`, `events`, `evidence`; the JSONL stays the source of truth.

## Open decisions (don't decide these on your own; propose options)
- Which models fill the two streams (RQ2 assessment; shortlist e.g. PLONK / GeoCLIP for Stream 1, Navig / Geo-R1 / GAEA / GeoVista for Stream 2).
- Final weights: current proposal is equal weights, with layers from the same source sharing weight and unconfirmed AI clues at 0.5.
- Exact threshold values and ε (tuned on validation).
- Frontend details (local web app with Leaflet; keep it simple). Map outputs must export to GeoPackage/GeoTIFF for QGIS.

## Evaluation
- Benchmark: OSV-5M test images paired with claims: the true place, plus false claims at 25 / 200 / 750 / 2,500 km (GeoNames populated places). Validation/test split.
- Main metrics: false support rate (headline), false contradiction rate, inconclusive rate, all per distance level; time and peak VRAM per image.
- Compare: automated vs simulated human input (correct and incorrect), ablations (each stream alone, without geospatial evidence), sensitivity (ε, weights).
- RQ4: does stream disagreement predict wrong verdicts? Analyse with R7 switched off.
- Run the AI streams once per image and cache their outputs; variants only rerun fusion and the decision tree.

## Layout (proposal; suggest an update to this file when it changes)
- `streams/` – shared stream interface + one adapter per model
- `clues/` – clue schema and extraction helpers
- `checks/` – one module per geospatial check, all returning an evidence layer
- `sdss/` – grid, normalisation, fusion, decision tree, templates
- `audit/` – event log, hash chain, PROV mapping
- `cases/` – case records and database access
- `app/` – backend + map interface
- `eval/` – benchmark builder and evaluation runs
- `data/` – data preparation scripts (+ small tables with their own licence notes; large data gitignored)
- `prompts/` – versioned prompt files for clue extraction (only edit when asked)
- `jobs/` – SLURM scripts for the UT-ITC cluster
- `tests/`

## Commands
- Add a package: `uv add <package>` (dev tools: `uv add --dev <package>`). Never use `pip install`.
- Run code: `uv run python <script>`
- Run tests: `uv run pytest`

## Conventions
- Python 3.11 (pinned in `.python-version`). Type hints and short docstrings on public functions.
- Small modules with one responsibility.
- Evaluation, fusion and decision-tree code must have tests; they produce thesis results.
- Coordinates in WGS84 (EPSG:4326). Name variables `lat`/`lon` explicitly; when a tuple is needed, use (lon, lat) as in GeoJSON.
- Code is MIT; third-party data keeps its own licence (e.g. CC BY-SA tables in `data/`). Record every dependency, model and dataset in `THIRD_PARTY_NOTICES.md`.
