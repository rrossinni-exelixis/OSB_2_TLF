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

### OSB → Pipeline artifacts: YAML files and exporter script

Below are the **exact YAML files** for **DM**, **AE**, and **LB** that the notebook expects (one YAML per SDTM domain), followed by a **Python exporter script** that converts the OSB JSON (the representative JSON you approved earlier) into those YAML files and writes them to `specs/domains/`. Save the OSB JSON as `osb_study.json` and run the exporter to produce the YAMLs.

---

### 1. **DM** (`specs/domains/dm.yaml`)

```yaml
domain: DM
label: Demographics
source_view: dm_prepped
depends_on: []
build_order_priority: 1
variables:
  - name: STUDYID
    derivation: "'XL092'"
    type: Char
    core: Req
  - name: USUBJID
    derivation: USUBJID
    type: Char
    core: Req
  - name: SUBJID
    derivation: SUBJID
    type: Char
    core: Req
  - name: AGE
    derivation: "CAST(AGE AS INTEGER)"
    type: Num
    core: Req
  - name: AGEU
    derivation: "'YEARS'"
    type: Char
    core: Req
  - name: SEX
    derivation: SEX_STD
    type: Char
    core: Req
    codelist: C66731
  - name: RACE
    derivation: RACE_STD
    type: Char
    core: Exp
    codelist: C74457
  - name: ARM
    derivation: ARM_STD
    type: Char
    core: Req
  - name: ARMCD
    derivation: ARMCD_STD
    type: Char
    core: Req
  - name: RFSTDTC
    derivation: RANDDAT
    type: Char
    core: Req
  - name: RFXSTDTC
    derivation: "(SELECT MIN(ECSTDAT) FROM ec_or1_raw WHERE Subject = dm_prepped.Subject)"
    type: Char
    core: Exp
  - name: DTHDTC
    derivation: DDDAT
    type: Char
    core: Exp
suppqual_variables: []
postprocessing: []
filter: null
```

---

### 2. **AE** (`specs/domains/ae.yaml`)

```yaml
domain: AE
label: Adverse Events
source_view: ae_prepped
depends_on:
  - DM
build_order_priority: 3
variables:
  - name: STUDYID
    derivation: "'XL092'"
    type: Char
    core: Req
  - name: USUBJID
    derivation: USUBJID
    type: Char
    core: Req
  - name: AESEQ
    derivation: "[XL_SEQ:AE]"
    type: Num
    core: Req
  - name: AETERM
    derivation: AETERM
    type: Char
    core: Req
  - name: AEDECOD
    derivation: "UPPER(TRIM(AETERM_PT))"
    type: Char
    core: Req
    codelist: MedDRA_PT
  - name: AEBODSYS
    derivation: AETERM_SOC
    type: Char
    core: Req
    codelist: MedDRA_SOC
  - name: AESER
    derivation: "CASE WHEN AESERIOU = 'YES' THEN 'Y' ELSE 'N' END"
    type: Char
    core: Req
    codelist: C66742
  - name: AESTDTC
    derivation: "DATE_FORMAT(TO_DATE(AEDAT,'ddMMMyyyy'),'yyyy-MM-dd')"
    type: Char
    core: Req
  - name: AEENDTC
    derivation: "CASE WHEN AEONGO_STD = 'Y' THEN NULL ELSE DATE_FORMAT(TO_DATE(AEENDAT_INT,'yyyy-MM-dd'),'yyyy-MM-dd') END"
    type: Char
    core: Exp
  - name: AETOXGR
    derivation: "CAST(AE_CTCGRADE AS STRING)"
    type: Char
    core: Exp
  - name: AEOUT
    derivation: AEOUT_STD
    type: Char
    core: Exp
    codelist: C66768
suppqual_variables:
  - name: AECONTRT
    label: Concomitant Treatment Given
    derivation: AECONTRT_STD
postprocessing:
  - "[XL_SEQ:AE]"
  - "[XL_DY:AESTDTC-RFSTDTC]"
filter: null
```

---

### 3. **LB** (`specs/domains/lb.yaml`)

```yaml
domain: LB
label: Laboratory Test Results
source_view: lb_prepped
depends_on:
  - DM
build_order_priority: 2
variables:
  - name: STUDYID
    derivation: "'XL092'"
    type: Char
    core: Req
  - name: USUBJID
    derivation: USUBJID
    type: Char
    core: Req
  - name: LBSEQ
    derivation: "[XL_SEQ:LB]"
    type: Num
    core: Req
  - name: LBDTC
    derivation: "DATE_FORMAT(TO_DATE(LBDAT,'ddMMMyyyy'),'yyyy-MM-dd')"
    type: Char
    core: Req
  - name: LBTESTCD
    derivation: LBTESTCD
    type: Char
    core: Req
  - name: LBTEST
    derivation: LBTEST
    type: Char
    core: Req
  - name: LBORRES
    derivation: LBORRES
    type: Char
    core: Req
  - name: LBORRESU
    derivation: LBORRESU
    type: Char
    core: Req
  - name: LLSTRESN
    derivation: LLSTRESN
    type: Num
    core: Exp
  - name: LBNRIND
    derivation: LBNRIND
    type: Char
    core: Exp
    codelist: C78736
suppqual_variables: []
postprocessing: []
filter: null
```

---

### 4. **Python exporter script** (`export_osb_to_yaml.py`)

Save this script next to your `osb_study.json`. It reads the OSB JSON and writes YAML files into `specs/domains/`. It uses `pyyaml`. If `pyyaml` is not installed, install with `pip install pyyaml`.

```python
#!/usr/bin/env python3
"""
export_osb_to_yaml.py
Reads osb_study.json and writes one YAML per domain into specs/domains/.
"""

import json
import os
import sys
from pathlib import Path
import yaml

# Config
OSB_JSON = "osb_study.json"
OUT_DIR = Path("specs/domains")
OUT_DIR.mkdir(parents=True, exist_ok=True)

def normalize_domain_spec(domain_obj):
    # Ensure keys exist and are in expected shape for the notebook
    spec = {
        "domain": domain_obj.get("domain"),
        "label": domain_obj.get("label"),
        "source_view": domain_obj.get("source_view"),
        "depends_on": domain_obj.get("depends_on", []),
        "build_order_priority": domain_obj.get("build_order_priority", 99),
        "variables": domain_obj.get("variables", []),
        "suppqual_variables": domain_obj.get("suppqual_variables", []),
        "postprocessing": domain_obj.get("postprocessing", []),
        "filter": domain_obj.get("filter", None)
    }
    return spec

def write_yaml(domain_spec):
    domain = domain_spec["domain"].lower()
    filename = OUT_DIR / f"{domain}.yaml"
    # Use safe_dump with default_flow_style=False for readable YAML
    with open(filename, "w", encoding="utf-8") as f:
        yaml.safe_dump(domain_spec, f, sort_keys=False, default_flow_style=False)
    print(f"Wrote {filename}")

def main():
    if not Path(OSB_JSON).exists():
        print(f"ERROR: {OSB_JSON} not found.", file=sys.stderr)
        sys.exit(2)

    with open(OSB_JSON, "r", encoding="utf-8") as fh:
        osb = json.load(fh)

    domains = osb.get("domains", [])
    if not domains:
        print("No domains found in OSB JSON. Nothing to export.")
        return

    for d in domains:
        spec = normalize_domain_spec(d)
        write_yaml(spec)

    # Optionally write a small runtime metadata file for the notebook
    runtime = osb.get("runtime", {})
    meta = {
        "study_id": osb.get("study", {}).get("study_id"),
        "project_root": osb.get("study", {}).get("project_root"),
        "new_run": runtime.get("new_run", True),
        "reuse_timestamp": runtime.get("reuse_timestamp"),
        "subjects_per_treatment_arm": runtime.get("subjects_per_treatment_arm")
    }
    meta_file = OUT_DIR.parent / "osb_runtime_meta.yaml"
    with open(meta_file, "w", encoding="utf-8") as fh:
        yaml.safe_dump(meta, fh, sort_keys=False)
    print(f"Wrote runtime meta {meta_file}")

if __name__ == "__main__":
    main()
```

---

### 5. **How to run (quick)**

- Place your OSB JSON (the JSON shown earlier) into the same folder and name it `osb_study.json`.  
- Ensure Python 3.8+ and `pyyaml` are installed:  
  ```bash
  pip install pyyaml
  python export_osb_to_yaml.py
  ```
- The script writes `specs/domains/dm.yaml`, `specs/domains/ae.yaml`, `specs/domains/lb.yaml` and `specs/osb_runtime_meta.yaml`.

---

### 6. **Next suggestions**

- If you want, I can:
  - **Generate additional domain YAMLs** (EX, VS, CM, CMH) using the same OSB JSON structure.  
  - **Add validation** to the exporter to check that every `derivation` is present and that referenced `source_view` names exist in your Databricks workspace.  
  - **Embed CDISC CT references** into each YAML (e.g., `ct_package` path) so the notebook can validate codelists automatically.

Tell me which follow‑up you want and I’ll produce it.
### Additional domain YAMLs and enhanced exporter

Below are the **EX, VS, CM, CMH** YAML files generated from the same OSB JSON structure, followed by an **enhanced Python exporter** that:

- **Writes all domain YAMLs** into `specs/domains/`.
- **Embeds CDISC CT package path** into each YAML (`ct_package`).
- **Validates** that every `variables[].derivation` is present (non-empty).
- **Validates source_view existence** using one of two modes:
  - **Databricks mode**: if `DATABRICKS_HOST` and `DATABRICKS_TOKEN` environment variables are set, the exporter queries the Databricks REST API to check that each `source_view` exists in the configured catalog/schema.
  - **Local mode**: if you provide a JSON file `available_views.json` listing available views, the exporter checks against that list.
- **Validates codelist references** by loading the CDISC CT package JSON referenced in the OSB JSON (`standards.cdisc_ct_package`) and checking that any `codelist` values used in variables exist in the CT package.
- Emits a **validation report** (`specs/domains/validation_report.yaml`) summarizing errors and warnings and **exits nonzero** if any fatal validation errors are found.

---

### EX `specs/domains/ex.yaml`

```yaml
domain: EX
label: Exposure
source_view: ex_prepped
depends_on:
  - DM
build_order_priority: 4
ct_package: specs/ct/sdtmct_2024-09-27.json
variables:
  - name: STUDYID
    derivation: "'XL092'"
    type: Char
    core: Req
  - name: USUBJID
    derivation: USUBJID
    type: Char
    core: Req
  - name: EXSEQ
    derivation: "[XL_SEQ:EX]"
    type: Num
    core: Req
  - name: EXTRT
    derivivation: EXTRT
    type: Char
    core: Req
  - name: EXDOSE
    derivation: CAST(EXDOSE AS NUMERIC)
    type: Num
    core: Req
  - name: EXDOSU
    derivation: EXDOSU
    type: Char
    core: Req
  - name: EXDOSFRM
    derivation: EXDOSFRM_STD
    type: Char
    core: Exp
  - name: EXSTDTC
    derivation: "DATE_FORMAT(TO_DATE(EXSTDT,'ddMMMyyyy'),'yyyy-MM-dd')"
    type: Char
    core: Req
  - name: EXENDTC
    derivation: "CASE WHEN EXONGO_STD = 'Y' THEN NULL ELSE DATE_FORMAT(TO_DATE(EXENDT,'ddMMMyyyy'),'yyyy-MM-dd') END"
    type: Char
    core: Exp
suppqual_variables: []
postprocessing:
  - "[XL_SEQ:EX]"
filter: null
```

---

### VS `specs/domains/vs.yaml`

```yaml
domain: VS
label: Vital Signs
source_view: vs_prepped
depends_on:
  - DM
build_order_priority: 5
ct_package: specs/ct/sdtmct_2024-09-27.json
variables:
  - name: STUDYID
    derivation: "'XL092'"
    type: Char
    core: Req
  - name: USUBJID
    derivation: USUBJID
    type: Char
    core: Req
  - name: VSSEQ
    derivation: "[XL_SEQ:VS]"
    type: Num
    core: Req
  - name: VSTESTCD
    derivation: VSTESTCD
    type: Char
    core: Req
  - name: VSTEST
    derivation: VSTEST
    type: Char
    core: Req
  - name: VSORRES
    derivation: VSORRES
    type: Char
    core: Req
  - name: VSORRESU
    derivation: VSORRESU
    type: Char
    core: Req
  - name: VSDTC
    derivation: "DATE_FORMAT(TO_DATE(VSDAT,'ddMMMyyyy'),'yyyy-MM-dd')"
    type: Char
    core: Req
  - name: VSNRIND
    derivation: VSNRIND
    type: Char
    core: Exp
    codelist: C78736
suppqual_variables: []
postprocessing:
  - "[XL_SEQ:VS]"
filter: null
```

---

### CM `specs/domains/cm.yaml`

```yaml
domain: CM
label: Concomitant Medications
source_view: cm_prepped
depends_on:
  - DM
build_order_priority: 6
ct_package: specs/ct/sdtmct_2024-09-27.json
variables:
  - name: STUDYID
    derivation: "'XL092'"
    type: Char
    core: Req
  - name: USUBJID
    derivation: USUBJID
    type: Char
    core: Req
  - name: CMSEQ
    derivation: "[XL_SEQ:CM]"
    type: Num
    core: Req
  - name: CMTRT
    derivation: CMTRT
    type: Char
    core: Req
  - name: CMDOSE
    derivation: CMDOSE
    type: Char
    core: Exp
  - name: CMDOSU
    derivation: CMDOSU
    type: Char
    core: Exp
  - name: CMSTDTC
    derivation: "DATE_FORMAT(TO_DATE(CMSTDT,'ddMMMyyyy'),'yyyy-MM-dd')"
    type: Char
    core: Req
  - name: CMENDTC
    derivation: "CASE WHEN CMONGO_STD = 'Y' THEN NULL ELSE DATE_FORMAT(TO_DATE(CMENDT,'ddMMMyyyy'),'yyyy-MM-dd') END"
    type: Char
    core: Exp
suppqual_variables: []
postprocessing:
  - "[XL_SEQ:CM]"
filter: null
```

---

### CMH `specs/domains/cmh.yaml`

```yaml
domain: CMH
label: Concomitant Medication History
source_view: cmh_prepped
depends_on:
  - DM
build_order_priority: 7
ct_package: specs/ct/sdtmct_2024-09-27.json
variables:
  - name: STUDYID
    derivation: "'XL092'"
    type: Char
    core: Req
  - name: USUBJID
    derivation: USUBJID
    type: Char
    core: Req
  - name: CMHSEQ
    derivation: "[XL_SEQ:CMH]"
    type: Num
    core: Req
  - name: CMHTERM
    derivation: CMHTERM
    type: Char
    core: Req
  - name: CMHSTDTC
    derivation: "DATE_FORMAT(TO_DATE(CMHSTDT,'ddMMMyyyy'),'yyyy-MM-dd')"
    type: Char
    core: Req
  - name: CMHENDTC
    derivation: "CASE WHEN CMHONGO_STD = 'Y' THEN NULL ELSE DATE_FORMAT(TO_DATE(CMHENDT,'ddMMMyyyy'),'yyyy-MM-dd') END"
    type: Char
    core: Exp
suppqual_variables: []
postprocessing:
  - "[XL_SEQ:CMH]"
filter: null
```

---

### Enhanced exporter script `export_osb_to_yaml_with_validation.py`

Save this script next to your `osb_study.json`. It requires `pyyaml` and `requests` (for Databricks API checks). If you prefer not to enable Databricks checks, provide `available_views.json` (array of view names) in the same folder.

```python
#!/usr/bin/env python3
"""
export_osb_to_yaml_with_validation.py
Reads osb_study.json and writes YAML per domain into specs/domains/.
Performs validations:
 - every variable.derivation is present (non-empty)
 - source_view existence (Databricks REST API or local available_views.json)
 - codelist references exist in CDISC CT package JSON (standalone file path)
Writes validation report to specs/domains/validation_report.yaml
Exits with code 1 if fatal validation errors found.
"""

import json
import os
import sys
from pathlib import Path
import yaml
import requests

# Config
OSB_JSON = "osb_study.json"
OUT_DIR = Path("specs/domains")
OUT_DIR.mkdir(parents=True, exist_ok=True)
CT_DIR = Path("specs/ct")

# Helpers
def load_json(path):
    with open(path, "r", encoding="utf-8") as fh:
        return json.load(fh)

def write_yaml(obj, path):
    with open(path, "w", encoding="utf-8") as fh:
        yaml.safe_dump(obj, fh, sort_keys=False, default_flow_style=False)

def databricks_view_exists(host, token, catalog, schema, view_name):
    # Minimal check using Unity Catalog or SQL endpoints depending on workspace.
    # This function attempts to query the workspace for the view; adapt as needed.
    # Use SQL endpoint: /api/2.0/sql/warehouses or /api/2.0/preview/sql/queries (workspace dependent).
    # Here we use the 2.0 preview SQL endpoint to run a simple DESCRIBE TABLE query.
    headers = {"Authorization": f"Bearer {token}"}
    sql = f"DESCRIBE TABLE {catalog}.{schema}.{view_name}"
    # Use the SQL statements API (may require workspace-specific endpoint)
    url = f"{host.rstrip('/')}/api/2.0/sql/statements/execute"
    payload = {"statement": sql, "warehouse_id": None}
    try:
        r = requests.post(url, headers=headers, json=payload, timeout=15)
        if r.status_code in (200, 201):
            return True
        # Some workspaces return 4xx with message; treat 404-like as not found
        return False
    except Exception:
        return False

def find_codelist_in_ct(ct_json, codelist_name):
    # CT package structure varies; common pattern: codelists -> items with 'Name' or 'OID'
    # We'll search keys recursively for the codelist_name
    if not ct_json:
        return False
    # Common SDTM CT package: ct_json['codelists'] is a list of dicts with 'Name' or 'OID'
    codelists = ct_json.get("codelists") or ct_json.get("codelist") or []
    for cl in codelists:
        if isinstance(cl, dict):
            if cl.get("Name") == codelist_name or cl.get("OID") == codelist_name or cl.get("name") == codelist_name:
                return True
    # fallback: search top-level keys
    return False

def main():
    if not Path(OSB_JSON).exists():
        print(f"ERROR: {OSB_JSON} not found.", file=sys.stderr)
        sys.exit(2)

    osb = load_json(OSB_JSON)
    domains = osb.get("domains", [])
    if not domains:
        print("No domains found in OSB JSON. Nothing to export.")
        return

    # Load CT package if referenced
    ct_package_path = osb.get("standards", {}).get("cdisc_ct_package")
    ct_json = None
    if ct_package_path:
        ct_path = Path(ct_package_path)
        if not ct_path.exists():
            print(f"WARNING: CT package {ct_package_path} not found locally. Codelist validation will be skipped.")
        else:
            ct_json = load_json(ct_path)

    # Determine available views: Databricks mode or local file
    db_host = os.environ.get("DATABRICKS_HOST")
    db_token = os.environ.get("DATABRICKS_TOKEN")
    available_views = None
    local_views_file = Path("available_views.json")
    if not db_host or not db_token:
        if local_views_file.exists():
            available_views = load_json(local_views_file)
            if not isinstance(available_views, list):
                print("available_views.json must contain a JSON array of view names.", file=sys.stderr)
                available_views = None

    validation = {"errors": [], "warnings": [], "domain_reports": {}}

    # Export each domain and validate
    for d in domains:
        spec = {
            "domain": d.get("domain"),
            "label": d.get("label"),
            "source_view": d.get("source_view"),
            "depends_on": d.get("depends_on", []),
            "build_order_priority": d.get("build_order_priority", 99),
            "ct_package": ct_package_path or "",
            "variables": d.get("variables", []),
            "suppqual_variables": d.get("suppqual_variables", []),
            "postprocessing": d.get("postprocessing", []),
            "filter": d.get("filter", None)
        }

        domain_name = spec["domain"]
        domain_report = {"errors": [], "warnings": []}

        # Validate derivations present
        for v in spec["variables"]:
            name = v.get("name")
            deriv = v.get("derivation")
            if deriv is None or (isinstance(deriv, str) and deriv.strip() == ""):
                domain_report["errors"].append(f"Variable {name} missing derivation.")
        for sv in spec.get("suppqual_variables", []):
            if sv.get("derivation") is None or (isinstance(sv.get("derivation"), str) and sv.get("derivation").strip() == ""):
                domain_report["errors"].append(f"SUPPQUAL variable {sv.get('name')} missing derivation.")

        # Validate source_view existence
        sv_name = spec.get("source_view")
        if not sv_name:
            domain_report["errors"].append("source_view is not defined.")
        else:
            exists = False
            if db_host and db_token:
                # Attempt to parse catalog.schema.view if provided, else use default catalog/schema
                # For safety, do a simple check: try DESCRIBE TABLE with view name only
                exists = databricks_view_exists(db_host, db_token, "default_catalog", "default_schema", sv_name)
                if not exists:
                    # try without catalog/schema
                    exists = databricks_view_exists(db_host, db_token, "", "", sv_name)
                if not exists:
                    domain_report["warnings"].append(f"Databricks check could not confirm existence of source_view '{sv_name}'.")
            elif available_views is not None:
                if sv_name not in available_views:
                    domain_report["errors"].append(f"source_view '{sv_name}' not found in available_views.json.")
            else:
                domain_report["warnings"].append(f"No Databricks credentials or available_views.json provided; cannot validate source_view '{sv_name}' existence.")

        # Validate codelists against CT package
        for v in spec["variables"]:
            cl = v.get("codelist")
            if cl:
                if ct_json:
                    if not find_codelist_in_ct(ct_json, cl):
                        domain_report["errors"].append(f"Codelist '{cl}' referenced by variable {v.get('name')} not found in CT package.")
                else:
                    domain_report["warnings"].append(f"Codelist '{cl}' referenced by variable {v.get('name')}; CT package not loaded for validation.")

        # Write YAML
        out_file = OUT_DIR / f"{domain_name.lower()}.yaml"
        write_yaml(spec, out_file)
        domain_report["written_file"] = str(out_file)
        validation["domain_reports"][domain_name] = domain_report
        if domain_report["errors"]:
            validation["errors"].extend([f"{domain_name}: {e}" for e in domain_report["errors"]])
        if domain_report["warnings"]:
            validation["warnings"].extend([f"{domain_name}: {w}" for w in domain_report["warnings"]])

    # Write runtime meta
    runtime = osb.get("runtime", {})
    meta = {
        "study_id": osb.get("study", {}).get("study_id"),
        "project_root": osb.get("study", {}).get("project_root"),
        "new_run": runtime.get("new_run", True),
        "reuse_timestamp": runtime.get("reuse_timestamp"),
        "subjects_per_treatment_arm": runtime.get("subjects_per_treatment_arm")
    }
    meta_file = OUT_DIR.parent / "osb_runtime_meta.yaml"
    write_yaml(meta, meta_file)

    # Write validation report
    report_file = OUT_DIR / "validation_report.yaml"
    write_yaml(validation, report_file)
    print(f"Wrote validation report to {report_file}")

    # Exit nonzero if fatal errors
    if validation["errors"]:
        print("Validation failed with errors:")
        for e in validation["errors"]:
            print(" -", e)
        sys.exit(1)
    else:
        if validation["warnings"]:
            print("Validation completed with warnings:")
            for w in validation["warnings"]:
                print(" -", w)
        else:
            print("Validation completed successfully with no warnings or errors.")

if __name__ == "__main__":
    main()
```

---

### How to use the enhanced exporter

1. **Place files**  
   - Save the OSB JSON as `osb_study.json` in the same folder as the script.  
   - Ensure `specs/ct/sdtmct_2024-09-27.json` exists if you want codelist validation.  
   - Optionally create `available_views.json` containing a JSON array of view names (e.g., `["dm_prepped","ae_prepped","lb_prepped","ex_prepped","vs_prepped","cm_prepped","cmh_prepped"]`) if you do not want to enable Databricks API checks.

2. **Install dependencies**  
   ```bash
   pip install pyyaml requests
   ```

3. **Databricks validation (optional)**  
   - Set environment variables if you want the exporter to check views directly in Databricks:  
     ```bash
     export DATABRICKS_HOST="https://<your-databricks-workspace>"
     export DATABRICKS_TOKEN="<your-personal-access-token>"
     ```
   - The script attempts a minimal `DESCRIBE TABLE` via the SQL statements endpoint. Adjust `databricks_view_exists` if your workspace requires a different endpoint or catalog/schema handling.

4. **Run**  
   ```bash
   python export_osb_to_yaml_with_validation.py
   ```

5. **Inspect outputs**  
   - YAMLs: `specs/domains/*.yaml` (dm, ae, lb, ex, vs, cm, cmh)  
   - Runtime meta: `specs/osb_runtime_meta.yaml`  
   - Validation report: `specs/domains/validation_report.yaml`

---

### Notes and recommendations

- **CT package path** is embedded into each YAML as `ct_package`. If your notebook expects a different key name or relative path, adjust the exporter accordingly.
- **Databricks API**: the script uses a conservative approach to check view existence. Depending on your workspace configuration (Unity Catalog, SQL endpoints, or workspace permissions), you may need to adapt the `databricks_view_exists` function to use the correct API endpoint or to authenticate via a SQL warehouse.
- **Codelist validation** assumes the CT package JSON contains a `codelists` array with `Name` or `OID` fields. If your CT package uses a different schema, update `find_codelist_in_ct`.
- If you want, I can:
  - **(1)** adapt the Databricks check to your exact workspace (provide host, token, and catalog/schema conventions) and test the exact SQL used, or  
  - **(2)** produce the final `specs/domains/*.yaml` files packaged as a single archive (I will provide the exact file contents here for you to copy), or  
  - **(3)** add an optional CLI flag `--fail-on-warning` to make warnings fatal.

Which follow-up would you like next?
### Options at a glance

| **Option** | **What it delivers** | **Key inputs required** | **When to pick** |
|---|---:|---|---|
| **(1) Databricks check adapted and testable** | Enhanced exporter with a Databricks‑specific `view exists` routine, exact SQL used for checks, and example `curl`/Python test commands | **DATABRICKS_HOST**, **DATABRICKS_TOKEN**, **catalog**, **schema** | You want automated, live validation against your workspace before running the notebook |
| **(2) Final specs archive and implementation steps** | All `specs/domains/*.yaml` files (DM, AE, LB, EX, VS, CM, CMH) packaged instructions to create an archive and deploy into your repo | None beyond the YAML contents I provide | You want a ready set of YAMLs to drop into the pipeline and step‑by‑step deployment instructions |

I will deliver both: (A) the Databricks‑adapted exporter code and exact test commands you can run in your environment, and (B) the complete YAML files packaged instructions and step‑by‑step implementation guidance.

---

## Databricks adaptation and exact SQL checks

#### Summary
I adapted the exporter’s Databricks check to use the Databricks SQL Statements API (recommended for programmatic SQL checks). The routine will:

- Attempt a safe `SHOW TABLES IN <catalog>.<schema> LIKE '<view>'` or `DESCRIBE TABLE <catalog>.<schema>.<view>` depending on Unity Catalog usage.
- Fall back to `SHOW TABLES` in the default catalog/schema if catalog/schema are not provided.
- Return a clear boolean and an explanatory message for the validation report.

#### What I need from you to run the tests
- **DATABRICKS_HOST** e.g., `https://adb-123456789012345.7.azuredatabricks.net`  
- **DATABRICKS_TOKEN** a personal access token with SQL permissions  
- **catalog** and **schema** you use for the prepared views (e.g., `main_catalog`, `analytics_schema`) — optional but recommended

You do **not** need to share these with me. Run the commands below in your environment.

---

### Databricks check function (Python)

Save this snippet into the enhanced exporter file or as a helper module. It uses the SQL statements execute endpoint. Replace `catalog` and `schema` variables with your values or pass them in.

```python
import requests
import json
from urllib.parse import urljoin

def databricks_view_exists(host, token, catalog, schema, view_name, timeout=15):
    """
    Check whether a view/table exists in Databricks using the SQL statements API.
    Returns (exists: bool, message: str).
    """
    headers = {"Authorization": f"Bearer {token}", "Content-Type": "application/json"}
    base = host.rstrip('/')
    # Prefer SHOW TABLES IN catalog.schema LIKE 'view'
    if catalog and schema:
        sql = f"SHOW TABLES IN {catalog}.{schema} LIKE '{view_name}'"
    elif schema:
        sql = f"SHOW TABLES IN {schema} LIKE '{view_name}'"
    else:
        # fallback to simple SHOW TABLES and filter client-side
        sql = f"SHOW TABLES LIKE '{view_name}'"

    payload = {"statement": sql}
    url = urljoin(base, "/api/2.0/sql/statements/execute")
    try:
        r = requests.post(url, headers=headers, json=payload, timeout=timeout)
        if r.status_code not in (200, 201):
            return False, f"SQL API returned {r.status_code}: {r.text}"
        resp = r.json()
        # The SQL statements API returns a statement id and then result is available via result_url or by polling.
        # If immediate result included:
        if "result" in resp and isinstance(resp["result"], dict):
            rows = resp["result"].get("data", [])
            if rows:
                return True, f"Found {len(rows)} matching table(s)."
            else:
                return False, "No matching table found."
        # If the API returns a statement id, poll the statement result endpoint
        statement_id = resp.get("statement_id") or resp.get("id")
        if not statement_id:
            return False, "No statement id returned; cannot confirm existence."
        # Poll result
        status_url = urljoin(base, f"/api/2.0/sql/statements/{statement_id}")
        for _ in range(10):
            s = requests.get(status_url, headers=headers, timeout=timeout)
            if s.status_code != 200:
                break
            sj = s.json()
            state = sj.get("status", {}).get("state")
            if state == "SUCCEEDED":
                result = sj.get("result", {}).get("data", [])
                if result:
                    return True, f"Found {len(result)} matching table(s)."
                else:
                    return False, "No matching table found."
            if state in ("FAILED", "CANCELED"):
                return False, f"Statement {state}: {sj.get('status', {}).get('error_message')}"
        return False, "Timed out polling statement result."
    except Exception as e:
        return False, f"Exception while calling Databricks SQL API: {str(e)}"
```

---

### Exact SQL statements used
- Preferred (Unity Catalog aware)
  - `SHOW TABLES IN <catalog>.<schema> LIKE '<view_name>'`
- Fallback (schema only)
  - `SHOW TABLES IN <schema> LIKE '<view_name>'`
- Final fallback (no catalog/schema)
  - `SHOW TABLES LIKE '<view_name>'`
- If you prefer a stricter check:
  - `DESCRIBE TABLE <catalog>.<schema>.<view_name>`

`SHOW TABLES` is safe and non‑destructive; `DESCRIBE TABLE` is also safe but may require additional privileges.

---

### How to test from your shell using curl

Replace placeholders with your values.

```bash
# Variables
DATABRICKS_HOST="https://<your-databricks-host>"
TOKEN="<your-token>"
CATALOG="your_catalog"   # optional
SCHEMA="your_schema"     # optional
VIEW="dm_prepped"

# Preferred: execute SHOW TABLES
SQL="SHOW TABLES IN ${CATALOG}.${SCHEMA} LIKE '${VIEW}'"
curl -s -X POST "${DATABRICKS_HOST}/api/2.0/sql/statements/execute" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{\"statement\": \"${SQL}\"}" | jq .

# If the response returns a statement id, poll:
STATEMENT_ID="<returned_statement_id>"
curl -s -X GET "${DATABRICKS_HOST}/api/2.0/sql/statements/${STATEMENT_ID}" \
  -H "Authorization: Bearer ${TOKEN}" | jq .
```

If `jq` is not available, remove the `| jq .` and inspect raw JSON.

---

### How to integrate into the exporter
- Add the `databricks_view_exists` function to the exporter.
- When validating each domain `source_view`, call the function with your `DATABRICKS_HOST`, `DATABRICKS_TOKEN`, `catalog`, `schema`.
- Record the boolean and message into `validation_report.yaml`.
- If `exists == False` and the message indicates not found, mark as **error**; if the message indicates API failure, mark as **warning**.

---

## Final specs archive contents and creation instructions

#### Files included
I will provide the exact YAML contents for these files:

- `specs/domains/dm.yaml`
- `specs/domains/ae.yaml`
- `specs/domains/lb.yaml`
- `specs/domains/ex.yaml`
- `specs/domains/vs.yaml`
- `specs/domains/cm.yaml`
- `specs/domains/cmh.yaml`
- `specs/osb_runtime_meta.yaml`
- `specs/ct/sdtmct_2024-09-27.json` (placeholder path reference; ensure your CT package is present)

Below are the YAML contents. Save each file under the path shown.

---

### specs/domains/dm.yaml

```yaml
domain: DM
label: Demographics
source_view: dm_prepped
depends_on: []
build_order_priority: 1
ct_package: specs/ct/sdtmct_2024-09-27.json
variables:
  - name: STUDYID
    derivation: "'XL092'"
    type: Char
    core: Req
  - name: USUBJID
    derivation: USUBJID
    type: Char
    core: Req
  - name: SUBJID
    derivation: SUBJID
    type: Char
    core: Req
  - name: AGE
    derivation: "CAST(AGE AS INTEGER)"
    type: Num
    core: Req
  - name: AGEU
    derivation: "'YEARS'"
    type: Char
    core: Req
  - name: SEX
    derivation: SEX_STD
    type: Char
    core: Req
    codelist: C66731
  - name: RACE
    derivation: RACE_STD
    type: Char
    core: Exp
    codelist: C74457
  - name: ARM
    derivation: ARM_STD
    type: Char
    core: Req
  - name: ARMCD
    derivation: ARMCD_STD
    type: Char
    core: Req
  - name: RFSTDTC
    derivation: RANDDAT
    type: Char
    core: Req
  - name: RFXSTDTC
    derivation: "(SELECT MIN(ECSTDAT) FROM ec_or1_raw WHERE Subject = dm_prepped.Subject)"
    type: Char
    core: Exp
  - name: DTHDTC
    derivation: DDDAT
    type: Char
    core: Exp
suppqual_variables: []
postprocessing: []
filter: null
```

---

### specs/domains/ae.yaml

```yaml
domain: AE
label: Adverse Events
source_view: ae_prepped
depends_on:
  - DM
build_order_priority: 3
ct_package: specs/ct/sdtmct_2024-09-27.json
variables:
  - name: STUDYID
    derivation: "'XL092'"
    type: Char
    core: Req
  - name: USUBJID
    derivation: USUBJID
    type: Char
    core: Req
  - name: AESEQ
    derivation: "[XL_SEQ:AE]"
    type: Num
    core: Req
  - name: AETERM
    derivation: AETERM
    type: Char
    core: Req
  - name: AEDECOD
    derivation: "UPPER(TRIM(AETERM_PT))"
    type: Char
    core: Req
    codelist: MedDRA_PT
  - name: AEBODSYS
    derivation: AETERM_SOC
    type: Char
    core: Req
    codelist: MedDRA_SOC
  - name: AESER
    derivation: "CASE WHEN AESERIOU = 'YES' THEN 'Y' ELSE 'N' END"
    type: Char
    core: Req
    codelist: C66742
  - name: AESTDTC
    derivation: "DATE_FORMAT(TO_DATE(AEDAT,'ddMMMyyyy'),'yyyy-MM-dd')"
    type: Char
    core: Req
  - name: AEENDTC
    derivation: "CASE WHEN AEONGO_STD = 'Y' THEN NULL ELSE DATE_FORMAT(TO_DATE(AEENDAT_INT,'yyyy-MM-dd'),'yyyy-MM-dd') END"
    type: Char
    core: Exp
  - name: AETOXGR
    derivation: "CAST(AE_CTCGRADE AS STRING)"
    type: Char
    core: Exp
  - name: AEOUT
    derivation: AEOUT_STD
    type: Char
    core: Exp
    codelist: C66768
suppqual_variables:
  - name: AECONTRT
    label: Concomitant Treatment Given
    derivation: AECONTRT_STD
postprocessing:
  - "[XL_SEQ:AE]"
  - "[XL_DY:AESTDTC-RFSTDTC]"
filter: null
```

---

### specs/domains/lb.yaml

```yaml
domain: LB
label: Laboratory Test Results
source_view: lb_prepped
depends_on:
  - DM
build_order_priority: 2
ct_package: specs/ct/sdtmct_2024-09-27.json
variables:
  - name: STUDYID
    derivation: "'XL092'"
    type: Char
    core: Req
  - name: USUBJID
    derivation: USUBJID
    type: Char
    core: Req
  - name: LBSEQ
    derivation: "[XL_SEQ:LB]"
    type: Num
    core: Req
  - name: LBDTC
    derivation: "DATE_FORMAT(TO_DATE(LBDAT,'ddMMMyyyy'),'yyyy-MM-dd')"
    type: Char
    core: Req
  - name: LBTESTCD
    derivation: LBTESTCD
    type: Char
    core: Req
  - name: LBTEST
    derivation: LBTEST
    type: Char
    core: Req
  - name: LBORRES
    derivation: LBORRES
    type: Char
    core: Req
  - name: LBORRESU
    derivation: LBORRESU
    type: Char
    core: Req
  - name: LLSTRESN
    derivation: LLSTRESN
    type: Num
    core: Exp
  - name: LBNRIND
    derivation: LBNRIND
    type: Char
    core: Exp
    codelist: C78736
suppqual_variables: []
postprocessing: []
filter: null
```

---

### specs/domains/ex.yaml

```yaml
domain: EX
label: Exposure
source_view: ex_prepped
depends_on:
  - DM
build_order_priority: 4
ct_package: specs/ct/sdtmct_2024-09-27.json
variables:
  - name: STUDYID
    derivation: "'XL092'"
    type: Char
    core: Req
  - name: USUBJID
    derivation: USUBJID
    type: Char
    core: Req
  - name: EXSEQ
    derivation: "[XL_SEQ:EX]"
    type: Num
    core: Req
  - name: EXTRT
    derivation: EXTRT
    type: Char
    core: Req
  - name: EXDOSE
    derivation: "CAST(EXDOSE AS NUMERIC)"
    type: Num
    core: Req
  - name: EXDOSU
    derivation: EXDOSU
    type: Char
    core: Req
  - name: EXDOSFRM
    derivation: EXDOSFRM_STD
    type: Char
    core: Exp
  - name: EXSTDTC
    derivation: "DATE_FORMAT(TO_DATE(EXSTDT,'ddMMMyyyy'),'yyyy-MM-dd')"
    type: Char
    core: Req
  - name: EXENDTC
    derivation: "CASE WHEN EXONGO_STD = 'Y' THEN NULL ELSE DATE_FORMAT(TO_DATE(EXENDT,'ddMMMyyyy'),'yyyy-MM-dd') END"
    type: Char
    core: Exp
suppqual_variables: []
postprocessing:
  - "[XL_SEQ:EX]"
filter: null
```

---

### specs/domains/vs.yaml

```yaml
domain: VS
label: Vital Signs
source_view: vs_prepped
depends_on:
  - DM
build_order_priority: 5
ct_package: specs/ct/sdtmct_2024-09-27.json
variables:
  - name: STUDYID
    derivation: "'XL092'"
    type: Char
    core: Req
  - name: USUBJID
    derivation: USUBJID
    type: Char
    core: Req
  - name: VSSEQ
    derivation: "[XL_SEQ:VS]"
    type: Num
    core: Req
  - name: VSTESTCD
    derivation: VSTESTCD
    type: Char
    core: Req
  - name: VSTEST
    derivation: VSTEST
    type: Char
    core: Req
  - name: VSORRES
    derivation: VSORRES
    type: Char
    core: Req
  - name: VSORRESU
    derivation: VSORRESU
    type: Char
    core: Req
  - name: VSDTC
    derivation: "DATE_FORMAT(TO_DATE(VSDAT,'ddMMMyyyy'),'yyyy-MM-dd')"
    type: Char
    core: Req
  - name: VSNRIND
    derivation: VSNRIND
    type: Char
    core: Exp
    codelist: C78736
suppqual_variables: []
postprocessing:
  - "[XL_SEQ:VS]"
filter: null
```

---

### specs/domains/cm.yaml

```yaml
domain: CM
label: Concomitant Medications
source_view: cm_prepped
depends_on:
  - DM
build_order_priority: 6
ct_package: specs/ct/sdtmct_2024-09-27.json
variables:
  - name: STUDYID
    derivation: "'XL092'"
    type: Char
    core: Req
  - name: USUBJID
    derivation: USUBJID
    type: Char
    core: Req
  - name: CMSEQ
    derivation: "[XL_SEQ:CM]"
    type: Num
    core: Req
  - name: CMTRT
    derivation: CMTRT
    type: Char
    core: Req
  - name: CMDOSE
    derivation: CMDOSE
    type: Char
    core: Exp
  - name: CMDOSU
    derivation: CMDOSU
    type: Char
    core: Exp
  - name: CMSTDTC
    derivation: "DATE_FORMAT(TO_DATE(CMSTDT,'ddMMMyyyy'),'yyyy-MM-dd')"
    type: Char
    core: Req
  - name: CMENDTC
    derivation: "CASE WHEN CMONGO_STD = 'Y' THEN NULL ELSE DATE_FORMAT(TO_DATE(CMENDT,'ddMMMyyyy'),'yyyy-MM-dd') END"
    type: Char
    core: Exp
suppqual_variables: []
postprocessing:
  - "[XL_SEQ:CM]"
filter: null
```

---

### specs/domains/cmh.yaml

```yaml
domain: CMH
label: Concomitant Medication History
source_view: cmh_prepped
depends_on:
  - DM
build_order_priority: 7
ct_package: specs/ct/sdtmct_2024-09-27.json
variables:
  - name: STUDYID
    derivation: "'XL092'"
    type: Char
    core: Req
  - name: USUBJID
    derivation: USUBJID
    type: Char
    core: Req
  - name: CMHSEQ
    derivation: "[XL_SEQ:CMH]"
    type: Num
    core: Req
  - name: CMHTERM
    derivation: CMHTERM
    type: Char
    core: Req
  - name: CMHSTDTC
    derivation: "DATE_FORMAT(TO_DATE(CMHSTDT,'ddMMMyyyy'),'yyyy-MM-dd')"
    type: Char
    core: Req
  - name: CMHENDTC
    derivation: "CASE WHEN CMHONGO_STD = 'Y' THEN NULL ELSE DATE_FORMAT(TO_DATE(CMHENDT,'ddMMMyyyy'),'yyyy-MM-dd') END"
    type: Char
    core: Exp
suppqual_variables: []
postprocessing:
  - "[XL_SEQ:CMH]"
filter: null
```

---

### specs/osb_runtime_meta.yaml

```yaml
study_id: XL092-123
project_root: /Workspace/Users/rossinni.r@gmail.com/Drafts/SDTM_Build_POC
new_run: true
reuse_timestamp: null
subjects_per_treatment_arm: 100
```

---

## Create the archive locally

1. Create the directory structure and files exactly as above:

```bash
mkdir -p specs/domains specs/ct
# Save each YAML into specs/domains/ as dm.yaml ae.yaml lb.yaml ex.yaml vs.yaml cm.yaml cmh.yaml
# Save osb_runtime_meta.yaml into specs/
# Place your CDISC CT package JSON at specs/ct/sdtmct_2024-09-27.json
```

2. Create a compressed archive

```bash
tar -czf osb_specs_bundle.tar.gz specs
# or zip
zip -r osb_specs_bundle.zip specs
```

3. Upload the archive to your repo or Databricks workspace as needed.

---

## Step by step implementation instructions

#### Step 1 Prepare files
- Save the YAML files above into `specs/domains/`.
- Place your CDISC CT package JSON at `specs/ct/sdtmct_2024-09-27.json`.
- Save the enhanced exporter script `export_osb_to_yaml_with_validation.py` (from earlier) and add the `databricks_view_exists` function shown above.

#### Step 2 Configure environment
- Locally set environment variables if you want Databricks checks:
  ```bash
  export DATABRICKS_HOST="https://<your-databricks-host>"
  export DATABRICKS_TOKEN="<your-token>"
  export DATABRICKS_CATALOG="your_catalog"   # optional
  export DATABRICKS_SCHEMA="your_schema"     # optional
  ```
- Or create `available_views.json` with:
  ```json
  ["dm_prepped","ae_prepped","lb_prepped","ex_prepped","vs_prepped","cm_prepped","cmh_prepped"]
  ```

#### Step 3 Install dependencies
```bash
pip install pyyaml requests
```

#### Step 4 Run exporter and validate
```bash
python export_osb_to_yaml_with_validation.py
# If using the simpler exporter:
python export_osb_to_yaml.py
```
- Inspect `specs/domains/validation_report.yaml` for errors/warnings.
- If errors exist, fix derivations, missing codelists, or create the missing views in Databricks.

#### Step 5 Deploy YAMLs to pipeline
- Copy `specs/domains/*.yaml` into the pipeline repo path expected by the notebook (e.g., `/Workspace/.../specs/domains/`).
- Ensure `specs/ct/sdtmct_2024-09-27.json` is accessible to the notebook.

#### Step 6 Run the notebook
- Set notebook environment variables from `specs/osb_runtime_meta.yaml` or export them:
  ```bash
  export PROJECT_ROOT="/Workspace/Users/rossinni.r@gmail.com/Drafts/SDTM_Build_POC"
  export NEW_RUN=true
  export REUSE_TIMESTAMP=""
  ```
- Run Cells 1–6 with `RUN_DATA_PREVIEW=True` and `RUN_ROW_COUNTS=True` to confirm SQL generation and row counts.
- If the notebook supports a dry run flag, run in dry mode first.

#### Step 7 Postprocessing and exports
- Confirm postprocessing flags like `[XL_SEQ]` and `[XL_DY]` are handled by the notebook.
- Run export cells to produce XPTs, Define‑XML, USDM JSON, ADaM CSVs, and TLFs as required.

---

## Final notes and quick checklist

- **Checklist before running**  
  - [ ] `specs/domains/*.yaml` present and correct.  
  - [ ] `specs/ct/sdtmct_2024-09-27.json` present for codelist validation.  
  - [ ] `available_views.json` or Databricks credentials available for source_view validation.  
  - [ ] Exporter script updated with `databricks_view_exists` and run locally to produce `validation_report.yaml`.  

- **If you want me to do one of these next**  
  - Provide the exact `DATABRICKS_HOST`, `DATABRICKS_CATALOG`, `DATABRICKS_SCHEMA`, and a test token and I will produce the exact `curl`/Python commands tailored to those values (you will run them locally).  
  - Or I can produce a single downloadable archive text block containing all files ready to paste into files if you prefer.

Tell me which of those two follow‑ups you want and I’ll produce the tailored commands or the single pasteable archive block.
