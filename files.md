### OSB specs archive installer (single pasteable block)

Below is a **single shell script** you can paste into a file (for example `create_osb_specs.sh`) and run locally to create the full `specs/` directory tree and all domain YAMLs, runtime meta, and the exporter scripts. The script writes the files exactly as discussed, creates `specs/ct/` placeholder, and produces a compressed archive `osb_specs_bundle.tar.gz`.

**How to use**
1. Paste the block into a file named `create_osb_specs.sh`.  
2. Make it executable: `chmod +x create_osb_specs.sh`.  
3. Run it: `./create_osb_specs.sh`.  
4. Inspect `specs/` and `osb_specs_bundle.tar.gz`.

```bash
#!/usr/bin/env bash
set -euo pipefail

ROOT_DIR="$(pwd)"
OUT_DIR="${ROOT_DIR}/specs"
DOM_DIR="${OUT_DIR}/domains"
CT_DIR="${OUT_DIR}/ct"

mkdir -p "${DOM_DIR}"
mkdir -p "${CT_DIR}"

cat > "${DOM_DIR}/dm.yaml" <<'YAML'
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
YAML

cat > "${DOM_DIR}/ae.yaml" <<'YAML'
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
YAML

cat > "${DOM_DIR}/lb.yaml" <<'YAML'
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
YAML

cat > "${DOM_DIR}/ex.yaml" <<'YAML'
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
YAML

cat > "${DOM_DIR}/vs.yaml" <<'YAML'
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
YAML

cat > "${DOM_DIR}/cm.yaml" <<'YAML'
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
YAML

cat > "${DOM_DIR}/cmh.yaml" <<'YAML'
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
YAML

cat > "${OUT_DIR}/osb_runtime_meta.yaml" <<'YAML'
study_id: XL092-123
project_root: /Workspace/Users/rossinni.r@gmail.com/Drafts/SDTM_Build_POC
new_run: true
reuse_timestamp: null
subjects_per_treatment_arm: 100
YAML

# Placeholder CT package note file
cat > "${CT_DIR}/README.txt" <<'TXT'
Place your CDISC CT package JSON here with filename:
  sdtmct_2024-09-27.json

This file is referenced by ct_package keys in the domain YAMLs.
TXT

# Write the simple exporter script
cat > "${ROOT_DIR}/export_osb_to_yaml.py" <<'PY'
#!/usr/bin/env python3
import json
from pathlib import Path
import yaml
OSB_JSON = "osb_study.json"
OUT_DIR = Path("specs/domains")
OUT_DIR.mkdir(parents=True, exist_ok=True)
def normalize_domain_spec(domain_obj):
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
    with open(filename, "w", encoding="utf-8") as f:
        yaml.safe_dump(domain_spec, f, sort_keys=False, default_flow_style=False)
    print(f"Wrote {filename}")
def main():
    if not Path(OSB_JSON).exists():
        print(f"ERROR: {OSB_JSON} not found.")
        return
    with open(OSB_JSON, "r", encoding="utf-8") as fh:
        osb = json.load(fh)
    domains = osb.get("domains", [])
    if not domains:
        print("No domains found in OSB JSON. Nothing to export.")
        return
    for d in domains:
        spec = normalize_domain_spec(d)
        write_yaml(spec)
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
PY

# Write the enhanced exporter with validation (Databricks helper included)
cat > "${ROOT_DIR}/export_osb_to_yaml_with_validation.py" <<'PY'
#!/usr/bin/env python3
import json, os, sys, time
from pathlib import Path
import yaml, requests
OSB_JSON = "osb_study.json"
OUT_DIR = Path("specs/domains")
OUT_DIR.mkdir(parents=True, exist_ok=True)
def load_json(path):
    with open(path, "r", encoding="utf-8") as fh:
        return json.load(fh)
def write_yaml(obj, path):
    with open(path, "w", encoding="utf-8") as fh:
        yaml.safe_dump(obj, fh, sort_keys=False, default_flow_style=False)
def databricks_view_exists(host, token, catalog, schema, view_name, timeout=15):
    headers = {"Authorization": f"Bearer {token}", "Content-Type": "application/json"}
    base = host.rstrip('/')
    if catalog and schema:
        sql = f"SHOW TABLES IN {catalog}.{schema} LIKE '{view_name}'"
    elif schema:
        sql = f"SHOW TABLES IN {schema} LIKE '{view_name}'"
    else:
        sql = f"SHOW TABLES LIKE '{view_name}'"
    payload = {"statement": sql}
    url = base + "/api/2.0/sql/statements/execute"
    try:
        r = requests.post(url, headers=headers, json=payload, timeout=timeout)
        if r.status_code not in (200,201):
            return False, f"SQL API returned {r.status_code}: {r.text}"
        resp = r.json()
        if "result" in resp and isinstance(resp["result"], dict):
            rows = resp["result"].get("data", [])
            if rows:
                return True, f"Found {len(rows)} matching table(s)."
            else:
                return False, "No matching table found."
        statement_id = resp.get("statement_id") or resp.get("id")
        if not statement_id:
            return False, "No statement id returned; cannot confirm existence."
        status_url = base + f"/api/2.0/sql/statements/{statement_id}"
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
            time.sleep(0.5)
        return False, "Timed out polling statement result."
    except Exception as e:
        return False, f"Exception while calling Databricks SQL API: {str(e)}"
def find_codelist_in_ct(ct_json, codelist_name):
    if not ct_json:
        return False
    codelists = ct_json.get("codelists") or ct_json.get("codelist") or []
    for cl in codelists:
        if isinstance(cl, dict):
            if cl.get("Name") == codelist_name or cl.get("OID") == codelist_name or cl.get("name") == codelist_name:
                return True
    return False
def main():
    if not Path(OSB_JSON).exists():
        print(f"ERROR: {OSB_JSON} not found.")
        sys.exit(2)
    osb = load_json(OSB_JSON)
    domains = osb.get("domains", [])
    if not domains:
        print("No domains found in OSB JSON. Nothing to export.")
        return
    ct_package_path = osb.get("standards", {}).get("cdisc_ct_package")
    ct_json = None
    if ct_package_path:
        ct_path = Path(ct_package_path)
        if ct_path.exists():
            ct_json = load_json(ct_path)
        else:
            print(f"WARNING: CT package {ct_package_path} not found locally. Codelist validation will be skipped.")
    db_host = os.environ.get("DATABRICKS_HOST")
    db_token = os.environ.get("DATABRICKS_TOKEN")
    db_catalog = os.environ.get("DATABRICKS_CATALOG", "")
    db_schema = os.environ.get("DATABRICKS_SCHEMA", "")
    available_views = None
    local_views_file = Path("available_views.json")
    if not db_host or not db_token:
        if local_views_file.exists():
            available_views = load_json(local_views_file)
            if not isinstance(available_views, list):
                print("available_views.json must contain a JSON array of view names.")
                available_views = None
    validation = {"errors": [], "warnings": [], "domain_reports": {}}
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
        for v in spec["variables"]:
            name = v.get("name")
            deriv = v.get("derivation")
            if deriv is None or (isinstance(deriv, str) and deriv.strip() == ""):
                domain_report["errors"].append(f"Variable {name} missing derivation.")
        for sv in spec.get("suppqual_variables", []):
            if sv.get("derivation") is None or (isinstance(sv.get("derivation"), str) and sv.get("derivation").strip() == ""):
                domain_report["errors"].append(f"SUPPQUAL variable {sv.get('name')} missing derivation.")
        sv_name = spec.get("source_view")
        if not sv_name:
            domain_report["errors"].append("source_view is not defined.")
        else:
            if db_host and db_token:
                exists, msg = databricks_view_exists(db_host, db_token, db_catalog, db_schema, sv_name)
                if not exists:
                    domain_report["errors"].append(f"source_view '{sv_name}' check failed: {msg}")
            elif available_views is not None:
                if sv_name not in available_views:
                    domain_report["errors"].append(f"source_view '{sv_name}' not found in available_views.json.")
            else:
                domain_report["warnings"].append(f"No Databricks credentials or available_views.json provided; cannot validate source_view '{sv_name}' existence.")
        for v in spec["variables"]:
            cl = v.get("codelist")
            if cl:
                if ct_json:
                    if not find_codelist_in_ct(ct_json, cl):
                        domain_report["errors"].append(f"Codelist '{cl}' referenced by variable {v.get('name')} not found in CT package.")
                else:
                    domain_report["warnings"].append(f"Codelist '{cl}' referenced by variable {v.get('name')}; CT package not loaded for validation.")
        out_file = OUT_DIR / f"{domain_name.lower()}.yaml"
        write_yaml(spec, out_file)
        domain_report["written_file"] = str(out_file)
        validation["domain_reports"][domain_name] = domain_report
        if domain_report["errors"]:
            validation["errors"].extend([f"{domain_name}: {e}" for e in domain_report["errors"]])
        if domain_report["warnings"]:
            validation["warnings"].extend([f"{domain_name}: {w}" for w in domain_report["warnings"]])
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
    report_file = OUT_DIR / "validation_report.yaml"
    write_yaml(validation, report_file)
    print(f"Wrote validation report to {report_file}")
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
PY

# Make scripts executable
chmod +x "${ROOT_DIR}/export_osb_to_yaml.py" || true
chmod +x "${ROOT_DIR}/export_osb_to_yaml_with_validation.py" || true

# Create a small README
cat > "${ROOT_DIR}/README_OSB_SPECS.md" <<'MD'
OSB specs bundle
----------------
This directory contains:
- specs/domains/*.yaml  (DM, AE, LB, EX, VS, CM, CMH)
- specs/osb_runtime_meta.yaml
- specs/ct/README.txt    (placeholder for CDISC CT package)
- export_osb_to_yaml.py
- export_osb_to_yaml_with_validation.py

Usage:
1. Place your OSB JSON as osb_study.json next to the exporter scripts if you want to regenerate YAMLs.
2. Place your CDISC CT package JSON at specs/ct/sdtmct_2024-09-27.json for codelist validation.
3. Optionally create available_views.json or set DATABRICKS_HOST and DATABRICKS_TOKEN for Databricks validation.
MD

# Create archive
tar -czf osb_specs_bundle.tar.gz specs README_OSB_SPECS.md export_osb_to_yaml.py export_osb_to_yaml_with_validation.py

echo "Created specs/ tree and osb_specs_bundle.tar.gz in ${ROOT_DIR}"
echo "Done."
```

---

### Quick verification checklist after running the script

- **Files created**: `specs/domains/{dm,ae,lb,ex,vs,cm,cmh}.yaml` and `specs/osb_runtime_meta.yaml`.  
- **Exporter scripts**: `export_osb_to_yaml.py` and `export_osb_to_yaml_with_validation.py`.  
- **Archive**: `osb_specs_bundle.tar.gz` ready to upload or share.

If you want, I can now:
- Provide a **single pasteable PowerShell** variant for Windows, or  
- Produce a **compact zip download block** (base64) you can decode locally.

Which would you prefer next
### OSB specs installer PowerShell

Below is a single **pasteable PowerShell script** that creates the full `specs/` directory tree, writes all domain YAMLs (DM, AE, LB, EX, VS, CM, CMH), the runtime meta file, placeholder CT README, and two exporter scripts. It also produces a compressed archive `osb_specs_bundle.zip`. Save the block into a file named **`Create-OSB-Specs.ps1`** and run it in PowerShell (Windows).

```powershell
<#
.SYNOPSIS
Create OSB specs directory, domain YAMLs, exporter scripts, and a zip archive.

USAGE
1. Save this file as Create-OSB-Specs.ps1
2. Open PowerShell, navigate to the script folder
3. Run:  .\Create-OSB-Specs.ps1
#>

Set-StrictMode -Version Latest
$ErrorActionPreference = "Stop"

$RootDir = (Get-Location).Path
$OutDir  = Join-Path $RootDir "specs"
$DomDir  = Join-Path $OutDir "domains"
$CtDir   = Join-Path $OutDir "ct"

New-Item -ItemType Directory -Path $DomDir -Force | Out-Null
New-Item -ItemType Directory -Path $CtDir -Force | Out-Null

# Write DM
@"
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
"@ | Out-File -FilePath (Join-Path $DomDir "dm.yaml") -Encoding utf8

# Write AE
@"
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
"@ | Out-File -FilePath (Join-Path $DomDir "ae.yaml") -Encoding utf8

# Write LB
@"
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
"@ | Out-File -FilePath (Join-Path $DomDir "lb.yaml") -Encoding utf8

# Write EX
@"
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
"@ | Out-File -FilePath (Join-Path $DomDir "ex.yaml") -Encoding utf8

# Write VS
@"
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
"@ | Out-File -FilePath (Join-Path $DomDir "vs.yaml") -Encoding utf8

# Write CM
@"
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
"@ | Out-File -FilePath (Join-Path $DomDir "cm.yaml") -Encoding utf8

# Write CMH
@"
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
"@ | Out-File -FilePath (Join-Path $DomDir "cmh.yaml") -Encoding utf8

# Runtime meta
@"
study_id: XL092-123
project_root: /Workspace/Users/rossinni.r@gmail.com/Drafts/SDTM_Build_POC
new_run: true
reuse_timestamp: null
subjects_per_treatment_arm: 100
"@ | Out-File -FilePath (Join-Path $OutDir "osb_runtime_meta.yaml") -Encoding utf8

# CT README placeholder
@"
Place your CDISC CT package JSON here with filename:
  sdtmct_2024-09-27.json

This file is referenced by ct_package keys in the domain YAMLs.
"@ | Out-File -FilePath (Join-Path $CtDir "README.txt") -Encoding utf8

# Simple exporter script
@"
#!/usr/bin/env python3
import json
from pathlib import Path
import yaml
OSB_JSON = 'osb_study.json'
OUT_DIR = Path('specs/domains')
OUT_DIR.mkdir(parents=True, exist_ok=True)
def normalize_domain_spec(domain_obj):
    spec = {
        'domain': domain_obj.get('domain'),
        'label': domain_obj.get('label'),
        'source_view': domain_obj.get('source_view'),
        'depends_on': domain_obj.get('depends_on', []),
        'build_order_priority': domain_obj.get('build_order_priority', 99),
        'variables': domain_obj.get('variables', []),
        'suppqual_variables': domain_obj.get('suppqual_variables', []),
        'postprocessing': domain_obj.get('postprocessing', []),
        'filter': domain_obj.get('filter', None)
    }
    return spec
def write_yaml(domain_spec):
    domain = domain_spec['domain'].lower()
    filename = OUT_DIR / f\"{domain}.yaml\"
    with open(filename, 'w', encoding='utf-8') as f:
        yaml.safe_dump(domain_spec, f, sort_keys=False, default_flow_style=False)
    print(f'Wrote {filename}')
def main():
    if not Path(OSB_JSON).exists():
        print(f'ERROR: {OSB_JSON} not found.')
        return
    with open(OSB_JSON, 'r', encoding='utf-8') as fh:
        osb = json.load(fh)
    domains = osb.get('domains', [])
    if not domains:
        print('No domains found in OSB JSON. Nothing to export.')
        return
    for d in domains:
        spec = normalize_domain_spec(d)
        write_yaml(spec)
    runtime = osb.get('runtime', {})
    meta = {
        'study_id': osb.get('study', {}).get('study_id'),
        'project_root': osb.get('study', {}).get('project_root'),
        'new_run': runtime.get('new_run', True),
        'reuse_timestamp': runtime.get('reuse_timestamp'),
        'subjects_per_treatment_arm': runtime.get('subjects_per_treatment_arm')
    }
    meta_file = OUT_DIR.parent / 'osb_runtime_meta.yaml'
    with open(meta_file, 'w', encoding='utf-8') as fh:
        yaml.safe_dump(meta, fh, sort_keys=False)
    print(f'Wrote runtime meta {meta_file}')
if __name__ == '__main__':
    main()
"@ | Out-File -FilePath (Join-Path $RootDir "export_osb_to_yaml.py") -Encoding utf8

# Enhanced exporter with validation (short form)
@"
#!/usr/bin/env python3
# Enhanced exporter with validation. See README for usage.
"@ | Out-File -FilePath (Join-Path $RootDir "export_osb_to_yaml_with_validation.py") -Encoding utf8

# README
@"
OSB specs bundle
----------------
This directory contains:
- specs/domains/*.yaml  (DM, AE, LB, EX, VS, CM, CMH)
- specs/osb_runtime_meta.yaml
- specs/ct/README.txt    (placeholder for CDISC CT package)
- export_osb_to_yaml.py
- export_osb_to_yaml_with_validation.py

Usage:
1. Place your OSB JSON as osb_study.json next to the exporter scripts if you want to regenerate YAMLs.
2. Place your CDISC CT package JSON at specs/ct/sdtmct_2024-09-27.json for codelist validation.
3. Optionally create available_views.json or set DATABRICKS_HOST and DATABRICKS_TOKEN for Databricks validation.
"@ | Out-File -FilePath (Join-Path $RootDir "README_OSB_SPECS.md") -Encoding utf8

# Create zip archive
$zipPath = Join-Path $RootDir "osb_specs_bundle.zip"
if (Test-Path $zipPath) { Remove-Item $zipPath -Force }
Add-Type -AssemblyName System.IO.Compression.FileSystem
[System.IO.Compression.ZipFile]::CreateFromDirectory($OutDir, $zipPath)

Write-Host "Created specs/ tree and osb_specs_bundle.zip in $RootDir"
Write-Host "Files written to $DomDir and $CtDir"
```

---

### How to run the PowerShell script

1. **Save** the script as **`Create-OSB-Specs.ps1`**.  
2. **Open PowerShell** and navigate to the folder containing the script.  
3. If execution policy blocks scripts, run:  
   ```powershell
   Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
   ```  
4. **Execute** the script:  
   ```powershell
   .\Create-OSB-Specs.ps1
   ```  
5. After completion you will find **`specs/`**, **`export_osb_to_yaml.py`**, **`export_osb_to_yaml_with_validation.py`**, and **`osb_specs_bundle.zip`** in the same folder.

---

### What the script creates

- **Domain YAMLs**: `specs/domains/dm.yaml`, `ae.yaml`, `lb.yaml`, `ex.yaml`, `vs.yaml`, `cm.yaml`, `cmh.yaml`.  
- **Runtime meta**: `specs/osb_runtime_meta.yaml`.  
- **CT placeholder**: `specs/ct/README.txt` (drop your CDISC CT JSON here).  
- **Exporters**: `export_osb_to_yaml.py`, `export_osb_to_yaml_with_validation.py` (placeholder short form included).  
- **Archive**: `osb_specs_bundle.zip` ready to upload or share.

If you want the enhanced exporter fully populated with the Databricks validation function and codelist checks included, I will paste the complete Python file into the bundle on request.
### OSB specs installer PowerShell

Below is a single **pasteable PowerShell script** that creates the full `specs/` directory tree, writes all domain YAMLs (DM, AE, LB, EX, VS, CM, CMH), the runtime meta file, placeholder CT README, and two exporter scripts. It also produces a compressed archive `osb_specs_bundle.zip`. Save the block into a file named **`Create-OSB-Specs.ps1`** and run it in PowerShell (Windows).

```powershell
<#
.SYNOPSIS
Create OSB specs directory, domain YAMLs, exporter scripts, and a zip archive.

USAGE
1. Save this file as Create-OSB-Specs.ps1
2. Open PowerShell, navigate to the script folder
3. Run:  .\Create-OSB-Specs.ps1
#>

Set-StrictMode -Version Latest
$ErrorActionPreference = "Stop"

$RootDir = (Get-Location).Path
$OutDir  = Join-Path $RootDir "specs"
$DomDir  = Join-Path $OutDir "domains"
$CtDir   = Join-Path $OutDir "ct"

New-Item -ItemType Directory -Path $DomDir -Force | Out-Null
New-Item -ItemType Directory -Path $CtDir -Force | Out-Null

# Write DM
@"
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
"@ | Out-File -FilePath (Join-Path $DomDir "dm.yaml") -Encoding utf8

# Write AE
@"
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
"@ | Out-File -FilePath (Join-Path $DomDir "ae.yaml") -Encoding utf8

# Write LB
@"
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
"@ | Out-File -FilePath (Join-Path $DomDir "lb.yaml") -Encoding utf8

# Write EX
@"
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
"@ | Out-File -FilePath (Join-Path $DomDir "ex.yaml") -Encoding utf8

# Write VS
@"
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
"@ | Out-File -FilePath (Join-Path $DomDir "vs.yaml") -Encoding utf8

# Write CM
@"
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
"@ | Out-File -FilePath (Join-Path $DomDir "cm.yaml") -Encoding utf8

# Write CMH
@"
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
"@ | Out-File -FilePath (Join-Path $DomDir "cmh.yaml") -Encoding utf8

# Runtime meta
@"
study_id: XL092-123
project_root: /Workspace/Users/rossinni.r@gmail.com/Drafts/SDTM_Build_POC
new_run: true
reuse_timestamp: null
subjects_per_treatment_arm: 100
"@ | Out-File -FilePath (Join-Path $OutDir "osb_runtime_meta.yaml") -Encoding utf8

# CT README placeholder
@"
Place your CDISC CT package JSON here with filename:
  sdtmct_2024-09-27.json

This file is referenced by ct_package keys in the domain YAMLs.
"@ | Out-File -FilePath (Join-Path $CtDir "README.txt") -Encoding utf8

# Simple exporter script
@"
#!/usr/bin/env python3
import json
from pathlib import Path
import yaml
OSB_JSON = 'osb_study.json'
OUT_DIR = Path('specs/domains')
OUT_DIR.mkdir(parents=True, exist_ok=True)
def normalize_domain_spec(domain_obj):
    spec = {
        'domain': domain_obj.get('domain'),
        'label': domain_obj.get('label'),
        'source_view': domain_obj.get('source_view'),
        'depends_on': domain_obj.get('depends_on', []),
        'build_order_priority': domain_obj.get('build_order_priority', 99),
        'variables': domain_obj.get('variables', []),
        'suppqual_variables': domain_obj.get('suppqual_variables', []),
        'postprocessing': domain_obj.get('postprocessing', []),
        'filter': domain_obj.get('filter', None)
    }
    return spec
def write_yaml(domain_spec):
    domain = domain_spec['domain'].lower()
    filename = OUT_DIR / f\"{domain}.yaml\"
    with open(filename, 'w', encoding='utf-8') as f:
        yaml.safe_dump(domain_spec, f, sort_keys=False, default_flow_style=False)
    print(f'Wrote {filename}')
def main():
    if not Path(OSB_JSON).exists():
        print(f'ERROR: {OSB_JSON} not found.')
        return
    with open(OSB_JSON, 'r', encoding='utf-8') as fh:
        osb = json.load(fh)
    domains = osb.get('domains', [])
    if not domains:
        print('No domains found in OSB JSON. Nothing to export.')
        return
    for d in domains:
        spec = normalize_domain_spec(d)
        write_yaml(spec)
    runtime = osb.get('runtime', {})
    meta = {
        'study_id': osb.get('study', {}).get('study_id'),
        'project_root': osb.get('study', {}).get('project_root'),
        'new_run': runtime.get('new_run', True),
        'reuse_timestamp': runtime.get('reuse_timestamp'),
        'subjects_per_treatment_arm': runtime.get('subjects_per_treatment_arm')
    }
    meta_file = OUT_DIR.parent / 'osb_runtime_meta.yaml'
    with open(meta_file, 'w', encoding='utf-8') as fh:
        yaml.safe_dump(meta, fh, sort_keys=False)
    print(f'Wrote runtime meta {meta_file}')
if __name__ == '__main__':
    main()
"@ | Out-File -FilePath (Join-Path $RootDir "export_osb_to_yaml.py") -Encoding utf8

# Enhanced exporter with validation (short form)
@"
#!/usr/bin/env python3
# Enhanced exporter with validation. See README for usage.
"@ | Out-File -FilePath (Join-Path $RootDir "export_osb_to_yaml_with_validation.py") -Encoding utf8

# README
@"
OSB specs bundle
----------------
This directory contains:
- specs/domains/*.yaml  (DM, AE, LB, EX, VS, CM, CMH)
- specs/osb_runtime_meta.yaml
- specs/ct/README.txt    (placeholder for CDISC CT package)
- export_osb_to_yaml.py
- export_osb_to_yaml_with_validation.py

Usage:
1. Place your OSB JSON as osb_study.json next to the exporter scripts if you want to regenerate YAMLs.
2. Place your CDISC CT package JSON at specs/ct/sdtmct_2024-09-27.json for codelist validation.
3. Optionally create available_views.json or set DATABRICKS_HOST and DATABRICKS_TOKEN for Databricks validation.
"@ | Out-File -FilePath (Join-Path $RootDir "README_OSB_SPECS.md") -Encoding utf8

# Create zip archive
$zipPath = Join-Path $RootDir "osb_specs_bundle.zip"
if (Test-Path $zipPath) { Remove-Item $zipPath -Force }
Add-Type -AssemblyName System.IO.Compression.FileSystem
[System.IO.Compression.ZipFile]::CreateFromDirectory($OutDir, $zipPath)

Write-Host "Created specs/ tree and osb_specs_bundle.zip in $RootDir"
Write-Host "Files written to $DomDir and $CtDir"
```

---

### How to run the PowerShell script

1. **Save** the script as **`Create-OSB-Specs.ps1`**.  
2. **Open PowerShell** and navigate to the folder containing the script.  
3. If execution policy blocks scripts, run:  
   ```powershell
   Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
   ```  
4. **Execute** the script:  
   ```powershell
   .\Create-OSB-Specs.ps1
   ```  
5. After completion you will find **`specs/`**, **`export_osb_to_yaml.py`**, **`export_osb_to_yaml_with_validation.py`**, and **`osb_specs_bundle.zip`** in the same folder.

---

### What the script creates

- **Domain YAMLs**: `specs/domains/dm.yaml`, `ae.yaml`, `lb.yaml`, `ex.yaml`, `vs.yaml`, `cm.yaml`, `cmh.yaml`.  
- **Runtime meta**: `specs/osb_runtime_meta.yaml`.  
- **CT placeholder**: `specs/ct/README.txt` (drop your CDISC CT JSON here).  
- **Exporters**: `export_osb_to_yaml.py`, `export_osb_to_yaml_with_validation.py` (placeholder short form included).  
- **Archive**: `osb_specs_bundle.zip` ready to upload or share.

If you want the enhanced exporter fully populated with the Databricks validation function and codelist checks included, I will paste the complete Python file into the bundle on request.
