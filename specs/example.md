```markdown
# How to use `ontology_clinical_example.yaml` in the raw → TLF pipeline

**Purpose**  
This README explains, in plain language, how the pipeline should use the clinical ontology file (`ontology_clinical_example.yaml`) together with OSB domain YAMLs to run trusted metrics, build SDTM domains, derive ADaM, and produce TLFs — while recording provenance and enforcing governance.

---

## Where to put the file
Place the ontology file in your repo at:
```
specs/ontology/ontology_clinical_example.yaml
```

Also ensure your OSB exports are available at:
```
specs/domains/*.yaml
```

---

## High-level flow (plain steps)

### 1. Metric lookup (preferred path)
When a user or notebook requests a metric (e.g., SAE count):

1. Look up the metric by **id** in the ontology (e.g., `canonical_metrics.MET_SAE_COUNT`).  
2. If the metric has a `canonical_sql` (a trusted asset), use that SQL.  
3. Execute the SQL against the prepared view (for example, `ae_prepped`) in Databricks.  
4. Record provenance for the run: include `osb_export_ref`, `trusted_asset_id` (or metric id), `notebook_run_id`, timestamp, SQL snapshot, and `row_count`.

**Rule:** Always prefer `canonical_sql` from `trusted_assets` when present.

---

### 2. SDTM domain build (domain extraction)
1. **Preferred:** If the ontology contains a `trusted_assets[*].sql` for the domain, run that approved SQL to extract the SDTM domain.  
2. **Fallback:** If no trusted asset exists:
   - Read `specs/domains/<DOMAIN>.yaml`.
   - For each `variables[].derivation`, generate `SELECT <derivation> AS <VAR>` and `FROM <source_view>`.
   - Run the generated SQL to produce the SDTM domain table.
3. **Dry run:** Always preview generated SQL (dry run) and inspect row counts before full execution.

---

### 3. ADaM and TLF
- ADaM programs should reference canonical metric IDs (e.g., `MET_BASELINE_ALT_MEAN`) rather than ad‑hoc SQL.  
- TLF programs compute outputs from ADaM and include metric provenance (metric id, `osb_export_ref`, trusted asset id) in QC logs or footers.

---

## Governance & provenance (practical rules)
- Owners listed in `governance.owners` are notified on changes; conflict policy must be followed before a metric becomes `approved`.  
- Tag trusted assets with `last_validated` and `owner`.  
- Store the ontology file in Git and include `osb_export_id` in commits so runs are reproducible.  
- Capture provenance for every execution (see suggested fields below).

---

## Suggested provenance fields to capture
- `metric_id`  
- `domain`  
- `trusted_asset_id` (or `metric_id` if the metric is the trusted asset)  
- `osb_export_ref`  
- `notebook_run_id`  
- `user` (who triggered the run)  
- `run_ts` (timestamp)  
- `sql_snapshot` (the exact SQL executed)  
- `row_count`  
- `status` (success / failure)  

Store provenance in a Delta table (or equivalent) so it can be queried by QC and audit processes.

---

## Practical tips for a pilot
- **Start small:** 5–10 canonical metrics and the SDTM domains that feed them.  
- **Trusted assets:** Use `trusted_assets` for any SQL you want the pipeline to run without human re‑review. Mark each with `last_validated` and `owner`.  
- **Link OSB → ontology:** Use `osb_variable_links` to connect OSB YAML variables to ontology trusted assets so traceability is automatic:  
  `TLF → ADaM → SDTM → OSB derivation → ontology trusted SQL → raw view`.  
- **Automate change detection:** Enable change feeds or CI checks to notify owners when source views, OSB exports, or trusted SQL change.

---

## Quick checklist before running
- [ ] `specs/ontology/ontology_clinical_example.yaml` present and approved.  
- [ ] `specs/domains/*.yaml` validated (no empty derivations).  
- [ ] Trusted SQL assets reviewed and `last_validated` set.  
- [ ] Notebook has a Delta table for provenance and write permissions.  
- [ ] Dry run: preview generated SQL for each domain and metric.

---

## Troubleshooting tips
- **SQL fails on execution:** copy the generated SQL from the notebook cell, run interactively in a SQL editor, fix the derivation in OSB or the trusted asset, re‑export YAML, and re-run.  
- **Metric returns unexpected numbers:** check `last_validated`, `owner`, and `osb_export_ref` in provenance; compare to previous validated runs.  
- **Missing `source_view`:** confirm the `source_view` exists in the workspace or add it to `available_views.json` for validation.

---

## Minimal end‑to‑end flow (one run)
1. Init cell loads ontology and domain specs.  
2. Dry‑run cell prints generated SQL for AE and LB.  
3. Metric cell runs `MET_SAE_COUNT` using `canonical_sql` and writes provenance.  
4. SDTM build cell runs AE extraction (trusted asset or derivation) and writes SDTM table and provenance.  
5. ADaM cell derives analysis datasets referencing metric ids.  
6. TLF cell produces tables and includes provenance links for audit.

---

## Where to put this file in the repo
```
specs/
  ontology/
    ontology_clinical_example.yaml
  domains/
    ae.yaml
    lb.yaml
    dm.yaml
  ct/
    sdtmct_2024-09-27.json
```

```
