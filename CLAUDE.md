# Geo-LocalizationHITL

A model-agnostic, human-in-the-loop (HITL) tool that helps investigative and OSINT journalists find or verify where an image was taken. MSc thesis project (GIMA). The goal is transparent, local, and defensible verification, not maximum benchmark accuracy.

Status: early design, nothing implemented yet. Build one component at a time when asked; don't scaffold the whole structure at once.

## How we work together
- I'm a student and want to understand every line and every structural choice. Prefer simple, readable code over clever code.
- Before changes across several files, explain the plan and wait for my go.
- When I ask how to do something, explain and give hints instead of editing files, unless I explicitly ask you to implement it.
- When you introduce a concept, library or pattern I may not know, explain it in 1–2 sentences.
- Comments explain *why*, not *what*.
- Challenge my design: if you see a better approach or a problem, say so before implementing.
- Don't commit or push unless asked.

## Core idea
1. **Input**: an image, optionally with a claimed location (verification mode). Without a claim, the tool searches open-ended.
2. **Evidence streams**: N independent AI models with different approaches analyse the image. Human context is also a stream. The framework must work for any N; the thesis configures two AI streams + the human.
3. **Orchestrator**: one local model collects stream outputs, can ask the human to do tasks (e.g. "look up this street sign"), and integrates the answers.
4. **Evidence layers**: each piece of evidence becomes a weighted probability layer on a common world grid.
5. **Fusion**: layers are combined by Bayesian updating into a posterior over locations.
6. **Hierarchical output**: confidence per level (continent, country, region, city) by aggregating the posterior over administrative units. No classifier training.
7. **Verdict**: the tool *proposes* a graded outcome (verified / partially verified / not verifiable / false) with its justification. The journalist confirms or overrides it.
8. **Map**: shows the posterior, the per-level confidence, and where each piece of concrete evidence points.
9. **Case database**: every case is stored with inputs, evidence, layers, verdict and a log of human and AI actions.

## Hard constraints
- **Local only**: everything runs on the user's machine or a controlled environment (target: a single GPU node owned by a newsroom; development on university HPC). No external APIs, cloud models, online tile servers, CDNs or telemetry at runtime. Any external lookup must be triggered by the user, labelled and logged.
- **Open source**: open-source dependencies and open-weight models only. Ask before adding a dependency.
- **Model-agnostic**: model-specific code stays inside its stream adapter, behind one shared interface. Nothing else imports a model directly.
- **Human has the final say**: the tool never finalises a verdict; all AI output can be overridden or ignored. "Not verifiable" is a valid outcome, not a failure.
- **Traceable**: every conclusion links back to its evidence (image region, stream, reasoning) and is logged with model version, prompt version and settings. Deterministic inference where possible (temperature 0, fixed seeds).
- **Explicit uncertainty**: always show spatial uncertainty as an area or probability, never only a single point.
- **Sensitive data stays out of the repo**: case images, claims, sources and the case database live outside the repo (path set in `config.local.yaml`, which is gitignored). Tests and examples use public or synthetic data only.

## Open decisions (don't decide these on your own; propose options)
- Which models fill the streams. Keep code model-agnostic; hardcode no model.
- Orchestrator model and how it interacts with the human.
- Spatial grid for the layers (e.g. raster vs hexagonal) and its resolution.
- Priors, weights and likelihoods for the Bayesian update.
- How the claimed location enters the pipeline. Until decided: don't pass it to models or use it as a prior; treat it as the hypothesis being tested.
- Frontend: local web app vs QGIS. Either way, map outputs must export to open GIS formats (GeoPackage, GeoTIFF) so they open in QGIS.
- Administrative boundary dataset for the hierarchy.
- Database technology for the cases.

## Evaluation
- Test set: OpenStreetView-5M (OSV-5M) test split.
- Accuracy at 1 / 25 / 200 / 750 / 2500 km (street / city / region / country / continent) like other academics do.
- Also: calibration (is high confidence actually right more often?), stream disagreement vs error, and the workflow with vs without the human.

## Layout (proposal; suggest an update to this file when it changes)
- `streams/` – shared stream interface + one adapter per model
- `orchestrator/` – collects evidence, asks the human
- `fusion/` – evidence layers, Bayesian update, hierarchy
- `cases/` – case records and database access
- `app/` – map interface
- `eval/` – evaluation on OSV-5M
- `prompts/` – versioned prompt files (only edit when asked)
- `jobs/` – SLURM scripts for the UT-ITC cluster
- `tests/`

## Commands
- Add a package: `uv add <package>` (dev tools: `uv add --dev <package>`). Never use `pip install`.
- Run code: `uv run python <script>`
- Run tests: `uv run pytest`

## Conventions
- Python 3.11 (pinned in `.python-version`). Type hints and short docstrings on public functions.
- Small modules with one responsibility.
- Evaluation and fusion code must have tests; they produce thesis results.
- Coordinates in WGS84 (EPSG:4326). Name variables `lat`/`lon` explicitly; when a tuple is needed, use (lon, lat) as in GeoJSON.