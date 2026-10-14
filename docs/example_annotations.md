# Example Annotations for Project Development

This document contain reference materials useful for refining biomarker annotations.

## Biomarker role system prompt (adapted from CarboCurator)

> You are triaging a glycan biomarker article by intended USE (biomarker role). Role = how the
> glycan measurement is meant to be applied to indicate a particular biological state or condition. Roles are NON-exclusive: a paper
> may support several.
>
> The 7 roles and their tests:
>
> - diagnostic: detects/confirms current disease presence, or classifies a subtype (cases vs controls).
> - susceptibility: future disease risk in subjects who are DISEASE-FREE at baseline.
> - prognostic: future clinical event (recurrence/progression/death) in subjects who ALREADY have the disease.
> - predictive: who benefits or is harmed differently AFTER a specific product/therapy/exposure (needs biomarker+ vs biomarker- comparison).
> - response: a biological CHANGE measured AFTER exposure to a product/agent (pharmacodynamic).
> - safety: toxicity/adverse effect linked to a product/exposure.
> - monitoring: the glycan is measured REPEATEDLY over time to track status/trajectory.
>
> You will receive the article sentences <S:n>...</S:n> and a list of candidate glycan entities.
> Decide which roles the article actually supports, and for each supported role list the glycan
> entities (copied EXACTLY from the provided list) that play that role. Omit roles with no support.
> Also flag multicomponent = true if two or more glycans are combined into a single panel/score/model.
>
> Return STRICT JSON only:

```json
{
    "roles": [
        {
            "role": "<one of the 7>",
            "glycans": ["<exact entity>", ...],
            "rationale": "<short>"
        }
    ],
    "multicomponent": <true|false>
}
```

## Glycan biomarker text mining data class (adapted from CarboCurator)

You will need to change much of the example material based on the new use case. Use your own judgement. Example classes to reuse: `RelationCandidate`, `MappedRelation`.
Concepts you might consider adding:

- biomarker entity type and their corresponding annotations
- where the samples were drawn from (e.g., what anatomical entity/tissue? Is it specific to the disease like tumor biopsies?)
- what was the process of sample preparation/processing (e.g., adding something? Extracting a certain cell type? Extracting certain proteins? Extracting extracellular vesicles? Separation by chromatography?)
- what was the method of measurement (RNA-sequencing? ELISA? Binding assay? lateral flow assay? HPLC? MS-MS? LC-MS? spectrometry?)?

```python
# Enums

Decision = Literal["RELEVANT", "IRRELEVANT"]
StudyType = Literal["case-control", "cohort", "cross-sectional", "intervention", "review", "meta-analysis", "editorial", "other"]
Direction = Literal["increased", "decreased", "absent", "novel_presence", "associated"]
TextMode = Literal["abstract", "fulltext", "chunked"]
StrengthLabel = Literal["weak", "moderate", "strong"]

# Records

class IDs(TypedDict, total=False):
processing_id: str
pmid: Optional[str]
pmcid: Optional[str]

class Doc(TypedDict, total=False):
title: str
abstract: str
metadata: Dict[str, Any]

class Flags(TypedDict, total=False):
has_pmcid: bool
can_use_fulltext: bool
needs_fulltext: bool
discard: bool
text_mode: TextMode # "abstract" | "fulltext" | "chunked"
fulltext_license: Optional[str]

class Provenance(TypedDict, total=False):
ingest_hash: str
source_line_no: Optional[int]
article_index: Optional[int]

class ScreeningFlags(TypedDict, total=False):
needs_fulltext: bool

class Screening(TypedDict, total=False):
decision: Decision
reason: str
study_type: StudyType
flags: ScreeningFlags
confidence: float # deterministic feature score in [0,1]

class Section(TypedDict, total=False):
id: str # e.g., "RESULTS.2"
title: Optional[str]
text: str # concatenated sentences
path: List[str] # hierarchy, e.g., ["RESULTS", "Subsection"]

class Corpus(TypedDict, total=False):
Title: Optional[str]
Abstract: Optional[Dict[str, Any]]
Introduction: Optional[Dict[str, Any]]
Methods: Optional[Dict[str, Any]]
Results: Optional[Dict[str, Any]]
Discussion: Optional[Dict[str, Any]]
SupplementaryInformation: Optional[Dict[str, Any]]
License: Optional[str]

class SentenceIndexed(TypedDict, total=False):
section_id: str # matches Section.id
sentence_index: int # global <S:n> index across entire document
char_start: int
char_end: int
global_index: int # alias for sentence_index (kept for backwards compatibility)
locator_key: str # canonical locator key "<S:n>@SECTION"

# New ver. locator record for corpus_indexed map includes the sentence text itself

class Locator(TypedDict, total=False):
section: str # SECTION label (e.g., "RESULTS")
subheading: Optional[str] # lowest-level title if present
paragraph_idx: int
sentence_idx: int
char_start: int
char_end: int
sentence: str # the actual sentence text
global_index: int
locator_key: str

class Span(TypedDict, total=False):
section_id: str
sentence_index: int
start: int
end: int

class Entity(TypedDict, total=False):
glycan_structure_term: str # the matched glycan structure text
non_structural_descriptor: Optional[str] # e.g., 'large', 'WA-binding', 'Fc-specific'
alignment: Literal["whole_structure", "substructure", "core"]
aglycon: Literal["R", "Ser/Thr", "Asn", "Cer", "free_glycan", "other"]
chemical_structure: Optional[str] # IUPAC-like sequence if explicitly provided
spans: List[Span] # locator references
evidence_sentence: Optional[str] # the actual sentence text where this glycan was first mentioned

class GlycanEntityMetadata(TypedDict, total=False):
glycan_structure_term: Optional[str]
non_structural_descriptor: Optional[str]
alignment: Optional[Literal["whole_structure", "substructure", "core"]]
aglycon: Optional[Literal["R", "Ser/Thr", "Asn", "Cer", "free_glycan", "other"]]

class Metric(TypedDict, total=False):
name: str # e.g., "AUC", "OR", "HR", "p"
value: Optional[float]
raw: Optional[str] # preserve original token if parsing is partial

class RelationCandidate(TypedDict, total=False):
glycan: str # surface or provisional norm_id
biomarker_role: List[str] # non-exclusive BEST roles, e.g. ["diagnostic","monitoring"]
role_annotations: Dict[str, Any] # role-specific fields (keys per roles.ROLE_SPECS)
is_multicomponent: bool # part of a multi-biomarker panel/score
change: Direction # "increased"/"decreased"/...
disease: str # surface or provisional norm_id
specimen: Optional[str]
specimen_annotation: Optional[str] # disease/state qualifier stripped off the specimen
sample_preparation: Optional[str] # free text: collection -> measurement processing
species: Optional[str]
protein_name: Optional[str] # HGNC notation preferred, else common name
evidence_sentence_indexes: List[int] # sentence numbers from corpus_indexed keys
evidence_locators: List[SentenceIndexed] # populated post-processing
metrics: List[Metric]
measurement_method: List[str] # instrument/assay used, not the statistical model

class Candidates(TypedDict, total=False):
entities: List[Entity]
relations: List[RelationCandidate]
red_flags: List[str] # any heuristic concerns

class SpecimenMapping(TypedDict, total=False):
category: Optional[Literal["tissue", "cell_line", "fluid", "other"]]
original: Optional[str]
annotation: Optional[str] # disease/state qualifier split off the specimen ("tumor")
mapped_name: Optional[str]
mapped_id: Optional[str]
ontology: Optional[str]
source: Optional[Literal["extracted", "evidence_backfill", "validator_fix"]]

class MappedRelation(TypedDict, total=False):
glycan_name: Optional[str] # original surface form
glycan_mapped_name: Optional[str] # preferred name from ontology
glycan_id: Optional[str] # bGSL term id or placeholder
glycan_entity_metadata: Optional[GlycanEntityMetadata] # carry entity metadata (descriptor/alignment/aglycon)
disease_name: Optional[str] # original disease text
disease_mapped_name: Optional[str] # preferred disease label
disease_id: Optional[str] # DOID
specimen: Optional[SpecimenMapping] # consolidated specimen mapping info
sample_preparation: Optional[str] # free text: collection -> measurement processing
species_name: Optional[str] # original species text
species_mapped_name: Optional[str] # canonical taxonomy label
species_id: Optional[str] # NCBI TaxID if available
protein_name: Optional[str] # original protein text
protein_mapped_name: Optional[str] # canonical protein label
protein_id: Optional[str] # UniProt accession, or GO id for a complex
protein_ontology: Optional[str] # "UniProt" (single gene) or "GO" (complex)
biomarker_role: List[str] # non-exclusive BEST roles
role_annotations: Dict[str, Any] # role-specific fields (keys per roles.ROLE_SPECS)
is_multicomponent: bool # part of a multi-biomarker panel/score
direction: Direction
evidence_locators: List[SentenceIndexed]
metrics: List[Metric]
measurement_method: List[str] # instrument/assay used, not the statistical model
notes: Optional[str] # mapping conflict notes

class ValidatedRelation(TypedDict, total=False):
relation: MappedRelation
valid: bool
issues: List[str]

class CleanedRelation(TypedDict, total=False):
relation: MappedRelation # de-duplicated & normalized
cluster_id: str # intra-article dedup group

class EvidenceRecord(TypedDict, total=False):
locator: SentenceIndexed
sentence_index: int
sentence_preview: Optional[str] # optional (OA or short quote policy)
score: float # 0–1 numeric
label: StrengthLabel # weak/moderate/strong

class LLMLog(TypedDict, total=False):
node: str
tool_calls: Optional[List[Dict[str, Any]]]
metadata: Optional[Dict[str, Any]]

class ScoredRelation(TypedDict, total=False):
relation: MappedRelation
evidence: List[EvidenceRecord]
aggregate_score: float
label: StrengthLabel

class Outputs(TypedDict, total=False):
curations_path: Optional[str] # JSONL output path
approvals_path: Optional[str]
runlog_path: Optional[str]
violations_path: Optional[str]
```

## Organized biomarker role-specific annotations (adapted from CarboCurator)

I am 90% sure you will need to visit the BEST resource and understand each role yourself to judge annotations suggested by AI.
Think about:

- Are there annotations under the same role that are redundant? (Yes)
- Similar-sounding concepts: exposure agent, environmental agent, treatment, therapy, medical intervention

```python
ROLE_SPECS: Dict[str, Dict[str, Any]] = {
    "diagnostic": {
        "label": "diagnostic",
        "definition": (
            "A diagnostic biomarker detects or confirms the presence of a disease/condition, "
            "or identifies a disease subtype, at the time of measurement — a current-state "
            "classification (cases vs controls, or subtype vs subtype)."
        ),
        "must_prove": (
            "The glycan distinguishes disease from non-disease (or one subtype from another) "
            "in samples measured at a single, current timepoint."
        ),
        "guided_questions": [
            "Which disease/condition is being detected, and does the glycan mark its presence/absence or a subtype?",
            "What is the comparison group (healthy controls, benign disease, another subtype)?",
            "Is there a reference/gold-standard diagnosis the biomarker is checked against?",
            "Is any diagnostic performance reported (sensitivity, specificity, AUC, PPV/NPV)?",
        ],
        "required_fields": [
            ("target_condition", "the disease/condition being diagnosed"),
            ("diagnostic_task", "'presence_absence' or 'subtype_classification'"),
            ("comparison_group", "what the cases are compared against (controls / other subtype)"),
        ],
        "optional_fields": [
            ("case_definition_or_reference", "reference/gold-standard diagnosis used"),
            ("disease_subtype", "subtype identified, if any"),
            ("disease_stage_or_severity", "stage/grade/severity context"),
            ("control_type", "healthy / benign / other-disease control"),
            ("diagnostic_performance", "sensitivity/specificity/AUC/PPV/NPV as stated"),
        ],
    },
    "susceptibility": {
        "label": "susceptibility/risk",
        "definition": (
            "A susceptibility/risk biomarker indicates the potential to DEVELOP a disease in a "
            "subject who does NOT currently have clinically apparent disease — future risk in the "
            "disease-free. It is distinct from prognostic, which applies after disease is present."
        ),
        "must_prove": (
            "Subjects are disease-free at baseline and the glycan is linked to future/incident "
            "onset of the target condition."
        ),
        "guided_questions": [
            "Are the subjects free of the target disease at baseline?",
            "Which future/incident disease does the glycan predict the risk of?",
            "In which direction does the glycan move risk (higher vs lower), and in which at-risk population?",
            "Is a risk time horizon or a reference/low-risk comparison group given?",
        ],
        "required_fields": [
            ("target_future_condition", "the incident disease being risk-stratified"),
            ("baseline_disease_absence", "statement that subjects are disease-free at baseline"),
            ("risk_direction", "increased vs decreased future risk"),
            ("at_risk_population", "the disease-free population studied"),
        ],
        "optional_fields": [
            ("risk_event", "the incident-disease event definition"),
            ("comparison_group", "low-risk / reference group"),
            ("risk_time_horizon", "follow-up window for incident risk"),
            ("exposure_agent", "exposure the risk marker reflects, if any (e.g. tobacco metabolite)"),
            ("risk_estimate", "absolute/relative risk as stated"),
        ],
    },
    "prognostic": {
        "label": "prognostic",
        "definition": (
            "A prognostic biomarker indicates the likelihood of a future clinical event "
            "(recurrence, progression, death, new event) in subjects who ALREADY have the "
            "disease/condition, measured at a defined baseline."
        ),
        "must_prove": (
            "Subjects already have the disease and the glycan is linked to a future clinical "
            "outcome over follow-up."
        ),
        "guided_questions": [
            "Do the subjects already have the disease at the baseline measurement?",
            "Which future event/outcome is predicted (survival, recurrence, progression, time-to-event)?",
            "Over what follow-up period, and in which direction (better vs worse outcome)?",
            "Is a hazard/odds ratio, survival metric, or risk-group cutoff reported?",
        ],
        "required_fields": [
            ("existing_condition", "the disease the subjects already have"),
            ("baseline_disease_state", "baseline disease stage/state at the measurement (disease PRESENT at baseline)"),
            ("baseline_timepoint", "the defined baseline timepoint of measurement"),
            ("future_event_or_transition", "recurrence / progression / death / new event"),
            ("endpoint_of_interest", "the clinical endpoint measured"),
            ("event_direction", "glycan associated with better vs worse outcome"),
        ],
        "optional_fields": [
            ("follow_up_period", "length of follow-up"),
            ("disease_stage", "stage/grade"),
            ("disease_subtype", "subtype"),
            ("background_treatment", "standard-of-care/background therapy"),
            ("outcome_metric", "HR/OR/survival/recurrence/progression metric"),
            ("risk_group_cutoff", "threshold defining risk strata"),
        ],
    },
    "predictive": {
        "label": "predictive",
        "definition": (
            "A predictive biomarker identifies subjects more or less likely to experience a "
            "favorable or unfavorable EFFECT from a specific medical product or exposure, relative "
            "to biomarker-negative subjects. Evidence generally needs a comparison across "
            "biomarker groups (and, ideally, treatment vs control)."
        ),
        "must_prove": (
            "An exposure/product is present AND the glycan marks a DIFFERENTIAL benefit or harm "
            "between biomarker-positive and biomarker-negative subjects."
        ),
        "guided_questions": [
            "Which specific medical product / therapy / environmental agent is involved?",
            "Does the glycan mark a favorable or unfavorable effect of that product?",
            "What are the biomarker-defined groups being compared (biomarker+ vs biomarker-)?",
            "Is there a treated-vs-control (or alternative-intervention) contrast, i.e. an interaction effect?",
        ],
        "required_fields": [
            ("medical_product_or_exposure_agent", "the product/therapy/agent"),
            ("effect_type", "'favorable' or 'unfavorable' differential effect"),
            ("effect_endpoint", "the outcome the effect is measured on"),
            ("biomarker_defined_group", "the biomarker-positive group"),
            ("comparator_biomarker_group", "the biomarker-negative/other group"),
            ("treated_or_exposed_group", "the treated/exposed subject group"),
            ("interaction_claim", "explicit biomarker×treatment interaction (vs a mere prognostic association under treatment)"),
        ],
        "optional_fields": [
            ("control_or_alternative_intervention", "treatment vs control / alternative arm (when available)"),
            ("target_condition", "disease/prevention context"),
            ("dose", "dose/exposure level"),
            ("companion_diagnostic_flag", "used as a companion diagnostic"),
            ("biomarker_cutoff", "threshold defining biomarker positivity"),
        ],
    },
    "response": {
        "label": "response/pharmacodynamic",
        "definition": (
            "A response/pharmacodynamic biomarker shows a biological CHANGE after exposure to a "
            "medical product or environmental agent. A pharmacodynamic biomarker indicates "
            "biological activity but does not by itself establish efficacy or clinical benefit; "
            "when used as a trial endpoint it is a surrogate endpoint biomarker."
        ),
        "must_prove": (
            "An exposure/product is present AND the glycan changes after that exposure."
        ),
        "guided_questions": [
            "Which product/therapy/agent was administered, and was the glycan measured after exposure?",
            "In which direction and by how much did the glycan change after exposure?",
            "At what timing after exposure was it measured (and relative to a baseline)?",
            "Is the change presented as a surrogate endpoint for clinical benefit (candidate / reasonably-likely / validated)?",
        ],
        "required_fields": [
            ("medical_product_or_exposure_agent", "the product/therapy/agent"),
            ("exposure_status", "that the glycan is measured after exposure"),
            ("response_measurement", "what biological response is measured"),
            ("response_direction_or_magnitude", "direction/magnitude of change after exposure"),
        ],
        "optional_fields": [
            ("timing_after_exposure", "timepoint relative to exposure"),
            ("baseline_measurement", "pre-exposure baseline"),
            ("dose", "dose/exposure level"),
            ("target_engagement", "target-engagement/mechanistic pathway"),
            ("surrogate_endpoint_flag", "used as a surrogate endpoint"),
            ("surrogate_validation_level", "candidate / reasonably_likely / validated"),
            ("clinical_outcome_link", "link to a clinical outcome, if claimed"),
        ],
    },
    "safety": {
        "label": "safety",
        "definition": (
            "A safety biomarker is measured before or after exposure to a medical product or "
            "environmental agent to indicate the likelihood, presence, or extent of TOXICITY as "
            "an adverse effect."
        ),
        "must_prove": (
            "An exposure/product is present AND the glycan indicates toxicity/adverse effect "
            "(likelihood, presence, or extent)."
        ),
        "guided_questions": [
            "Which product/therapy/agent, and what toxicity or adverse effect is indicated?",
            "Does the glycan indicate the likelihood, the presence, or the extent of toxicity?",
            "Is it measured before exposure (to avoid harm) or after (developing toxicity)?",
            "Which organ system is affected, and in which direction does harm move?",
        ],
        "required_fields": [
            ("medical_product_or_exposure_agent", "the product/therapy/agent"),
            ("toxicity_concept", "the toxicity/adverse effect"),
            ("toxicity_status", "'likelihood' | 'presence' | 'extent'"),
            ("timing_relative_to_exposure", "before vs after exposure"),
        ],
        "optional_fields": [
            ("harm_direction", "direction of harm"),
            ("organ_system", "affected organ system"),
            ("toxicity_severity", "severity/grade"),
            ("dose", "dose/exposure level"),
            ("action_threshold", "threshold triggering action"),
            ("clinical_action", "dose adjustment / interruption / contraindication"),
        ],
    },
    "monitoring": {
        "label": "monitoring",
        "definition": (
            "A monitoring biomarker is measured REPEATEDLY over time to assess the status or "
            "trajectory of a disease, or the effect of an exposure/product. It is a temporal "
            "wrapper over diagnostic/prognostic/response/safety use — it requires serial measurement."
        ),
        "must_prove": (
            "The glycan is measured at multiple timepoints to track a disease/exposure trajectory."
        ),
        "guided_questions": [
            "Is the glycan measured repeatedly/serially (multiple timepoints), not just once?",
            "What is being monitored (disease burden, progression/recurrence, treatment response, toxicity)?",
            "What is the baseline value and the direction/rate/magnitude of change over time?",
            "At what interval is it monitored, and does a change trigger a defined action?",
        ],
        "required_fields": [
            ("repeated_measurement", "statement of serial/repeated measurement"),
            ("monitored_entity", "disease status / progression / response / toxicity being tracked"),
            ("change_direction", "direction of change over time"),
        ],
        "optional_fields": [
            ("timepoints", "the timepoints/schedule"),
            ("baseline_value", "baseline/prior value"),
            ("change_magnitude_or_rate", "magnitude or rate of change"),
            ("monitoring_interval", "interval between measurements"),
            ("medical_product_or_exposure_agent", "product/exposure being monitored, if any"),
            ("action_threshold", "threshold triggering a clinical action"),
        ],
    },
}
```
