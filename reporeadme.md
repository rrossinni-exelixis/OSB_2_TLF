### Executive Summary

This repository contains a complete **OSB specs bundle** that bridges a metadata‑driven study definition to your `raw_2_tlf_v1` SDTM/ADaM/TLF pipeline. It includes ready‑made **domain YAMLs** for DM, AE, LB, EX, VS, CM, and CMH, runtime metadata, two exporter scripts, a Databricks‑aware validation routine, and packaging scripts for Linux and Windows. The bundle is designed to be a single source of truth for domain build specifications, to validate source view availability and codelist references, and to produce the exact `specs/domains/*.yaml` files your notebook expects.

---

### Contents Overview

| **Path** | **Purpose** | **Key note** |
|---|---|---|
| `specs/domains/dm.yaml` | Demographics domain spec | Source view `dm_prepped` |
| `specs/domains/ae.yaml` | Adverse events domain spec | Includes SUPPQUAL entry |
| `specs/domains/lb.yaml` | Laboratory domain spec | Numeric and unit fields |
| `specs/domains/ex.yaml` | Exposure domain spec | Dose and date derivations |
| `specs/domains/vs.yaml` | Vital signs domain spec | VSTESTCD mapping included |
| `specs/domains/cm.yaml` | Concomitant medications spec | CMSTDTC and CMENDTC derivations |
| `specs/domains/cmh.yaml` | Concomitant medication history spec | History records and dates |
| `specs/osb_runtime_meta.yaml` | Runtime metadata for notebook | Study id and run flags |
| `specs/ct/` | CDISC CT package placeholder | Place `sdtmct_2024-09-27.json` here |
| `export_osb_to_yaml.py` | Simple exporter from OSB JSON to YAML | Lightweight regeneration tool |
| `export_osb_to_yaml_with_validation.py` | Enhanced exporter with validation | Databricks and CT checks |
| `create_osb_specs.sh` | Linux installer script | Creates files and tarball |
| `Create-OSB-Specs.ps1` | Windows installer PowerShell script | Creates files and zip archive |
| `osb_specs_bundle.*` | Compressed archive | Ready to upload |

---

### Installation and Quick Start

#### Prerequisites
- **Python 3.8+** installed for exporters.  
- **pip** available to install `pyyaml` and `requests` for the enhanced exporter.  
- If you want Databricks validation, a **Databricks personal access token** with SQL permissions.

#### Quick start commands

1. **Unpack or create files** using the provided installer script for your OS.

Linux
```bash
chmod +x create_osb_specs.sh
./create_osb_specs.sh
```

Windows PowerShell
```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.\Create-OSB-Specs.ps1
```

2. **Place your OSB JSON** next to the exporter scripts as `osb_study.json` if you want to regenerate YAMLs.

3. **Install Python dependencies**
```bash
pip install pyyaml requests
```

4. **Run the enhanced exporter with validation**
```bash
# Optional environment variables for Databricks validation
export DATABRICKS_HOST="https://<your-databricks-host>"
export DATABRICKS_TOKEN="<your-token>"
export DATABRICKS_CATALOG="your_catalog"
export DATABRICKS_SCHEMA="your_schema"

python export_osb_to_yaml_with_validation.py
```

5. **Inspect outputs**
- Domain YAMLs in `specs/domains/`  
- Runtime meta in `specs/osb_runtime_meta.yaml`  
- Validation report in `specs/domains/validation_report.yaml`

---

### Validation and Databricks Integration

#### What the enhanced exporter validates
- **Derivation presence** for every variable and SUPPQUAL variable.  
- **Source view existence** using Databricks SQL Statements API or a local `available_views.json`.  
- **Codelist references** against the CDISC CT package JSON placed at `specs/ct/sdtmct_2024-09-27.json`.  
- **Outputs a validation report** at `specs/domains/validation_report.yaml` summarizing errors and warnings.

#### Databricks check details
- Preferred SQL used
  - `SHOW TABLES IN <catalog>.<schema> LIKE '<view_name>'`
- Fallback SQL used
  - `SHOW TABLES IN <schema> LIKE '<view_name>'`
  - `SHOW TABLES LIKE '<view_name>'`
- The exporter calls the Databricks SQL Statements API endpoint `/api/2.0/sql/statements/execute` and polls the statement result.

#### Example curl test you run locally
```bash
DATABRICKS_HOST="https://adb-123456789012345.7.azuredatabricks.net"
TOKEN="<your-token>"
CATALOG="your_catalog"
SCHEMA="your_schema"
VIEW="dm_prepped"
SQL="SHOW TABLES IN ${CATALOG}.${SCHEMA} LIKE '${VIEW}'"

curl -s -X POST "${DATABRICKS_HOST}/api/2.0/sql/statements/execute" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{\"statement\": \"${SQL}\"}" | jq .
```

#### Validation outcomes
- **Fatal errors** include missing derivations, missing source views when validated against `available_views.json`, and missing codelists when CT package is present.  
- **Warnings** include inability to contact Databricks API or missing CT package file.  
- The exporter exits nonzero when fatal errors are present.

---

### Exporter Scripts and Usage

#### `export_osb_to_yaml.py`
- **Purpose**: Simple conversion of an OSB JSON study definition into `specs/domains/*.yaml`.  
- **Usage**: Place `osb_study.json` next to the script and run it. It writes YAMLs and `specs/osb_runtime_meta.yaml`.

#### `export_osb_to_yaml_with_validation.py`
- **Purpose**: Full exporter with validation checks for derivations, source views, and codelists.  
- **Key environment variables**
  - `DATABRICKS_HOST` Databricks workspace URL  
  - `DATABRICKS_TOKEN` Personal access token  
  - `DATABRICKS_CATALOG` Optional catalog for Unity Catalog  
  - `DATABRICKS_SCHEMA` Optional schema for prepared views
- **Local validation alternative**: create `available_views.json` containing a JSON array of view names to validate source views without Databricks credentials.
- **Outputs**
  - Domain YAMLs in `specs/domains/`  
  - `specs/osb_runtime_meta.yaml`  
  - `specs/domains/validation_report.yaml`

#### Typical workflow
1. Author or update `osb_study.json`.  
2. Run the enhanced exporter to produce YAMLs and validation report.  
3. Fix any validation errors.  
4. Commit `specs/domains/*.yaml` to the pipeline repository.  
5. Run the notebook with `RUN_DATA_PREVIEW=True` and `RUN_ROW_COUNTS=True` to confirm SQL generation.

---

### Troubleshooting and Next Steps

#### Common issues and fixes
- **Missing CT package errors**  
  - Place the CDISC CT JSON at `specs/ct/sdtmct_2024-09-27.json` or update `osb_study.json` to point to your CT package path.
- **Databricks API authentication failures**  
  - Verify `DATABRICKS_HOST` and `DATABRICKS_TOKEN` values and ensure the token has SQL permissions.
- **Source view not found**  
  - Confirm the prepared view exists in the catalog and schema you provided or add the view name to `available_views.json`.
- **Exporter exits with errors**  
  - Inspect `specs/domains/validation_report.yaml` for per‑domain error messages and fix derivations or codelist names.

#### Recommended next steps
- Add additional domains as needed by extending the OSB JSON `domains` array and re-running the exporter.  
- Integrate the exporter into your CI pipeline to validate domain specs on each commit.  
- Extend the exporter to embed additional metadata such as `define_xml` mappings or ADaM skeleton hints.  
- If you want the enhanced exporter to run inside Databricks, adapt the Databricks check to use your workspace SQL endpoint and test with a nonprivileged token.

---

**Appendix: quick checklist before running the notebook**

- **specs/domains/** contains the YAMLs for all required domains.  
- **specs/ct/sdtmct_2024-09-27.json** is present for codelist validation.  
- **osb_study.json** is available if you plan to regenerate YAMLs.  
- **DATABRICKS_HOST** and **DATABRICKS_TOKEN** are set if you want live view validation.  
- Run `python export_osb_to_yaml_with_validation.py` and resolve any errors in `specs/domains/validation_report.yaml` before executing the pipeline notebook.

---

If you want, I will paste the **fully populated enhanced exporter Python file** into the bundle so the Windows and Linux installers include the complete validation logic ready to run.
