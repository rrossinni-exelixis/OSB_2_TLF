### Executive summary
You already have OSB artifacts (the OSB JSON, the `specs/domains/*.yaml` files, the CT placeholder, and exporter scripts). Those artifacts are **not arbitrary** — they are generated from a structured protocol/study definition authored in OpenStudyBuilder (OSB). Below I explain, in plain English and end‑to‑end, **where each artifact came from**, what inputs produced it, how OSB transforms those inputs into the JSON/YAML you’re using, and how to preserve provenance so every SDTM/ADaM/TLF value can be traced back to the original protocol or raw source.

---

### 1. Source inputs that feed OSB (where the artifacts originate)
OSB artifacts are produced from a small set of human and machine inputs. Think of OSB as a factory that consumes these inputs and emits structured study metadata.

**Primary human inputs**
- **Protocol text** — the clinical protocol (objectives, endpoints, visit schedule, assessments).  
- **Statistical Analysis Plan (SAP)** — analysis endpoints, populations, derived variables, TLF specs.  
- **Case Report Form (CRF) design** — what is collected on each visit; field names and units.  
- **Domain/SDTM/ADaM mapping decisions** — how protocol concepts map to SDTM variables and ADaM derivations.  
- **Controlled terminology choices** — which MedDRA/WHO/CT codes to use for terms.

**Primary machine inputs**
- **CDISC CT package (codelists JSON)** — the controlled terminology package used for validation.  
- **Existing raw views or sample raw files** — `dm_prepped`, `ae_prepped`, etc., used to test derivations.  
- **Templates** — Word protocol templates, SoA templates, YAML templates for domain specs.

OSB authors (protocol writers, data managers, statisticians) enter and link these inputs inside the OSB UI or via an API.

---

### 2. What OSB does to those inputs (the transformation)
OSB converts narrative and spreadsheet inputs into a **semantic, standards‑linked study model**. Key steps:

1. **Capture as structured metadata**  
   - Protocol sections (objectives, endpoints, visits) are stored as discrete objects (e.g., `Endpoint: ProgressionFreeSurvival`, `Visit: Week 5 Day 1`).  
   - CRF fields are captured as `Assessment` objects with field names, units, and expected data types.

2. **Link to standards and terminology**  
   - Each concept is linked to CDISC SDTM/ADaM concepts and to codelists (MedDRA, CT).  
   - OSB stores these links in the semantic graph (Neo4j in your architecture).

3. **Define derivations and templates**  
   - For each SDTM variable OSB stores a **derivation expression** (SQL snippet or template) and metadata (type, core/exp, codelist).  
   - For SoA and protocol text OSB stores template fragments and content controls for Word export.

4. **Versioning and audit trail**  
   - Every change is versioned; OSB records who changed what, when, and why. This becomes provenance metadata.

5. **Export**  
   - OSB serializes the semantic model into machine formats: a study JSON (single object) and/or per‑domain YAMLs. These exports include `source_view`, `variables[].derivation`, `codelist` references, `postprocessing` flags, and runtime metadata.

---

### 3. The artifacts you see and exactly how they were produced
Below are the artifacts in your bundle and the OSB origin for each.

- **`osb_study.json` (OSB JSON)**  
  - **Origin:** direct OSB export of the semantic study model.  
  - **Contains:** study metadata, arms, visits, domains array, runtime flags, CT package path.  
  - **How produced:** user clicks “Export JSON” or an API call `GET /study/<id>/export/json`.

- **`specs/domains/*.yaml` (DM, AE, LB, EX, VS, CM, CMH)**  
  - **Origin:** OSB export or exporter script that converts OSB JSON → one YAML per domain.  
  - **Contains:** `domain`, `source_view`, `variables` with `derivation`, `codelist`, `postprocessing`.  
  - **How produced:** OSB templates map semantic nodes to YAML fields; exporter serializes them.

- **`specs/ct/sdtmct_2024-09-27.json` (placeholder)**  
  - **Origin:** external CDISC CT package you place into the repo; OSB references it for validation.  
  - **How used:** exporter loads it to validate `codelist` names referenced in YAML.

- **`export_osb_to_yaml.py` / `export_osb_to_yaml_with_validation.py`**  
  - **Origin:** utility scripts you run locally to convert OSB JSON into the pipeline’s YAML format and to validate.  
  - **How used:** run the script; it reads `osb_study.json`, writes `specs/domains/*.yaml`, and produces `validation_report.yaml`.

- **`specs/osb_runtime_meta.yaml`**  
  - **Origin:** derived from OSB runtime settings (new_run, subjects per arm, timestamp format).  
  - **How used:** notebook reads this to set environment variables and output directories.

---

### 4. Provenance and traceability: how to prove where a value came from
To avoid the “gaps” you mentioned, maintain these provenance artifacts and practices:

**Provenance fields to include in OSB exports**
- `created_by`, `created_at`, `version` (study-level).  
- For each domain variable: `derivation`, `derivation_source` (protocol section or CRF field), `author`, `last_modified_at`, `ct_reference` (codelist OID).  
- For each YAML: `osb_export_id` and `osb_export_timestamp`.

**How traceability flows at runtime**
- Notebook generates SQL from `variables[].derivation` and logs the exact SQL and the `osb_export_id` used.  
- SDTM rows include metadata in logs (build timestamp, spec version).  
- ADaM programs include a mapping table (ADaM var → SDTM var → OSB derivation).  
- TLF QC scripts compare TLF numbers back to ADaM and then to SDTM and raw SQL outputs, referencing the OSB export id and variable derivations.

**Example trace chain for a TLF cell**
`TLF cell` → uses `ADAE.AESEV` → derived from `ADAE` code referencing `SDTM.AE.AESEV` → `SDTM.AE.AESEV` was produced by SQL `SELECT CASE WHEN AESERIOU='YES' THEN 'Y' ELSE 'N' END AS AESER` → `derivation` stored in `ae.yaml` and `osb_study.json` with `derivation_source: Protocol Section 7.2` and `ct_reference: MedDRA_PT`.

---

### 5. Practical step‑by‑step: from protocol to TLF with no gaps
1. **Author in OSB**  
   - Enter protocol sections, endpoints, visits, CRF fields, and mapping decisions. Link each concept to SDTM/ADaM concepts and codelists.

2. **Export OSB JSON**  
   - Export the study JSON and save as `osb_study.json`. This file contains the canonical derivations and links.

3. **Run exporter + validation**  
   - Run `export_osb_to_yaml_with_validation.py`. Fix any missing derivations, missing codelists, or missing source views. Commit the validated YAMLs.

4. **Record export metadata**  
   - Commit `osb_study.json`, `specs/domains/*.yaml`, and `specs/osb_runtime_meta.yaml` together and tag the commit with the OSB export id and timestamp.

5. **Dry run the notebook**  
   - Run the pipeline in preview mode. Inspect generated SQL and row counts. Save the generated SQL and the notebook run id.

6. **Full SDTM build**  
   - Execute SDTM build. Save SDTM build logs and the `osb_export_id` used.

7. **ADaM derivation with mapping table**  
   - Run ADaM programs that produce a mapping table `ADAM_VAR_MAP` linking ADaM vars to SDTM vars and OSB derivations.

8. **TLF generation and QC**  
   - Produce TLFs and run reconciliation scripts that reference `ADAM_VAR_MAP` and SDTM logs to confirm numbers.

---

### 6. Quick operational checklist you can paste into your repo
- [ ] OSB authored: protocol, SAP, CRF, mappings entered and linked in OSB.  
- [ ] Exported `osb_study.json` saved and versioned.  
- [ ] Run `export_osb_to_yaml_with_validation.py` → fix errors → commit `specs/domains/*.yaml`.  
- [ ] Place CT package at `specs/ct/sdtmct_2024-09-27.json`.  
- [ ] Run notebook dry run; save generated SQL and review.  
- [ ] Run SDTM build; save logs and `osb_export_id`.  
- [ ] Run ADaM derivations; produce `ADAM_VAR_MAP`.  
- [ ] Run TLFs; reconcile key numbers back to ADaM and SDTM using `osb_export_id` and mapping tables.

---

If you want, I will now:
- produce a **sample provenance block** (JSON snippet) you can embed in each YAML and in the notebook logs, or  
- generate the **one‑page cheat sheet** mapping each OSB field to the exact notebook cell and SQL pattern so your team can implement traceability without ambiguity.
