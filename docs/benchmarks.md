# Benchmarks

**Status: not reproducible from this repository yet.** The numbers below are
transcribed from the project's own slides and poster. Before any of them is
quoted anywhere, the harness that produced them needs to live in this repo.

## Query latency and accuracy against a SQL baseline

From Table 6 of the project deck, "Performance Breakdown by Clinical Query Type".
Time in milliseconds, lower is better; accuracy in percent.

| System | Medication (ms / acc) | Diagnosis (ms / acc) | Risk assessment (ms / acc) | Avg time vs SQL |
|---|---|---|---|---|
| MedGraphQA | 820 / 78.1 | 880 / 74.2 | - | +67% |
| Graph4MedQA | 890 / 81.3 | 950 / 78.5 | 940 / 76.2 | +82% |
| MedTKG | 750 / 83.7 | 810 / 79.1 | - | +59% |
| SQL baseline | 450 / 71.2 | 520 / 66.8 | 380 / 62.1 | baseline |
| **MedQGraph** | **158 / 96.2** | **182 / 93.8** | **133 / 92.4** | **-65%** |

What this table needs before it can be cited:

<!-- TODO -->
- What the SQL baseline is: hand-written SQL over the MIMIC-IV tables, a
  text-to-SQL system, or something else. Where its queries came from.
- The query set: how many questions, who wrote them, how they were chosen.
- How accuracy was scored, and by whom.
- Whether MedGraphQA / Graph4MedQA / MedTKG were run here or their numbers were
  taken from their papers. If taken from papers, cite them; the hardware will
  not match and the table should say so.
- The machine the timings were taken on, and how many runs each figure averages.

## Language-model comparison

From the poster. Models were compared on generated clinical answers:

| Model | BLEU | Edit distance | TF-IDF | ROUGE |
|---|---|---|---|---|
| Qwen2.5 7B | 0.2512 | 0.6879 | 0.7765 | 0.6667 |
| Gemma 3 4B | 0.1774 | 0.8716 | 0.8182 | 0.5714 |
| Mistral 7B | 0.0954 | 0.6214 | 0.5562 | 0.4762 |
| Llama 3.2 | 0.1383 | 0.7598 | 0.6698 | 0.5000 |

<!-- TODO: the reference answers these were scored against, and the prompt used.
     Note that GPT-4o, which the pipeline actually runs on, is absent from this
     table. -->

## Limitations, as the poster states them

- Structured data only; clinical notes are not included.
- Evaluated only on MIMIC-IV, which limits generalisability.
