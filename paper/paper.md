---
title: 'DBCLS BioHackathon 2026 report: Logica, an intelligent platform for semantic interoperability'
title_short: 'BioHackJP26: Logica'
tags:
  - Semantic interoperability
  - Clinical information models
  - Ontologies
  - Clinical AI
  - Workflows
# TODO: Replace these author and affiliation placeholders and confirm author order.
# Add verified ORCID, ROR, and CRediT roles when available.
authors:
  - name: 'Claude Nanjo'
    affiliation: 1
  - name: 'Toyofumi Fujiwara'
    affiliation: 2
  - name: 'Chang Sun'
    affiliation: 3
  - name: 'Michel Dumontier'
    affiliation: 3
  - name: 'Núria Queralt Rosinach'
    affiliation: 4
affiliations:
  - name: 'University of Utah'
    index: 1
  - name: 'National Institute of Genetics'
    index: 2
  - name: 'Maastricht University'
    index: 3
  - name: 'Leiden University Medical Center'
    index: 4
date: 18 September 2026
cito-bibliography: paper.bib
event: BH26JP
biohackathon_name: "DBCLS BioHackathon 2026"
biohackathon_url: "https://2026.biohackathon.org/"
biohackathon_location: "Matsuyama, Japan, 2026"
group: logica
git_url: https://github.com/biohackathon-japan/BH26-logica
authors_short: 'TODO: First author et al.'
---

<!-- Working manuscript populated from the full Logica write-up in the referenced
conversation. The reported artifacts and results have not been independently
checked against a run archive for this revision.
Before submission:
- Confirm author details, affiliations, contributions, funding, and acknowledgements.
- Supply verified references in paper.bib and cite them in the manuscript.
- Confirm the nine-stage nomenclature, model/runtime versions, and run identifier.
- Confirm how SerumCreatinine in CQL resolves to the Creatinine constraint set.
- Reconcile the quoted 1.1 mg/dL normalization example and evidence identifier with
  the evidence supporting the quoted 1.0-to-2.1 mg/dL answer.
- Confirm the relation between the September 17 run date and the September 11–14
  query interval, including the anchor for LAST_72_HOURS.
- Reconcile the trend statement with the quoted insufficient-data warning.
-->

# Abstract

Clinical AI depends on both the relevance of the information it receives and the meaning of that information. Logica is an interoperability platform that separates clinical meaning from physical data representation and separates language-model interpretation and reasoning from deterministic retrieval. A first language model interprets a clinical question through the COOL semantic graph. Explicit information requirements are compiled into Clinical Quality Language (CQL) over COOL, a logical representation of a query, bound to a physical backend (e.g., as a FHIR query or SQL query), and executed in code. Returned records are normalized into COOL evidence with source provenance before a second language model reasons over them. Deterministic calculations and citation-identifier validation make the resulting answer inspectable. We describe this architecture through a reported creatinine demonstration for a synthetic patient (Demo Patient A), in which a record containing 204 FHIR resources yielded 69 evidence items. The demonstration illustrates the pipeline and its retained artifacts as well as how the system works; however, further evaluation is needed to determine whether it actually improves the saliency of relevant clinical information or clinical reasoning accuracy compared with other approaches. The research hypothesis is that a sufficiently constrained semantic model improves context relevance, reasoning accuracy, and verifiability. Separating logical expressions from backend bindings also provides a basis for sharing clinical logic across institutions.

# Introduction

**Logica is an intelligent interoperability platform built around a simple idea: before AI can reason reliably over clinical data, the meaning of that data must be made explicit.**

A clinician can answer a question such as *“Why is this patient’s creatinine increasing?”* by opening the chart, finding the relevant laboratory values, medications, diagnoses, fluid balance, and clinical events, and interpreting them in context. The tempting AI equivalent is to place the electronic health record into a language model’s context and ask the same question.

That approach has a fundamental problem. Clinical records are not semantically uniform. The same clinical fact may appear as an order, an administration, a result, and a note. The same analyte may have different codes across institutions. Similar structures may carry different meanings, while equivalent concepts may be represented differently. Giving all of this directly to a language model forces the model to determine what the data means, decide what is relevant, reconcile representations, and perform clinical reasoning simultaneously. The resulting prose may sound convincing while being difficult to reproduce, inspect, or verify.

Logica separates those concerns.

Its pipeline makes clinical meaning explicit before reasoning occurs and keeps language models away from physical data retrieval. Of the nine stages in the current pipeline, only two require an LLM: one at the beginning to interpret the clinician’s question and one at the end to reason over normalized evidence. The stages between them are deterministic and inspectable.

The result is a chain:

**Clinical question → semantic interpretation → information requirements → logical query (CQL over COOL) → physical retrieval (FHIR query) → normalization → evidence → reasoning → cited answer**

Every intermediate artifact is retained. Consequently, an answer can be traced backward from a sentence, to evidence, to the source record, to the query that retrieved it, to the information requirement that requested it, and ultimately to the interpretation of the clinician’s question.

# Methods

## From a clinical question to explicit information requirements

Consider an actual Logica run for Demo Patient A on September 17, 2026:

> *Why is this patient's creatinine increasing?*

The first LLM does not inspect the patient's record or construct database queries. Instead, it navigates the **COOL semantic model**, a graph representation of the COOL model, through a fixed set of tools that allow it to search concepts, inspect types, traverse relationships, and examine constraint sets.

From the question it derives the intent:

> *Determine the likely cause(s) of the rising creatinine in this septic ICU patient.*

It identifies relevant concepts including **Acute Kidney Injury, Sepsis, Nephrotoxic Medication Exposure, Contrast Exposure, Hypotension, Fluid Status,** and laboratory measures related to renal function.

The LLM uses its medical knowledge to translate these concepts into explicit information requirements, specifying which clinical data classes are needed to address the question. For example:

```text
ir-1  SerumCreatinine  TREND     LAST_72_HOURS
ir-2  SerumCreatinine  BASELINE  PRE_ADMISSION_BASELINE
ir-3  EstimatedGFR     TREND
ir-4  BloodUreaNitrogen TREND
```

This is an important separation of responsibilities. The system decides what evidence is needed before reasoning about the evidence.

The requirements are deliberately recall-oriented: the planner can request more information than ultimately proves necessary, while later stages prune it. More importantly, the requirements are visible and editable. A clinician can inspect them, correct them, and rerun the pipeline while retaining the original run for comparison.

Relevance therefore becomes an explicit, testable property of the system rather than an emergent property of a large prompt.

## A semantic model rather than a proliferation of schemas

COOL does not create a separate structural class for every laboratory analyte. Creatinine, sodium, hemoglobin, and other quantitative laboratory results share a common clinical model:

```cool
SimpleLabResultQuantitative : SimpleLabResult<Quantity>
```

What makes a particular result creatinine is expressed through a **CCL constraint set**:

```ccl
constraints Creatinine : RoutineLabResult v1
    code        loinc#2160-0
    value.unit  'mg/dL'
```

The shared `RoutineLabResult` constraint set supplies the broader semantics of a laboratory result:

```ccl
constraints RoutineLabResult : SimpleLabResultQuantitative
    code                  coolvs/standard-lab-obs-quantitative   required
    value.unit            ucumvs/units-of-measure                required
    dataAbsentReason      coolvs/lab-null-flavor                 required
    interpretation        coolvs/abnormal-interpretation-numeric extensible
    referenceRange        0..1
    collected             1..1
    collected.on          1..1
    resulted              1..1
    performingLaboratory  1..1
    check                 collected.on <= resulted.on
```

This illustrates what Logica means by a **sufficiently constrained semantic model**.

A creatinine result is not merely a number stored in a field. It has an identity, terminology binding, unit, collection time, result time, provenance, and laboratory context, together with invariants that must hold. As the model documentation puts it:

> *A value with no collection time and no performing laboratory is not a result; it is a number.*

That distinction matters to AI because downstream reasoning can rely on these semantics rather than rediscovering or guessing them from raw records.

## Portable clinical logic

Information requirements are compiled into **CQL 2.0.0 expressed over COOL**, rather than directly over FHIR or a database schema:

```cql
define "SerumCreatinine Trend 1":
  [SerumCreatinine: code in "SerumCreatinine code"] O
    where O.effectiveTime during
      Interval[@2026-09-11T12:00Z, @2026-09-14T12:00Z]
    sort by effectiveTime
```

The expression is compiled by the reference CQL translator against model information generated from the COOL graph. In the reported demonstration, the reference translator accepted the expression. This compilation step makes the translation artifact available for inspection.

This establishes a critical architectural boundary.

**COOL says what something is. CCL says what it may contain and what constraints must hold. CQL says what information is being requested.**

None of those artifacts says where the information is stored.

The physical representation lies on the other side of a deliberate interoperability seam.

## Separating logical meaning from physical storage

Once the CQL expression has been compiled, Logica extracts the data retrieval requirements from the compiled ELM and translates them into backend-specific queries. For the FHIR backend, it generates FHIR search requests and then executes those requests against the server.

Against the demonstration FHIR R5 backend, a logical request for serum creatinine becomes a real search query such as:

```text
Observation?
  patient=demo-001
  &code=http://loinc.org|2160-0
  &date=ge2026-09-11T12:00:00Z
  &date=le2026-09-14T12:00:00Z
  &_sort=date
```

The query is retained and exposed in the interface, allowing a reviewer to inspect exactly what was sent rather than accepting a returned record count on trust.

FHIR, however, is not part of the clinical logic. The mapping from COOL concepts to FHIR resources is configuration data. Likewise, normalization from returned FHIR resources into COOL is defined declaratively using FHIRPath.

A FHIR observation containing `1.1 mg/dL`, for example, becomes a COOL quantitative observation while retaining provenance to its source:

```text
Observation/demo-001-obs-cr-3
```

There is no clinical branch in the normalization engine saying “if this is creatinine, do X.” New clinical classes are introduced through models, constraints, bindings, and mappings rather than new application logic.

The same compiled CQL could therefore be given to a different physical adapter. A MIMIC implementation could translate the same logical retrieve into SQL against `labevents`; another implementation could target a proprietary clinical data warehouse.

The intended adapter contract preserves the clinical logic. Execution against a MIMIC/SQL backend remains a proposed extension rather than a result demonstrated here.

# Results

## Reducing the record before reasoning

This separation is intended to increase the **density of clinically relevant information in the reasoning context**. By planning retrieval around explicit information requirements, Logica aims to retain the evidence needed to address the question while excluding unrelated chart content.

Demo Patient A contained **204 FHIR resources**. The Logica run assembled **69 evidence items** corresponding to the information requirements for investigating the creatinine change, including creatinine and eGFR measurements, urine output, nephrotoxic exposures, and diagnoses.

The hypothesis is that this planning step produces a context with a higher proportion of relevant information while preserving the evidence needed for clinical reasoning. The demonstration illustrates this approach, but does not yet test that hypothesis. Because source resources and evidence items are different representations, their counts alone cannot establish relevance density or evidence coverage.

The architectural distinction is that **evidence selection occurs before reasoning**, making its contribution to context relevance explicit and independently evaluable.

Moreover, the final model is not asked simultaneously to search the chart, determine which facts matter, normalize their representations, and reason clinically. It receives evidence selected according to explicit information requirements and represented in a known semantic form.

That separation also makes errors diagnosable. If an answer is poor, the system can distinguish between two fundamentally different failures:

**Did the planner fail to request the necessary evidence, or did the reasoner misinterpret evidence that was correctly retrieved?**

A monolithic prompt containing the entire chart makes those failure modes extremely difficult to distinguish.

## Evidence-bound reasoning

Only after retrieval and normalization does the second LLM enter the pipeline.

The model receives the normalized evidence together with deterministic calculations. Arithmetic, trend calculations, and similar operations are performed in code rather than delegated to the language model.

The reasoner can then produce statements such as:

> *Serum creatinine increased from 1.0 mg/dL to 2.1 mg/dL over 3 days, indicating worsening renal function.*

The statement is associated with the evidence identifiers supporting it:

```text
ev-rs-2-demo-001-obs-cr-0
ev-rs-3-demo-001-obs-cr-3
ev-rs-3-demo-001-obs-cr-4
```

Those citations are checked against the retrieved evidence identifiers. If the model invents an evidence identifier, the citation is removed and the corresponding statement is marked unsupported. Identifier validation establishes that cited evidence exists in the retrieved set; it does not by itself establish that the evidence supports the clinical claim.

The system also preserves negative information about its own retrieval. In this run, for example, it reported:

> *ir-1 SerumCreatinine TREND: Only 3 data points are available, insufficient for a clear trend analysis.*

The goal is therefore not merely to generate an answer. It is to generate an answer whose relationship to the available evidence can be inspected.

# Discussion

## Interoperability as the foundation for reusable intelligence

This architecture has a second consequence: **clinical logic becomes shareable**.

A decision rule written directly against a hospital's physical schema encodes local assumptions: table names, item identifiers, coding practices, historical migrations, null conventions, and other implementation details. Moving that rule to another institution requires rewriting those assumptions.

A rule written against COOL instead asks for concepts such as:

```text
[SerumCreatinine]
```

and expresses the clinical logic independently of the physical representation.

At a new institution, the local work is concentrated at the interoperability boundary: bind the semantic concept to the local representation and provide the mapping back into COOL. The clinical expression itself remains unchanged.

This is also what makes semantic models useful to AI agents.

An agent operating directly over a physical schema must learn the conventions of every institution and has limited ability to recognize when it has misunderstood them. An agent operating over a semantic graph instead navigates relationships explicitly asserted by the model. The graph can state how creatinine relates to renal function, how laboratory observations are structured, what terminology constrains them, and how related concepts connect.

The architecture aims to preserve those semantics when the underlying storage changes, provided that each site's bindings and normalization mappings implement the same semantic contract.

## The hypothesis Logica is designed to test

The central hypothesis behind Logica is:

> **A sufficiently constrained, semantically interoperable clinical model improves the relevance of information supplied to AI and the accuracy and verifiability of the reasoning performed over it.**

The front half of the pipeline provides a way to test **relevance**. Instead of guessing against an unfamiliar schema or searching an entire chart, the system determines what information is required by navigating an explicit semantic model and retrieves only evidence corresponding to those requirements.

The back half provides a way to test **reasoning accuracy and verifiability**. Evidence reaches the reasoner in a consistent semantic representation with provenance attached, deterministic calculations already performed, and explicit mechanisms for distinguishing supported from unsupported assertions.

Logica is therefore not simply an LLM interface to an EHR, nor is it another integration engine.

It is an attempt to make **semantic interoperability the substrate on which intelligent clinical systems operate**.

The nine-stage pipeline is the experimental apparatus for testing that proposition. Interpretation, information requirements, logical queries expressed as CQL, physical queries, retrieved records, normalized COOL evidence, calculations, reasoning, and citations remain available after every run.

That inspectability is not incidental. During development, it has repeatedly exposed runs that appeared successful at the final-answer level but were semantically or operationally empty upstream.

The larger proposition is that **AI does not eliminate the need for interoperability; it makes rigorous interoperability more important**. If clinical meaning is explicit, constrained, portable, and machine-readable, AI can operate over it while deterministic systems retain responsibility for retrieval, validation, computation, and provenance.

That is the role of Logica: **to provide the semantic and computational bridge between clinical intent, heterogeneous health data, and trustworthy AI reasoning.**

## Limitations and planned evaluation

The reported example is a demonstration run for one clinical question on one demonstration patient. It illustrates execution and inspectability, but does not establish improved context relevance, reasoning accuracy, clinical safety, or portability across institutions. The FHIR backend is part of the reported demonstration; MIMIC/SQL is a possible future adapter.

The semantic model and its mappings remain potential sources of error. Relevant evidence can be absent because the interpretation or requirements omit a concept, because a binding selects the wrong records, or because normalization fails to preserve meaning. Consistent representation does not remove missing data, incorrect source records, or uncertainty in clinical interpretation. A reasoner can also misinterpret evidence even when every cited identifier is valid.

A future evaluation should assess retrieval and reasoning separately. Clinician-reviewed questions and reference evidence sets could support measurement of retrieval relevance and coverage. Reasoning evaluation should assess factual correctness, support for claims, and handling of missing or conflicting evidence. Comparisons with direct chart prompting and retrieval without semantic normalization would help determine which architectural components contribute to any observed benefit. These comparisons are proposed work, not completed experiments reported here.

Portability should likewise be evaluated by executing the same clinical expressions against independently implemented backend bindings and comparing the meaning and completeness of the normalized evidence. Successful compilation and a working FHIR demonstration alone do not establish equivalent behavior across data sources.

## Acknowledgements

We thank the Database Center for Life Science (DBCLS), Japan, for organizing BioHackathon 2026 and supporting this work. We also acknowledge the University of Utah for its contributions to clinical informatics and semantic interoperability, and for providing  support that enabled the presenting author’s participation in the BioHackathon.

We also acknowledge Health Level Seven International (HL7) and its standards development community for their contributions to healthcare interoperability, particularly the FHIR and Clinical Quality Language (CQL) standards used in this prototype.

# References

1. Health Level Seven International. (2023). *FHIR specification: Release 5 (version 5.0.0).* [Specification](https://hl7.org/fhir/R5/).

2. Health Level Seven International. (2026). *Clinical Quality Language specification: Release 2 trial-use publication (version 2.0.0).* [Specification](https://cql.hl7.org/R2/).

3. Health Level Seven International. (2020). *FHIRPath normative release (version 2.0.0).* ANSI/HL7 NMN R1-2020. [Specification](https://hl7.org/fhirpath/N1/).

4. Oniki, T. A., Coyle, J. F., Parker, C. G., & Huff, S. M. (2014). Lessons learned in detailed clinical modeling at Intermountain Healthcare. *Journal of the American Medical Informatics Association, 21*(6), 1076–1081. [doi:10.1136/amiajnl-2014-002875](https://doi.org/10.1136/amiajnl-2014-002875).

5. Liu, N. F., Lin, K., Hewitt, J., Paranjape, A., Bevilacqua, M., Petroni, F., & Liang, P. (2024). Lost in the middle: How language models use long contexts. *Transactions of the Association for Computational Linguistics, 12*, 157–173. [doi:10.1162/tacl_a_00638](https://doi.org/10.1162/tacl_a_00638).

6. Xiong, G., Jin, Q., Lu, Z., & Zhang, A. (2024). Benchmarking retrieval-augmented generation for medicine. In *Findings of the Association for Computational Linguistics: ACL 2024* (pp. 6233–6251). Association for Computational Linguistics. [doi:10.18653/v1/2024.findings-acl.372](https://doi.org/10.18653/v1/2024.findings-acl.372).

