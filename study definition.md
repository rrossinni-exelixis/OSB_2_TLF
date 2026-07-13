### Executive summary
Below is a **reverse‑engineered OpenStudyBuilder (OSB) study definition** that would directly feed your `raw_2_tlf_v1` pipeline. It captures the same **domains, source views, derivations, controlled‑terminology links, SUPPQUAL definitions, build order, and run flags** the notebook expects so the pipeline can auto‑generate the YAML domain specs and downstream SDTM/ADaM/TLF outputs.

> **From the pipeline notebook:** “Each `.yaml` file (stored in `specs/domains/`) is a plain‑text blueprint that describes how to build one SDTM domain. It has four main sections.”   
> **Also:** “The Python builder in Cell 6 reads this file, loops over the `variables` list, and builds a single Spark SQL `SELECT` statement automatically.” 

---

### Scope chosen for the OSB output
- **Study:** XL092‑123 (two‑arm oncology POC; arms A15 / A25; N≈200 SAF)  
- **Pipeline targets:** DM, AE, LB (these are the YAMLs the notebook checks and uses to build SDTM).  
- **Goal:** produce a single OSB JSON object that can be transformed into the per‑domain YAML files the notebook expects (one YAML per SDTM domain), plus the SoA and visit/activity metadata used by pseudodata and ADTR generation.

---

### OSB study definition (representative JSON)
Below is a compact, ready‑to‑serialize OSB output. Use this as the **single source of truth**; a small exporter converts each `domains[*]` entry into `specs/domains/<domain>.yaml` with the same fields the notebook reads.

```json
{
  "study": {
    "study_id": "XL092-123",
    "title": "Two-arm oncology POC (Active 15 mg vs Active 25 mg)",
    "population": "SAF",
    "project_root": "/Workspace/Users/rossinni.r@gmail.com/Drafts/SDTM_Build_POC"
  },
  "runtime": {
    "new_run": true,
    "reuse_timestamp": null,
    "subjects_per_treatment_arm": 100,
    "screen_failure_subjects": 20,
    "pseudo_random_seed": 311,
    "timestamp_format": "YYYYMMDDTHHMMSSZ"
  },
  "arms": [
    {"armcd":"A15","arm":"Active Treatment 15 mg","site":"101","dose_mg":15},
    {"armcd":"A25","arm":"Active Treatment 25 mg","site":"102","dose_mg":25}
  ],
  "visits": [
    {"visit_name":"Week 1 Day 1","visit_num":1,"offset_days":0},
    {"visit_name":"Week 5 Day 1","visit_num":2,"offset_days":28},
    {"visit_name":"Week 9 Day 1","visit_num":3,"offset_days":56},
    {"visit_name":"Week 17 Day 1","visit_num":4,"offset_days":112}
  ],
  "domains": [
    {
      "domain": "DM",
      "label": "Demographics",
      "source_view": "dm_prepped",
      "depends_on": [],
      "build_order_priority": 1,
      "variables": [
        {"name":"STUDYID","derivation":"'XL092'","type":"Char","core":"Req"},
        {"name":"USUBJID","derivation":"Subject","type":"Char","core":"Req"},
        {"name":"SUBJID","derivation":"SUBJID","type":"Char","core":"Req"},
        {"name":"AGE","derivation":"CAST(AGE AS INTEGER)","type":"Num","core":"Req"},
        {"name":"AGEU","derivation":"'YEARS'","type":"Char","core":"Req"},
        {"name":"SEX","derivation":"SEX_STD","type":"Char","codelist":"C66731","core":"Req"},
        {"name":"RACE","derivation":"RACE_STD","type":"Char","codelist":"C74457","core":"Exp"},
        {"name":"ARM","derivation":"ARM_STD","type":"Char","core":"Req"},
        {"name":"ARMCD","derivation":"ARMCD_STD","type":"Char","core":"Req"},
        {"name":"RFSTDTC","derivation":"RANDDAT","type":"Char","core":"Req"},
        {"name":"RFXSTDTC","derivation":"(SELECT MIN(ECSTDAT) FROM ec_or1_raw WHERE Subject = dm_prepped.Subject)","type":"Char","core":"Exp"},
        {"name":"DTHDTC","derivation":"DDDAT","type":"Char","core":"Exp"}
      ],
      "suppqual_variables": []
    },
    {
      "domain": "AE",
      "label": "Adverse Events",
      "source_view": "ae_prepped",
      "depends_on": ["DM"],
      "build_order_priority": 3,
      "variables": [
        {"name":"STUDYID","derivation":"'XL092'","type":"Char","core":"Req"},
        {"name":"USUBJID","derivation":"USUBJID","type":"Char","core":"Req"},
        {"name":"AESEQ","derivation":"[XL_SEQ:AE]","type":"Num","core":"Req"},
        {"name":"AETERM","derivation":"AETERM","type":"Char","core":"Req"},
        {"name":"AEDECOD","derivation":"UPPER(TRIM(AETERM_PT))","type":"Char","core":"Req","codelist":"MedDRA_PT"},
        {"name":"AEBODSYS","derivation":"AETERM_SOC","type":"Char","core":"Req","codelist":"MedDRA_SOC"},
        {"name":"AESER","derivation":"CASE WHEN AESERIOU = 'YES' THEN 'Y' ELSE 'N' END","type":"Char","core":"Req","codelist":"C66742"},
        {"name":"AESTDTC","derivation":"DATE_FORMAT(TO_DATE(AEDAT,'ddMMMyyyy'),'yyyy-MM-dd')","type":"Char","core":"Req"},
        {"name":"AEENDTC","derivation":"CASE WHEN AEONGO_STD = 'Y' THEN NULL ELSE DATE_FORMAT(TO_DATE(AEENDAT_INT,'yyyy-MM-dd'),'yyyy-MM-dd') END","type":"Char","core":"Exp"},
        {"name":"AETOXGR","derivation":"CAST(AE_CTCGRADE AS STRING)","type":"Char","core":"Exp"},
        {"name":"AEOUT","derivation":"AEOUT_STD","type":"Char","core":"Exp","codelist":"C66768"}
      ],
      "suppqual_variables": [
        {"name":"AECONTRT","label":"Concomitant Treatment Given","derivation":"AECONTRT_STD"}
      ],
      "postprocessing": ["[XL_SEQ:AE]","[XL_DY:AESTDTC-RFSTDTC]"]
    },
    {
      "domain": "LB",
      "label": "Laboratory Test Results",
      "source_view": "lb_prepped",
      "depends_on": ["DM"],
      "build_order_priority": 2,
      "variables": [
        {"name":"STUDYID","derivation":"'XL092'","type":"Char","core":"Req"},
        {"name":"USUBJID","derivation":"USUBJID","type":"Char","core":"Req"},
        {"name":"LBSEQ","derivation":"[XL_SEQ:LB]","type":"Num","core":"Req"},
        {"name":"LBDTC","derivation":"DATE_FORMAT(TO_DATE(LBDAT,'ddMMMyyyy'),'yyyy-MM-dd')","type":"Char","core":"Req"},
        {"name":"LBTESTCD","derivation":"LBTESTCD","type":"Char","core":"Req"},
        {"name":"LBTEST","derivation":"LBTEST","type":"Char","core":"Req"},
        {"name":"LBORRES","derivation":"LBORRES","type":"Char","core":"Req"},
        {"name":"LBORRESU","derivation":"LBORRESU","type":"Char","core":"Req"},
        {"name":"LLSTRESN","derivation":"LLSTRESN","type":"Num","core":"Exp"},
        {"name":"LBNRIND","derivation":"LBNRIND","type":"Char","core":"Exp","codelist":"C78736"}
      ],
      "suppqual_variables": []
    }
  ],
  "outputs": {
    "export_formats": ["sdtm_xpt_v5","define_xml_2.1","usdm_json","adam_csv","tlf_html_png"],
    "out_dir": "out/<TS>/"
  },
  "standards": {
    "cdisc_ct_package": "sdtmct-2024-09-27.json",
    "meddra_version": "27.1"
  }
}
```

---

### How this maps to your pipeline (one‑line mappings)
| **OSB field** | **Pipeline YAML / Notebook target** |
|---|---|
| **domains[*].source_view** | `spec.source_view` used by builder to `FROM <source_view>` in SQL. |
| **domains[*].variables[].derivation** | `variables[].derivation` → SQL expression placed into `SELECT ... AS <NAME>`. |
| **domains[*].variables[].codelist** | Used by Cell 4 CT checks; maps to `codelist` references in YAML. |
| **domains[*].suppqual_variables** | Generates `suppqual_variables` section in YAML → SUPPQUAL builder (Cell 10). |
| **domains[*].postprocessing** | Flags like `[XL_SEQ]`, `[XL_DY]` → handled by postprocessing functions in Cell 6. |
| **runtime.new_run / timestamp** | Controls `NEW_RUN` and `OUT_TS_DIR` stamping (Cell 3/2). |
| **visits** | Drives TR/ADTR pseudodata generation and SoA; maps to `VISITS` used in pseudodata cell. |
| **outputs.export_formats** | Controls which export cells run (XPT, Define‑XML, ADaM, TLF). |

---

### Example: how to export OSB → YAML for the pipeline
1. **Read OSB JSON** → iterate `domains` array.  
2. For each domain produce `specs/domains/<domain>.yaml` with keys:
   - `domain`, `label`, `source_view`, `depends_on`, `variables` (list of `{name, derivation, type, core, codelist}`), `suppqual_variables`, `postprocessing`, `filter` (optional).
3. Save `specs/ct/sdtmct_2024-09-27.json` path from `standards.cdisc_ct_package`.  
4. Set environment variables from `study.project_root` and `runtime` into the notebook (PROJECT_ROOT, REUSE_TIMESTAMP, NEW_RUN, SUBJECTS_PER_TREATMENT_ARM).

---

### Recommended next steps (practical)
1. **Export script**: implement a small exporter (Python) that reads the OSB JSON and writes one YAML per domain using the same field names the notebook expects.  
2. **Smoke test**: run Cells 1–6 with `RUN_DATA_PREVIEW=True` and `RUN_ROW_COUNTS=True` to confirm SQL generation and row counts.  
3. **CT & MedDRA check**: ensure `standards.cdisc_ct_package` and `meddra_version` files are present in `specs/ct` and `specs/dictionaries/meddra`.  
4. **Iterate**: add any missing domains (EX, VS, CM) to OSB JSON as the pipeline expands; the notebook’s dependency resolver will topologically order builds.

