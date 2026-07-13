# OSB_2_TLF
DIgitized protocol to deterministic raw to TLF execution
🔍 What OpenStudyBuilder actually does
OpenStudyBuilder (OSB) is an open source metadata repository and study design system created by Novo Nordisk and released through the CDISC Open Source Alliance. It supports end to end digitization of clinical studies starting from the protocol. Key capabilities include:  
- Extracting protocol metadata into the Unified Study Definition Model (USDM)   
- Generating ICH M11 compliant protocol outputs using shared metadata   
- Storing study definitions (objectives, endpoints, schedules, assessments) in a semantic graph database for reuse across CRFs, datasets, and reports   
- Automating propagation of protocol elements into downstream trial systems (CRF design, SDTM/ADaM mapping, etc.)   

In short: OSB digitizes the protocol so the rest of the clinical trial workflow can be automated.

---

🆚 How this compares to AI powered protocol digitization tools
Other platforms (e.g., Verily Viewpoint, Agilisium) also digitize protocols but rely heavily on AI/LLM extraction, automated conformance checks, and amendment tracking engines.  
- Verily uses AI + human QC to convert PDF protocols into digital models.   
- Agilisium uses AI to extract study design, arms, endpoints, visits, and schedules, achieving up to 90% faster protocol digitization.   

OpenStudyBuilder is more metadata driven and standards centric, while these commercial tools are more AI driven.

---

📊 Quick comparison

| Capability | OpenStudyBuilder | Verily / Agilisium AI tools |
|-----------|------------------|------------------------------|
| Protocol digitization | ✔ Yes (metadata extraction + USDM) | ✔ Yes (AI driven PDF ingestion) |
| Standards support | Strong (USDM, ICH M11, CDISC SDTM/ADaM/CDASH) | Strong (USDM, CDISC validation) |
| Automation | Metadata propagation across CRF, SDTM, reports | Automated extraction + amendment intelligence |
| Open source | ✔ Yes | ✘ No |
| Best for | Sponsors wanting structured, reusable metadata | Sponsors needing fast AI powered PDF ingestion |

The core steps for digitizing a clinical study protocol in OpenStudyBuilder (OSB) are: enter structured study metadata → link it to standards in the Library → build the Schedule of Activities → generate USDM/ICH M11 protocol outputs.  
Below is a complete, workflow level breakdown grounded in OSB documentation and conference papers. 

---

🧩 1. Prepare the standards and building blocks (Library setup)
OSB relies on a metadata repository (MDR) containing reusable, standards aligned components. Before digitizing a protocol, ensure the Library is populated with:

- Controlled terminology (CDISC, MedDRA, units, activities)   
- Biomedical Concepts (BCs) and activity definitions used for endpoints, assessments, CRFs, and SDTM mapping   
- Standard templates for objectives, endpoints, inclusion/exclusion criteria, interventions, and syntax blocks used in protocol generation   

This step ensures that when you digitize the protocol, every element is linked to reusable metadata rather than free text.

---

🧩 2. Create the Study shell in OSB (Studies area)
In the Studies section, create a new study record and populate high level protocol metadata:

- Study title, identifiers, registry IDs  
- Purpose, phase, population, design structure  
- Interventions and arms  
- Eligibility criteria  
- Objectives and endpoints, selected from Library templates for consistency and reuse   

This replaces the traditional manual protocol drafting with structured metadata entry.

---

🧩 3. Build the Schedule of Activities (SoA)
The SoA is the backbone of protocol digitization. OSB uses the Activity concept to connect assessments, visits, CRF fields, and SDTM variables. Steps:

- Define visits/timepoints  
- Assign activities (e.g., vitals, labs, questionnaires) from the Library  
- Link activities to endpoints, CRF concepts, and SDTM domains via metadata relationships in the graph database   

This creates a fully machine readable SoA that drives downstream automation.

---

🧩 4. Validate metadata relationships
OSB’s graph database ensures traceability:

- Check that each activity is linked to the correct biomedical concept  
- Validate that objectives → endpoints → assessments → CRFs → SDTM mappings are consistent  
- Ensure terminology alignment (CDISC, MedDRA, units)  

This step is critical for end to end automation and is emphasized in OSB protocol automation papers. 

---

🧩 5. Generate protocol outputs (USDM + ICH M11)
Once metadata is complete:

- Export Unified Study Definition Model (USDM) JSON for digital data flow systems  
- Generate ICH M11 compliant protocol using shared metadata  
- Use the Word Add In to populate structured protocol sections (e.g., SoA flowchart, objectives, endpoints) directly into sponsor templates   

This is the actual “digitized protocol”—a structured, standards aligned, machine readable representation.

---

🧩 6. Push metadata downstream (optional but typical)
Digitized protocol metadata can now drive:

- CRF design (CDASH aligned)  
- EDC setup (via OSB API + DDF API adaptor)   
- SDTM/ADaM tabulation specifications  
- TFL/CTR generation  

This is the “define once, use many times” automation OSB was built for. 

---

📌 Summary workflow (quick reference)
1. Load standards & templates in Library  
2. Create study shell  
3. Enter structured protocol metadata  
4. Build Schedule of Activities  
5. Validate metadata relationships  
6. Export USDM + generate ICH M11 protocol  
7. Push metadata to CRF/EDC/SDTM systems  

Executive summary explaining osb protocol digitization process and output
OpenStudyBuilder (OSB) digitizes the clinical protocol by converting traditionally narrative, inconsistent study definitions into a structured, standards-linked metadata model that can automatically generate protocol content, Schedule of Activities (SoA), CRFs, and USDM/ICH M11 outputs.  
The core value: one structured study definition becomes the single source of truth for all downstream clinical systems.

📘 Executive Summary: OSB Protocol Digitization Process
OpenStudyBuilder replaces document‑centric protocol authoring with a metadata‑driven, semantic study definition. Instead of manually writing and re‑writing objectives, endpoints, visits, and assessments across multiple systems, OSB captures these elements once in a structured graph repository and propagates them automatically across protocol templates, CRFs, SDTM/ADaM mappings, and regulatory outputs.

This approach directly addresses long‑standing industry problems: duplicated effort, inconsistent terminology, siloed authoring, and error‑prone manual transcription. OSB’s architecture — a Vue.js web app, Neo4j semantic graph, and FastAPI backend — enables real‑time collaboration, versioning, and full audit trails. 

🔧 How Protocol Digitization Works (Process Overview)
1. Define Study Metadata  
Study purpose, endpoints, population, interventions, randomization, visit schedule, and activities are entered as structured metadata. Each element is versioned and linked to CDISC standards (SDTM, CDASH, ADaM) and controlled terminology. 

2. Build the Schedule of Activities (SoA)  
Activities and assessments are mapped to visits using standardized concepts. OSB stores these relationships in the semantic graph, enabling automated reuse across protocol, CRF, and EDC setup. 

3. Apply Standards & Templates  
OSB uses syntax templates and controlled terminology to ensure consistency across all generated content. This eliminates discrepancies between protocol text, CRFs, and SDTM datasets — a major source of rework in traditional workflows. 

4. Generate Structured Protocol Content  
OSB exports structured protocol sections — including SoA — into Word templates via content controls. A Word Add‑In (open‑source) supports direct protocol generation. 

5. Produce Digital Outputs (USDM / ICH M11)  
OSB can export study definitions as USDM‑compliant JSON or M11‑formatted HTML, enabling alignment with emerging regulatory digital submission standards. 

📤 Key Outputs of OSB Protocol Digitization
Structured Protocol Document  
Word‑based protocol with auto‑generated SoA and standardized terminology.

USDM JSON  
Machine‑readable study definition supporting Digital Data Flow (DDF) and regulatory automation.

ICH M11 HTML  
Harmonized protocol format aligned with global regulatory expectations.

CRF & EDC Setup Metadata  
Downstream systems can consume OSB metadata to automate CRF creation and EDC configuration. 

SDTM/ADaM Mapping Foundations  
Because activities, endpoints, and assessments are linked to CDISC concepts, OSB provides a consistent foundation for SDTM and ADaM dataset design.

🧩 Why This Matters
OSB’s “define once, use many times” model eliminates inconsistencies between protocol, CRFs, and datasets — a major source of regulatory findings and operational delays. It also enables future automation: synthetic test data generation, automated EDC setup, and seamless integration with DDF and other MDR systems.
