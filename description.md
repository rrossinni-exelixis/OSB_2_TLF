### Executive summary
This is a **nontechnical workflow** that shows exactly how the OSB artifacts  (OSB JSON / domain YAMLs / CT package / exporter scripts) are used to convert raw clinical data into final tables, listings, and figures (TLFs) via Databricks pipeline. Follow the steps below in order — each step names the artifact and the action to take.

---


- **OSB study JSON** — the canonical study definition (derivations, visits, endpoints).  
- **Domain YAMLs** (`specs/domains/*.yaml`) — one file per SDTM domain with `source_view` and `derivation` for each variable.  
- **CDISC CT package** (`specs/ct/...json`) — controlled terminology used for validation.  
- **Exporter scripts** (`export_osb_to_yaml.py`, `export_osb_to_yaml_with_validation.py`) — convert OSB JSON to YAML and validate.  
- **Databricks notebook pipeline** — reads the YAMLs, generates SQL, builds SDTM, derives ADaM, and runs TLF programs.

---

### Simple step‑by‑step workflow from raw dataset to TLF

1. **Author protocol and mappings in OSB**  
   - People enter protocol sections, CRF fields, endpoints, and mapping decisions into OSB.  
   - OSB stores each SDTM variable’s **derivation** (a small SQL expression or template) and links to codelists.

2. **Export OSB JSON**  
   - Export the study from OSB as `osb_study.json`. This file is the single source of truth for all downstream artifacts.

3. **Generate domain YAMLs**  
   - Run the exporter script to convert `osb_study.json` into `specs/domains/*.yaml`. Each YAML lists `source_view` and `variables[].derivation`.

4. **Validate specs locally**  
   - Run `export_osb_to_yaml_with_validation.py`. It checks: every variable has a derivation, `source_view` names exist (Databricks or `available_views.json`), and codelist names match the CT package. Fix any errors and re-export.

5. **Deploy YAMLs to Databricks repo**  
   - Copy `specs/domains/*.yaml` and `specs/ct/*` into the pipeline repo or workspace location the notebook expects.

6. **Dry run the notebook in Databricks**  
   - Set runtime meta from `specs/osb_runtime_meta.yaml`. Run the notebook in preview mode to show the generated SQL and row counts for each domain. Review and correct any SQL or missing view issues.

7. **Run full pipeline to build SDTM and ADaM**  
   - Notebook executes generated SQL against the `source_view` tables to create SDTM domains. ADaM programs then derive analysis datasets from SDTM using documented mappings.

8. **Generate TLFs and reconcile**  
   - TLF programs read ADaM datasets to produce formatted outputs. Run reconciliation scripts that compare key numbers back to ADaM and SDTM to confirm traceability.

---

### What the Databricks pipeline actually does:
- **Reads the YAMLs**: For each domain the notebook reads `source_view` and each variable’s `derivation`.  
- **Builds SQL**: It substitutes each derivation into `SELECT <derivation> AS <VAR>` and uses `FROM <source_view>`.  
- **Executes SQL to create SDTM**: The SQL runs in Databricks to produce SDTM domain tables. Postprocessing flags (sequence numbers, day calculations) are applied after the SELECT.  
- **Runs ADaM derivations**: ADaM code consumes SDTM to create analysis datasets with population flags and derived endpoints.  
- **Produces TLFs**: Statistical programs read ADaM to compute and format tables, listings, and figures. QC scripts reconcile TLF numbers to ADaM and SDTM.

---

### Checklist before execution:
- **OSB JSON exported and versioned**.  
- **specs/domains/*.yaml present and validated** (no empty derivations).  
- **specs/ct/** contains your CT JSON.  
- **available_views.json** or Databricks credentials available for source view checks.  
- **Run notebook dry mode** and confirm generated SQL and row counts.  
- **Save logs**: keep the OSB export id, generated SQL, and notebook run id for traceability.

---

### One‑line traceability example
**TLF number** → comes from **ADAM variable** → derived from **SDTM variable** → produced by SQL using **OSB derivation** stored in `ae.yaml` → originally defined in the **protocol** inside OSB.

---

If you want, I can now produce a **one‑page cheat sheet** that maps each YAML key to the exact notebook action (e.g., `variables[].derivation → SELECT expression`) so nontechnical team members can follow the pipeline without ambiguity.
