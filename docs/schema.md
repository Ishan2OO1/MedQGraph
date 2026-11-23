# Graph model

Source: the MedQGraph paper (*Structuring Temporal Medical Knowledge from
Clinical Narratives for AI-Driven Insights*), sections 3-5 and Appendix B.

## Source data

MIMIC-IV v3.0 (PhysioNet, credentialed access), maintained by the MIT Laboratory
for Computational Physiology. The full database holds 364,484 patients, 546,165
admissions spanning 2008-2019, and over 95,000 ICU stays.

Six modules are used:

| Module | Source tables | Join keys |
|---|---|---|
| Demographics | `patients` | `subject_id` |
| Admissions | `admissions`, `transfers` | `subject_id`, `hadm_id` |
| Medications | `prescriptions`, `pharmacy` | `hadm_id` |
| Labs | `labevents`, `d_labitems` | `hadm_id` |
| Microbiology | `microbiologyevents` | `hadm_id` |
| Diagnoses | `diagnoses_icd`, `drgcodes` | `hadm_id` |

## Ingest

Three JOIN stages, then denormalisation:

1. **Patient-admission core.** `patients INNER JOIN admissions ON subject_id`,
   producing 230,000 patient-admission pairs so every clinical event is
   attributable to a patient and a hospitalisation.
2. **Clinical events.** Sequential `LEFT JOIN`s append medications, labs,
   microbiology cultures and diagnosis/DRG codes. LEFT rather than INNER so that
   a pair survives when an event type is absent — which is why completeness
   varies by module.
3. **Denormalisation.** Flattened to one wide table, each row a clinical event
   snapshot. This removes runtime joins from graph queries and allows batch
   ingestion into Neo4j.

Quality gates: referential integrity (100% of records hold valid `subject_id`
and `hadm_id` links), deduplication preserving legitimate repeats such as daily
medication orders and serial labs, range validation (`anchor_age` 18-89 per
MIMIC-IV de-identification, severity and mortality 1-4, non-negative doses), and
categorical standardisation of admission type, race and route.

NULLs are preserved rather than imputed, so that a missing value becomes an
absent edge instead of a fabricated relationship.

## Final dataset

| | |
|---|---|
| Total records | 230,000 |
| Unique patients | 45,127 |
| Unique admissions | 78,043 |
| Features | 31 |
| Records per patient | 5.1 avg |
| Records per admission | 2.9 avg |
| Temporal coverage | 2008-2019 |

Coverage by module: demographics, admission details, medication administration,
lab testing and diagnosis/severity are at 100%; microbiology at 14.8% (34,000
records), since an absent culture means one was negative or never ordered.
Overall field coverage 86.4%.

Admission types: urgent 97,290 (42.3%), emergency 89,010 (38.7%), elective
34,960 (15.2%), other 8,740 (3.8%). APR-DRG severity runs minor 18.5%, moderate
34.2%, major 32.8%, extreme 14.5%.

## FHIR normalisation

Before graph ingestion, MIMIC-IV v3.0 and MIMIC-IV-ED are mapped to FHIR
profiles following the MIMIC-IV-on-FHIR transformation, covering Patient,
Encounter, Condition, Procedure, Medication, Observation, Specimen and
Organization. Custom profiles reproduce MIMIC's own coding schema rather than
remapping to SNOMED CT, LOINC or RxNorm, and ICU and ED encounters
(`MimicEncounterICU`, `MimicEncounterED`) are linked back to the primary
hospitalisation.

## Nodes

`Patient`, `Admission`, `Diagnosis`, `Medication`, `Procedure`, `Test`

Created with `MERGE` on a stable key — `subject_id` for patients, `hadm_id` for
admissions, `description` for diagnoses, `drug_name` for medications,
`test_name` for tests (Appendix B, Figure 5).

## Relationships

`HAD_ADMISSION`, `DIAGNOSED_WITH`, `PRESCRIBED`, `UNDERWENT_TEST`,
`HAS_MEDICAL_HISTORY`, `HAS_RISK`

Relationships are inferred from event co-occurrence within an admission, then
sorted chronologically within each admission. Clinical concepts are connected by
time-stamped edges, which is what makes the graph temporal rather than static.

<!-- TODO: one naming inconsistency to settle before the camera-ready and before
     this repo's code lands. The schema given to the Cypher Generation Agent
     (section 5.7.1) names HAD_ADMISSION / PRESCRIBED / UNDERWENT_TEST, and the
     worked example uses those. The loader in Appendix B Figure 6 creates
     HAS_ADMISSION / HAS_DIAGNOSIS / HAS_MEDICATION / HAS_TEST, and Figure 8's
     caption uses HAS_MEDICATION. Both sets cannot be right. Pick one and make
     the loader, the agent prompt and the figures agree. -->

## Query path

Four GPT-4o agents run in sequence:

1. **Query Understanding** — extracts clinical intent and entities from the
   natural-language question.
2. **Cypher Generation** — given a schema description and the question, returns
   a syntactically valid Cypher query restricted to schema-present nodes and
   properties, matching on `subject_id` when a patient ID is named. Queries are
   validated for syntax, schema compliance, property existence and query cost
   before execution; a validation failure triggers regeneration with structured
   feedback.
3. **Summarization** — turns rows into a clinician-readable summary, adapting
   style by query type and flagging time-sensitive findings.
4. **Recommendation** — proposes three to five follow-up questions built from
   the relationships adjacent to the answer.

Context is carried across turns, so a user can reason in steps without knowing
Cypher or the schema.

LangChain embeddings place queries and graph entities in a shared vector space,
giving similarity-based retrieval alongside the generated Cypher.

## Worked example

> **"Medications for patient 10000032"**

```cypher
MATCH (p:Patient {subject_id: 10000032})
      -[:HAD_ADMISSION]->(a:Admission)
      -[:PRESCRIBED]->(m:Medication)
RETURN m.drug_name
```

Returns ten medications — furosemide, heparin, ipratropium bromide neb, nicotine
patch, raltegravir, influenza vaccine quadrivalent, emtricitabine-tenofovir
(Truvada), spironolactone, potassium chloride, sodium chloride 0.9% flush — which
the Summarization Agent renders as prose.

Two more, from the follow-up agent:

```cypher
-- "What are the various medical histories of the patient?"
MATCH (n:MedicalCondition)-[r:HAS_MEDICAL_HISTORY]-(m) RETURN n, r, m LIMIT 100;

-- "What risk is associated with a procedure?"
MATCH (n:Procedure)-[r:HAS_TREATMENT_PLAN]-(m) RETURN n, r, m LIMIT 100;
```

## Visualisation

An Electron.js and NetworkX interface presents patient data as a timeline, with
filtering, zooming and node highlighting.
