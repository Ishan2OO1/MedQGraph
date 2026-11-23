# Graph model

Reconstructed from `data-creation-pipeline.drawio`, `complete-pipeline.drawio`
and the project slides. **Check this against the Cypher in the loader before
publishing** — the diagrams name the entities but not every property.

## Source

MIMIC-IV v3.0 (PhysioNet, credentialed access). Six tables are used:

| Table | Fields taken |
|---|---|
| `patients` | `subject_id`, gender, age, race |
| `admissions` | `hadm_id`, admission type, location |
| `prescriptions` | drug, dose, route |
| `labevents` | test name, specimen |
| `microbiologyevents` | organism, antibiotic |
| `diagnoses_icd` | ICD code, severity |

## Ingest

1. `patients` JOIN `admissions` on `subject_id`
2. join `prescriptions` on `hadm_id`
3. join `labevents` on `hadm_id`
4. join `microbiologyevents` on `hadm_id`
5. join `diagnoses_icd` on `hadm_id`
6. denormalise to one wide row per patient-plus-events
7. quality pass: referential integrity, deduplication, range validation,
   categorical standardisation
8. CSV export, then parameterised Cypher creates nodes and relationships

Resulting dataset: **230,000 records x 31 features**, covering **45,127
patients** and **78,043 admissions**.

## Nodes

`Patient`, `Admission`, `Diagnosis`, `Medication`, `Test`

<!-- TODO: property list per node type, straight from the loader's Cypher. -->

## Relationships

`HAD_ADMISSION`, `DIAGNOSED_WITH`, and others.

<!-- TODO: the full edge list, with direction and properties.

     This is the most important gap in the repo. The project is described as a
     *temporal* knowledge graph, so the README needs to show exactly where time
     is modelled: timestamps as edge properties, ordered event chains, or
     validity intervals. Name it concretely.

     Note the conflict to resolve: the poster's own Limitations section says
     "limited to structured data; does not include clinical notes or temporal
     relationships", while the body and the slides describe a Temporal Medical
     Knowledge Graph. Decide which is accurate for the system as built and make
     the repo, the poster and the portfolio say the same thing. -->

## Query path

A natural-language question is sent to GPT-4o together with a schema
description, which returns a Cypher query and nothing else. The query runs
against Neo4j; the rows come back to a second prompt that writes a plain-language
answer, and a third prompt proposes three to five follow-up questions built from
the relationships that the answer touched.

LangChain embeddings back a vector search for similarity-based retrieval
alongside the generated Cypher.

<!-- TODO: two or three real question/answer pairs with the Cypher each one
     produced. Figure 4 of the poster has one already: medications for patient
     10000032. -->
