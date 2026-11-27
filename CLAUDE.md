Geo-LocalizationHITL

A human-in-the-loop (HITL) spatial decision support system (SDSS) that helps investigative and OSINT journalists verify a claimed image location. MSc thesis project (GIMA). The goal is transparent, deterministic and defensible verification, not maximum benchmark accuracy.

Status: design settled (27 Nov 2026), implementation starting. The tool must be done by early January 2027. Build one component at a time when asked; don't scaffold the whole structure at once. Prefer a simple working version over a complete one.

How we work together
I'm a student and want to understand every line and every structural choice. Prefer simple, readable code over clever code.
Before changes across several files, explain the plan and wait for my go.
When I ask how to do something, explain and give hints instead of editing files, unless I explicitly ask you to implement it.
When you introduce a concept, library or pattern I may not know, explain it in 1–2 sentences.
Comments explain why, not what.
Challenge my design: if you see a better approach or a problem, say so before implementing.
Don't commit or push unless asked.
Current phase: GLOBE pilot

Everything downstream depends on GLOBE producing clean clues, so this comes first.

Run GLOBE locally: full BF16 via vllm serve on the HPC (evaluation), GGUF (Q8_0 or Q4_K_M, plus its vision projector mmproj file) via llama.cpp on the laptop (demo).
Write a prompt (in prompts/) that asks for the clue schema below as JSON; enforce it with guided decoding / a JSON schema where the server supports it.
Run on ~50 OSV-5M images and check per field: is the JSON valid, are the clues correct, does the country/city prediction hold, are the boxes usable?
Decide on crops: if GLOBE's boxes are poor or missing, add a detector (GroundingDINO tiny or OWLv2, both Apache-2.0) that boxes the clue text GLOBE names.
Compare BF16 with the quantized version on the same images. Pilot code lives in pilot/ and may be throwaway; what works moves into streams/.
Core workflow
Input: an image + a claimed location (+ optional claimed time). Free-text context (source, story) is stored but only shown to the journalist at the final assessment (linear sequential unmasking, Dror & Kukucka, 2021); no component computes with it. The AI streams only ever see the image.
AI Stream 1 – PLONK (Dufour et al.): a probability density over the globe.
AI Stream 2 – GLOBE (GLOBE-Qwen2.5VL-7B): reasoning text, a country/city prediction and structured clues with crops.
Human review: the journalist confirms, corrects, dismisses or adds clues. Only confirmed clues go to the lookups.
Geospatial evidence: confirmed clues are matched against local lookup tables → constraint layers per country; place names → cities500 factor layer.
SDSS fusion: every source becomes an evidence layer on an H3 grid; layers are fused into one probability surface.
Verdict: a rule-based decision tree gives supported / contradicted / inconclusive with a reason code. No LLM orchestrator.
Check requests: fixed templates, triggered by the rule that stopped the tree, tell the journalist what evidence is missing.
Human evidence: the journalist adds external evidence from tools the system can't reach (Google Earth, Street View, …) via a form: type (location match / area ruled out / supporting region), geometry drawn on the map, source tool, strength (weak / moderate / strong), note. The tree re-runs.
Final verdict: the loop ends when the journalist gives the final verdict. Tool verdict and human verdict are stored separately.
Audit trail + case database: every step is logged (W3C PROV) and every case is stored.
Clue schema (GLOBE output, draft – refine in the pilot)
json
{"reasoning": "...",
 "prediction": {"country": "GR", "city": "Thessaloniki"},
 "clues": [{"id": 1, "object_id": "a", "type": "script",
            "value": "GREEK", "text": "ΦΑΡΜΑΚΕΙΟ", "box": [x1, y1, x2, y2]}]}
type: script | driving_side | phone_code | country_name | place_name | other. Only the first five map to a lookup; other is shown to the human but not used by the rules.
object_id groups clues from the same object (one sign, one shop), which share weight. The human can regroup during review.
box may be null; then the crop comes from the detector.
Geospatial evidence

Lookup tables (GeoINT by marouane53, mostly from Geo-Sleuth) + GeoNames + Natural Earth, stored locally:

Table	Use	Type
driving_side.json (side + year switched)	driving side clue	constraint
text_signals.json (countries, weak_countries per script)	script clue	constraint
calling_codes.json	phone code clue	constraint (identifying)
country_names.json	country name on a sign	constraint (identifying)
countries.json	master list, ISO-2 join key	–
GeoNames cities500	place-name clue + geocoding the claim	factor
Natural Earth 1:50m admin 0	country polygons (join on ISO_A2_EH, not ISO_A2)	–
Lookups work at country scale only. City-scale evidence comes from GLOBE's city, PLONK, a cities500 match or human evidence.
Out of scope (journalist does these via external evidence; future work): EXIF, sun position, brands, land cover, OSM queries, DEM, licence plates, satellite comparison.
SDSS
Grid: H3 (local library, no API). Coarse globally, fine (res 6–7) around the claim.
Layer values: factors divided by their maximum (best cell = 1); every layer has a minimum value of 0.05 (a layer can make a cell at most 20× less likely, never impossible). Constraints: 1 = compatible, 0.05 = incompatible; weak_countries = 0.5. A clue without a lookup match is left out (neutral), never scored 0.
GLOBE layer: within ~20 km of predicted city = 1, rest of predicted country = 0.5, rest of world = 0.05.
External evidence: inside the drawn area = 1; outside = 0.5 / 0.2 / 0.05 for weak / moderate / strong.
Weights: every independent source = 1 (PLONK, GLOBE, each clue object, each external evidence item); clues from the same object share weight (1 / number of clues).
Fusion: log-linear pooling, S(c) = Σ wᵢ · log Lᵢ(c), p(c) = exp S(c) / Σ exp S (Genest & Zidek, 1986).
Weights only affect the fused surface (map, R5 flatness, R8). Contradictions never depend on chosen weights.
Decision tree (in order; inconclusive is the default)
R1 strong human evidence conflicts with a confirmed clue → inconclusive (re-check conflict)
R2 strong human location match → supported inside the claim area / contradicted far outside
R3 strong human "area ruled out" covers the claim → contradicted
R4 a confirmed clue is incompatible with the claimed country → contradicted
R5 too little evidence (no confirmed clues, no external evidence, flat surface) → inconclusive
R6 claim outside the 90% region of PLONK and of GLOBE → contradicted (AI-based)
R7 streams disagree (e.g. GLOBE's country outside PLONK's 90% region) → inconclusive (ask for distinguishing clues)
R8 claim inside the fused 50% region and at least one identifying confirmed clue → go to R9, else inconclusive
R9 city-scale evidence near the claim (PLONK peak, GLOBE city, place match) → supported; else inconclusive (country consistent)
Every verdict stores its reason code, the rule path, the evidence used and the layer values at the claim.
Parameters (start values; tune on validation only, fix before the test run)
Parameter	Value
Contradiction region (R6)	90%
Support region (R8)	50%
Identifying clue	fits ≤ 5 countries
City-scale distance	20 km
Flat surface (R5)	normalised entropy > 0.9
Minimum value	0.05
Weak countries	0.5
GLOBE rest of country	0.5
Hard constraints
Sensitive content stays local: image, crops, clues, claim, context, all AI inference and the audit trail run on the user's machine or a controlled self-hosted environment (newsroom GPU, university HPC, EU cloud). No AI APIs, no reverse image search; nothing derived from the image leaves.
GLOBE's README uses the openai client: that client must only ever point at the local vLLM / llama.cpp endpoint, never at an external API.
Allowed online: basemap/satellite tiles and geocoding of place names. Log every outbound call (what, where, when).
Open source: open-source dependencies and open-weight models only. Ask before adding a dependency. Prefer MIT/BSD/Apache.
Deterministic: verdicts come from code, never from a language model. Inference with fixed seeds and settings.
Human has the final say: the tool never finalises a verdict; all AI output can be overridden or ignored. "Inconclusive" is a valid outcome.
Traceable: every verdict links back to its evidence via the audit trail.
Sensitive data stays out of the repo: case images, claims, sources and the case database live outside the repo (path in config.local.yaml, gitignored). Tests use public or synthetic data only.
Audit trail and cases
W3C PROV: entities (image, clue versions, layers, verdicts), activities (plonk_run, globe_run, review_clue, lookup, fuse, evaluate_rules, add_evidence, final_assessment), agents (journalist, PLONK, GLOBE, rule code).
Per case: events.jsonl, append-only, one event per line, each with the hash of the previous line (hash chain). Nothing is overwritten: a corrected clue is a new version with wasDerivedFrom the old one; dismissed clues stay with status "dismissed".
Log model versions, prompt hashes, lookup-table hashes and the parameter set, so a case can be re-run with the same verdict.
Case database: local SQLite, one row per case (id, claim lat/lon, status, tool verdict, final verdict, timestamps, path to JSONL). The map view reads it; the JSONL is the source of truth. Optional export to PROV-JSON.
Open decisions (don't decide these on your own; propose options)
Crops: GLOBE boxes vs detector (pilot).
PLONK checkpoint: OSV-5M vs YFCC.
Claim area: how a claim (point or place name) becomes an area (e.g. point + 20 km, geocoded via cities500).
Use of claimed time: only the switched year in driving_side, or just logged for the human.
Interface details (keep simple, e.g. Streamlit + map; build last).
Evaluation (Objective 3)
RQ3: how accurately does the PoC verify claimed image locations, and how does human input affect this compared to a fully automated baseline?
RQ4: to what extent does disagreement between independent streams indicate unreliable verdicts? (analyse with R7 switched off)
Benchmark: OSV-5M images with the true place as claim, plus false claims at 25 / 200 / 750 / 2,500 km (GeoNames populated places). Validation/test split.
Metrics per distance level: false support rate (headline), false contradiction rate, inconclusive rate; time and peak VRAM per image.
Compare: automated vs simulated human input; ablations (each stream alone, without lookups, R6/R7 on/off); sensitivity (minimum value 0.01 / 0.05 / 0.1, GLOBE country value 0.3 / 0.5 / 0.7, weight sharing on/off, external-evidence floors).
Run the AI streams once per image and cache outputs; variants only re-run fusion and the tree.
Build order: pipeline end to end from the command line → evaluation → interface last.
Layout (proposal; suggest an update to this file when it changes)
pilot/ – GLOBE pilot scripts (throwaway)
streams/ – plonk.py, globe.py, shared output format
clues/ – clue schema, validation, crops/detector
lookups/ – loading the tables, clue → constraint layer
sdss/ – H3 grid, layers, fusion, decision tree, check-request templates
audit/ – event log, hash chain, PROV mapping
cases/ – case records and SQLite access
app/ – interface
eval/ – benchmark builder and evaluation runs
data/ – preparation scripts and small tables with licence notes (large data gitignored)
prompts/ – versioned prompt files (only edit when asked)
jobs/ – SLURM scripts for the UT-ITC cluster
tests/
Commands
Add a package: uv add <package> (dev tools: uv add --dev <package>). Never use pip install.
Run code: uv run python <script>
Run tests: uv run pytest
Conventions
Python 3.11 (pinned in .python-version). Type hints and short docstrings on public functions.
Small modules with one responsibility.
Lookups, fusion, decision-tree and evaluation code must have tests; they produce thesis results.
Coordinates in WGS84 (EPSG:4326). Name variables lat/lon explicitly; when a tuple is needed, use (lon, lat) as in GeoJSON. Countries as ISO-2 codes everywhere.
Code is MIT; third-party data keeps its own licence (GeoINT tables CC BY-SA 4.0 / MIT, GeoNames CC BY 4.0, Natural Earth public domain). Record every dependency, model and dataset in THIRD_PARTY_NOTICES.md.