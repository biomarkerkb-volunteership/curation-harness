# Harness engineering for PubMed article curation of molecular clinical biomarkers

## Overview

While our pilot LLM curation harness focuses on glycan biomarkers, BiomarkerKB covers a much wider range of measured entities: DNA variants, genes, RNA, proteins and protein complexes, metabolites, lipids, and other molecules. A protein abundance result, a somatic variant, and a circulating metabolite cannot be curated with the same set of questions. The study design, specimen, comparison group, and intended clinical use also change what information is needed to interpret a result.

The goal is to design a general curation harness for PubMed articles that can route each finding through the appropriate questions and produce an evidence-backed BiomarkerKB record. The major deliverable would be a clear, testable **curation protocol** that even a human curator can follow.

This project should be built **on top of** our existing data model:

- [Data upload wiki](https://wiki.biomarkerkb.org/Data_Submission/Data_Upload) describes the present submission fields and examples.
- [BiomarkerKB paper](<https://www.cell.com/patterns/fulltext/S2666-3899(26)00145-5>) discusses the data modeling in full.
- [Controlled vocabulary](https://github.com/clinical-biomarkers/biomarker-controlled-vocabulary) defines the **change**, **aspect**, and **entity** types (for example, increased / level / IL6).
- [Data portal](https://data.biomarkerkb.org/) provides datasets to inspect.

## Scope for Fall 2026

### 1. Survey identifier and ontology resources

Scout the [OBO Foundry](https://obofoundry.org/) and the resources already named in the [BiomarkerKB submission guide](<(https://wiki.biomarkerkb.org/Data_Submission/Data_Upload)>). Click into the individual ontology pages, and download the "Products" to examine the data (.obo files are more human readable in general). **Make a component-by-component shortlist** for assessed entities, diseases, treatments or exposures, specimens, anatomy, cell types, cell lines, molecular features, and measurement concepts. Some useful identifier resources are outside OBO; include them when they are the right source for a component.

For each candidate, inspect actual terms and metadata, not only the ontology description. Record its identifier pattern, term definitions, synonyms, cross-references, hierarchy or granularity, coverage of examples from biomarker papers, maintenance, and any relevant access or reuse constraints. Recommend a **preferred source and fallback order** for each component, with examples showing when the fallback applies. Compare your recommendations with BiomarkerKB's existing preferences and explain any proposed change. If a source only has a broader term, preserve the specific article wording and mark the mapping as unresolved or broader; do not make a record look more precise than the source supports.

**Deliverable:**

- [ ] A table of shortlisted ontologies, each with the decisions: recommendation, example mappings
- [ ] A document of fallback orders and uncovered gaps if any

### 2. Design the curation protocol and routing logic

Write the protocol as a set of decisions that a curator or future agent can follow. No coding is expected for this stage. It should address at least the following routes:

1. **Article and study:** Is this a human clinical biomarker claim, a preclinical result, or background discussion? Is the evidence from a case-control or cross-sectional comparison, a longitudinal cohort, a treatment study, a clinical trial, or a discovery/validation series? Record the relevant population, disease and subtype, comparator, sampling time, treatment/exposure, outcome, assay, and validation cohort when reported. Specify which details matter for each study type.
2. **Measured entity and observation:** Identify the entity type, molecular identity, measured aspect, change or status, and any essential qualifiers. A variant needs allele or locus detail; an RNA result may need transcript or expression context; a protein complex needs its members; a metabolite or lipid may need chemical specificity. Decide when several measured entities form one multicomponent biomarker and when they represent separate claims.
3. **Specimen and location:** Distinguish the specimen collected from the anatomical site, cell population, subcellular location, and cultured cell line. Record each only when the article supports it, and map each to an appropriate identifier where possible. State how to handle a result measured in several specimens or time points.
4. **BEST role:** Determine whether the reported use is susceptibility/risk, diagnostic, monitoring, prognostic, predictive, pharmacodynamic/response, or safety. For each role, list the context needed for a reusable record and the questions that would reveal it. For example, predictive claims need a treatment or exposure and a comparison of outcomes or responses by biomarker status; a change _after_ treatment alone may instead support a response claim. A paper may support more than one role, but each claim needs its own evidence and context.
5. **Evidence and completeness:** Link every claim and mapped value to the relevant article passage, table, or figure. Mark a field as **reported**, **not reported**, **not applicable**, or **uncertain**; do not fill a gap by guessing. Keep the author's finding distinct from our interpretation. State when an abstract is insufficient and full text is needed, and when a record should be held for review.

The routing design should show how study type, entity type, specimen, and BEST role interact, without requiring a separate monolithic workflow for every possible combination. A decision tree plus small annotation tables is fine. Test the decisions manually on a few deliberately different papers, including an ambiguous case. Track where the present BiomarkerKB schema fits and where a proposed extension or sidecar annotation would be needed.

**Deliverable:**

- [ ] A written curation protocol, routing diagram or decision table, role-specific annotation checklist, and worked examples with evidence and unresolved fields.

### 3. Optional implementation, if stages 1–2 are complete

If the protocol is stable before the semester ends, add small supporting scripts for tasks such as schema checks, identifier lookup, or completeness reports. Then package the protocol as a Claude/agent skill and try it on a small set of example articles. Compare its outputs with a human-reviewed reference: entity and identifier correctness, role assignment, required context, evidence support, and the frequency of unsupported or missing values. Record errors by route so that the protocol can be revised. This stage is optional; a well-specified and tested protocol is a complete Fall outcome.

See: [The Complete Guide to Building Skills for Claude](https://resources.anthropic.com/hubfs/The-Complete-Guide-to-Building-Skill-for-Claude.pdf)

## Weekly goals

| Week    | Dates (2026) | Goal / item to review                                                                                                                                             |
| ------- | ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Kickoff | Sep 29–Oct 4 | Kickoff meeting and project discussion. Familiarize with data model.                                                                                              |
| 1       | Oct 5–11     | Project planning. Read data upload guide and controlled vocabulary. Initiate OBO/non-OBO candidate inventory (S1).                                                |
| 2       | Oct 12–18    | Finish fallback table (S1). Select 1 paper for each biomarker entity type to discuss curation flow (S2).                                                          |
| 3       | Oct 19–25    | Draft `study-type` + `measurement/sample-preparation` + `treatment` routes and prompts, enumerate required/optional annotations.                                  |
| 4       | Oct 26–Nov 1 | Draft entity-specific questions and rules for single, multi-entity, and multicomponent biomarkers.                                                                |
| 5       | Nov 2–8      | Draft the BEST `biomarker-role` guiding questions, routes, and prompts, enumerate required/optional annotations.                                                  |
| 6       | Nov 9–15     | Draft `biomarker-entity-type`, `specimen` routes and prompts, enumerate required/optional annotations.                                                            |
| 7       | Nov 16–22    | Design and implement evidence requirements, missingness states, and escalation rules. Develop supporting scripts to prepare packaging into a harness.             |
| 8       | Nov 23–29    | Revise the complete protocol from worked examples; document schema, open decisions, and implementation. Optional script/skill work only if the protocol is ready. |
| Handoff | Nov 30       | Submit the decision table, protocol, worked examples, and a short list of remaining questions.                                                                    |

Weekly goals are checkpoints for discussion and progress tracking. Keep all prompts, schema, and decisions with their examples in the repository and **keep them updated through regular commits**.

## Fall handoff goals

Another curator should be able to take an unfamiliar molecular biomarker paper, follow the protocol, identify the supported claim and BEST role, choose defensible identifiers, and see exactly which context was reported or missing.

The second semester can build and evaluate the harness from this specification. We will scope that implementation after reviewing the Fall protocol and its worked examples.

## References

- [BiomarkerKB data submission and upload guide](https://wiki.biomarkerkb.org/Data_Submission/Data_Upload)
- [BiomarkerKB data portal](https://data.biomarkerkb.org/)
- [BiomarkerKB controlled vocabulary](https://github.com/clinical-biomarkers/biomarker-controlled-vocabulary)
- [FDA–NIH BEST biomarker categories](https://www.ncbi.nlm.nih.gov/books/NBK326791/)
- [OBO Foundry ontology registry](https://obofoundry.org/)
