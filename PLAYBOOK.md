# GO-TS Playbook: Graph-Ontology → Terminology Server

**The authoritative guide to building and operating a FHIR R4 terminology service for Australian clinical and
medicines vocabularies, seeded from a Graph-Ontology T0 release.** It provides:
- versioned **lookup, validation, subsumption, expansion** (including SNOMED CT expression constraints) and
  **translation**;
- a **free-text matcher**;
- **tier and provenance on every mapping** it serves.

Version 1 · 24 Sep 2026 · Arepo-Medtech/GO-TS

This playbook is self-contained. Its only dependency inside this repository is **`T0.md`**: the frozen, validated
Graph-Ontology release, including the SSSOM export of its mappings, that this server loads. It assumes T0 is complete
and that this track runs in isolation from the other Graph-Ontology tracks. External dependencies are public
standards, open-source software and, optionally, national terminology services.

> Status: plan. Nothing described here is built yet.

---

## Contents

1. [Mission and definition of done](#1-mission-and-definition-of-done)
2. [Non-negotiable rules](#2-non-negotiable-rules)
3. [Standards this server implements](#3-standards-this-server-implements)
4. [Build strategy: reuse, don't reimplement](#4-build-strategy-reuse-dont-reimplement)
5. [Architecture](#5-architecture)
6. [Inputs from T0](#6-inputs-from-t0)
7. [Stages and gates](#7-stages-and-gates)
8. [Code systems: identity, versions, content](#8-code-systems-identity-versions-content)
9. [Value sets and expansion](#9-value-sets-and-expansion)
10. [Concept maps and translation](#10-concept-maps-and-translation)
11. [The free-text matcher](#11-the-free-text-matcher)
12. [Testing and validation](#12-testing-and-validation)
13. [Performance and scaling](#13-performance-and-scaling)
14. [Security and licensing](#14-security-and-licensing)
15. [Release management](#15-release-management)
16. [Scorecard](#16-scorecard)
17. [Operations](#17-operations)
18. [Risk and failure-mode register](#18-risk-and-failure-mode-register)
19. [Roles and effort](#19-roles-and-effort)
20. [Prompt library](#20-prompt-library)
21. [Decisions to make](#21-decisions-to-make)
22. [References](#22-references)
- [Appendix A: Hand-check protocol](#appendix-a-hand-check-protocol)
- [Appendix B: Templates](#appendix-b-templates)

---

## 1. Mission and definition of done

### 1.1 Mission
Be the dependable place where applications, pipelines and people turn **phrases and codes into correct, current,
versioned codes**, and translate between Australian and international vocabularies. Every translation says how far
it can be trusted and where it came from.

### 1.2 Users
- clinical and analytics applications (FHIR clients);
- ETL pipelines (batch translation and validation);
- data stewards and terminologists (review of candidate mappings);
- search and question-answering tools (the matcher).

### 1.3 Definition of done (v1)
1. The implemented FHIR terminology operations pass the **HL7 FHIR Terminology Ecosystem** test cases that apply to
   them (§12.1).
2. SNOMED CT-AU expansions **match a reference server** at the same edition, for the differential test set (§12.2).
3. `$translate` returns **tier and provenance** on every target, and serves only clinical-grade maps by default
   (§10).
4. The matcher reaches **precision@1 ≥ 0.95** on clinical scopes at a calibrated abstain threshold (§11.6).
5. Latency service-level objectives are met under the expected load (§13).
6. **Licence gating is verified** with an unlicensed test client (§14).

---

## 2. Non-negotiable rules

| # | Rule | Enforced by |
|---|---|---|
| T1 | **Every response is versioned:** code system URI and version, and the T0 release id. | response middleware; tests |
| T2 | **Don't reimplement SNOMED CT logic.** Expression-constraint (ECL) evaluation and description-logic subsumption come from a proven SNOMED server. | architecture (§4) |
| T3 | **Clinical and candidate maps never mix.** `$translate` uses clinical maps by default; candidate maps only on explicit request, to authorised roles, flagged. | ConceptMap separation; authorisation |
| T4 | **Tier and provenance on every mapping target.** | ConceptMap extensions (§10.3) |
| T5 | **The matcher never invents a code.** It returns candidates from its index, or abstains. | matcher contract |
| T6 | **Serve only what the licences permit, to whom they permit.** | authorisation per code system; `t0 licences check --track T4` |
| T7 | **Retired codes are never silently dropped or silently current.** `$validate-code` says inactive, with replacements. | tests |
| T8 | **Pinned releases; reproducible expansions.** An expansion is identified by (value set, version, code system versions); its hash is stable. | expansion-hash regression tests |
| T9 | **Gates refuse.** | release checklist |

---

## 3. Standards this server implements

| Standard | What it governs here |
|---|---|
| **HL7 FHIR R4, terminology module** | CodeSystem, ValueSet, ConceptMap, TerminologyCapabilities; operations `$lookup`, `$validate-code`, `$subsumes`, `$expand`, `$translate`, optionally `$closure` |
| **HL7 FHIR Terminology Ecosystem IG** | conformance requirements and **test cases** for terminology servers; the test runner is built into the FHIR validator |
| **SNOMED CT Expression Constraint Language (ECL)** | value-set definitions over SNOMED CT. Version 2.2 was published in November 2023; version 2.3 (March 2026) adds an active filter wildcard, `refsetContainingAny` and simpler string matching. Declare which version the server supports |
| **SNOMED CT FHIR conventions** | system `http://snomed.info/sct`; version URIs `http://snomed.info/sct/{module}/version/{yyyymmdd}`; implicit value sets `http://snomed.info/sct?fhir_vs=ecl/{ECL}`, `…?fhir_vs=isa/{id}`, `…?fhir_vs=refset/{id}` |
| **SNOMED CT-AU** | the Australian edition, module `32506021000036107`, including the **Australian Medicines Terminology (AMT)**; published through the National Clinical Terminology Service (NCTS) of the Australian Digital Health Agency |
| **HL7 AU Base and AU Core** | Australian FHIR profiles and the code-system and value-set URIs they use; follow them where they define URIs |
| **LOINC** (`http://loinc.org`), **UCUM** (`http://unitsofmeasure.org`), **RxNorm** (`http://www.nlm.nih.gov/research/umls/rxnorm`) | code-system identity |
| **SSSOM** | the input format of mappings (from T0) |
| **SMART Backend Services** | system-to-system authorisation (§14) |

---

## 4. Build strategy: reuse, don't reimplement

| Capability | Recommended engine | Why |
|---|---|---|
| SNOMED CT-AU (and AMT): ECL, subsumption, lookup, designations | **front an existing SNOMED server:** the national service (Ontoserver-based) where access is licensed, **or** self-host **Snowstorm** (SNOMED International, Apache-2.0, Elasticsearch-backed) loaded with the AU RF2 release | ECL and description-logic reasoning are hard to get right; mature servers are tested against the specification |
| Other code systems (LOINC, PBS, MBS, RCPA sets, UCUM, graph-specific systems), value sets and ConceptMaps | **HAPI FHIR JPA server** (Apache-2.0), or the SNOMED server if it supports them | a standard FHIR store for the generated resources |
| Graph-Ontology mappings with tier and provenance | **generated here** from the T0 SSSOM export as ConceptMaps | this is the unique content |
| Free-text matcher | **built here** (§11) | national servers offer text search, but not over Graph-Ontology's full synonym set with abstention |
| Routing, versioning, tier extensions, licence gating | **a thin façade** (for example FastAPI) in front of the engines | one endpoint and policy point |

**Hybrid default:** SNOMED operations are proxied to the SNOMED engine; everything else goes to HAPI; the façade adds
versioning, tier extensions and authorisation.

---

## 5. Architecture

```
 FHIR clients / ETL / tools
   │  HTTPS + OAuth2 (SMART Backend Services for systems; OIDC for people)
   ▼
 FAÇADE (FastAPI)
   │  auth + licence gating per code system → routing → version stamping → tier extensions → caching → audit
   ├── SNOMED engine (Snowstorm or national server)   $lookup/$validate-code/$subsumes/$expand for SCT, ECL
   ├── HAPI FHIR JPA (Postgres)                        CodeSystems, ValueSets, ConceptMaps; $translate; others
   └── MATCHER                                         /match: BM25 + embeddings + scope filter + rerank + abstain
        └── index built from T0 `synonym` (local, licensed text)
   ▼
 Observability: request logs (no PHI), metrics, traces; release registry
```

---

## 6. Inputs from T0

| T0 artefact (see `T0.md`) | Used for |
|---|---|
| `t0.lock` → release | the release id stamped on every response |
| `manifest.json` source pins | code-system versions (SNOMED CT-AU edition date, LOINC version, and others) |
| `sssom/go_mappings.ids.sssom.tsv` (and `.full`, locally) | clinical ConceptMaps |
| `sssom/go_mappings.candidates.sssom.tsv` | candidate ConceptMaps (restricted) |
| `config/predicate_properties.yaml` | `sssom_predicate` → ConceptMap equivalence (§10.2) |
| `register/route_register` | tier, Wilson lower bound, evidence for each route (extensions) |
| `views/synonym` | the matcher index (local only) |
| `views/equivalence_group` | equivalent-code sets; conflict flags (never served as `equivalent` when conflicting) |
| `graph.duckdb` nodes for PBS, MBS, RCPA and other non-SNOMED systems | generated CodeSystem content |
| `config/licence_matrix.yaml` | serving, display and attribution permissions per code system |
| `quiz/items` tagged for translation and matching | regression tests |

The SNOMED engine loads the **RF2 release pinned in the T0 manifest**, or the national server is queried at that
version. The two must agree (§12.2).

---

## 7. Stages and gates

| Stage | Objective | Deliverables | Exit gate |
|---|---|---|---|
| **S0 Frame** | scope, users, licences, service-level objectives | `docs/scope.md`, `docs/slo.md`, licence review | licence check passes for every system to be served |
| **S1 Identity** | canonical URIs, versions, extension definitions | `docs/uris.md`; StructureDefinitions for the extensions | reviewed; consistent with AU Base where it defines URIs |
| **S2 SNOMED engine** | load or front SNOMED CT-AU | running engine at the pinned edition | smoke tests; version URIs resolve |
| **S3 Resources** | generate CodeSystems, ValueSets, ConceptMaps | generator; loaded resources | FHIR validator passes on every resource |
| **S4 Façade** | routing, versioning, tiers, auth | service; tests | unit and integration tests pass |
| **S5 Matcher** | `/match` | index, reranker, calibration report | precision@1 ≥ 0.95 on clinical scopes (§11.6) |
| **S6 Conformance and differential** | standards and correctness | test reports | ecosystem tests pass for implemented operations; differential 100% matched or explained |
| **S7 Performance and security** | service levels and gating | load-test and security-test reports | SLOs met; unlicensed client denied |
| **S8 Operate** | releases and monitoring | runbooks; release pipeline | ongoing |

---

## 8. Code systems: identity, versions, content

### 8.1 Canonical URIs
- **Use published URIs wherever they exist:** SNOMED CT, LOINC, UCUM and RxNorm as in §3, and HL7 AU or NCTS URIs
  for Australian systems where they're defined (check the current AU Base IG for PBS, MBS and others).
- **Otherwise mint a stable URI** under a domain you control, for example
  `https://terminology.arepo-tech.ai/CodeSystem/<name>`. Record it in `docs/uris.md`. Never change a URI once
  published.

### 8.2 Versions
- **SNOMED CT-AU:** `http://snomed.info/sct/32506021000036107/version/<yyyymmdd>`, from the T0 manifest.
- **Other systems:** the source's own version string (for example LOINC `2.83`), or the release date for schedules
  (PBS and MBS monthly).
- **Default version:** the pinned one. Clients may request a specific supported version.
- Keep at least the **current and previous** versions of each system loaded, so clients can transition.

### 8.3 Generated CodeSystems (non-SNOMED)
From the T0 nodes and their properties, generate CodeSystem resources with:
- `content = complete`;
- concepts with code, display and designations;
- `status` properties (active, retired) and `replacedBy` properties where the source provides them;
- the hierarchy (`subsumedBy`) where the source defines one.

Validate every resource with the FHIR validator before loading.

### 8.4 Designations and language
Serve `en-AU` designations where available (from the SNOMED CT-AU language reference set), with `displayLanguage`
support. The preferred term is the Australian preferred term.

---

## 9. Value sets and expansion

### 9.1 Kinds of value set
- **Intensional (SNOMED):** implicit ECL value sets (`?fhir_vs=ecl/…`) or ValueSet resources with an ECL filter.
- **Extensional:** explicit code lists, versioned.
- **Graph-derived:** value sets computed from the T0 release (for example "all AMT medicinal products listed on the
  PBS"). Generate them as extensional value sets, and record the generating query and the release id in the
  resource's metadata.

### 9.2 `$expand` behaviour
- Support `filter`, `count` / `offset` (paging), `activeOnly`, `includeDesignations`, `displayLanguage`, and
  `system-version` pinning.
- **Large expansions** are always paged. Set a maximum page size (for example 1,000) and a total-count limit, with a
  clear error beyond it.
- **Expansion identity:** `expansion.identifier` plus the parameters used, including code-system versions. Compute a
  **SHA-256 over the sorted codes** and return it in an extension, so clients and tests can compare expansions.

### 9.3 Caching
Cache expansions keyed by (value set URL, version, parameters, code-system versions). Invalidate on release. Never
serve a cached expansion across a code-system version change.

---

## 10. Concept maps and translation

### 10.1 Generating ConceptMaps from SSSOM
From the T0 SSSOM files, generate one ConceptMap per (source system, target system, mapping family):
- **`…-clinical`**: rows from routes graded Tier 1, Tier 2, or native (verified), from `go_mappings.ids.sssom.tsv`.
- **`…-candidate`**: rows from `go_mappings.candidates.sssom.tsv` (ungraded and inadmissible routes). Flagged, and
  restricted (rule T3).

Each ConceptMap records the T0 release, generation time and generator version.

### 10.2 Equivalence semantics (FHIR R4)
R4 `ConceptMap.group.element.target.equivalence` codes: `relatedto`, `equivalent`, `equal`, `wider`, `subsumes`,
`narrower`, `specializes`, `inexact`, `unmatched`, `disjoint`. Map from the SSSOM predicate:

| SSSOM `predicate_id` | R4 `equivalence` | Note |
|---|---|---|
| `skos:exactMatch` | `equivalent` | `equal` only when the concepts are identical by definition (the same code system and code) |
| `skos:closeMatch` | `inexact` | a map to a standard concept that may be broader |
| `skos:broadMatch` | `wider` | the target is broader than the source |
| `skos:narrowMatch` | `narrower` | the target is narrower than the source |
| `skos:relatedMatch` | `relatedto` | |

**Never** serve `equivalent` for a pair whose T0 equivalence group is flagged `has_conflict`: downgrade it to
`inexact`, with a comment. (If a later version moves to FHIR R5, map to its `relationship` codes instead.)

### 10.3 Tier and provenance extensions
Define StructureDefinitions for these extensions on `ConceptMap.group.element.target`, with canonical URLs under your
domain:

| Extension | Value |
|---|---|
| `go-tier` | `1`, `2`, `native`, `ungraded`, `inadmissible` |
| `go-route` | the T0 route id |
| `go-wilson-lower` | decimal (graded routes only) |
| `go-source` | the source of the mapping (edge `source`) |
| `go-source-locator` | a URL, file or schedule reference |
| `go-pin` | the source release the mapping is true under |
| `go-justification` | SSSOM `mapping_justification` |

### 10.4 `$translate` behaviour
- **Inputs:** `system` and `code` (or a `coding`), `target` (a system or value set), optionally
  `conceptMap`, `reverse` and `include-candidates` (restricted).
- **Output:** `result` true or false; `match` entries with `equivalence`, `concept`, the tier extensions, and the
  source ConceptMap.
- **Order:** by equivalence strength (`equivalent` > `inexact`/`narrower`/`wider` > `relatedto`), then tier, then
  Wilson lower bound.
- **Reverse translation** uses the same maps inverted, with `wider` ↔ `narrower` swapped. It never infers
  equivalence the forward map didn't state.
- **No chaining** across maps at request time. Chains are computed upstream (T0) and served as their own mapping
  family, with their own tier.

---

## 11. The free-text matcher

### 11.1 Contract
`POST /match` takes `{text, scope: {systems, semantic_tags?, ecl?, value_set?}, k (default 5), mode: clinical|analyst}`.
It returns `{candidates: [{system, code, display, matched_term, term_type, score}], abstain: bool, reason?}`.
The matcher **never** returns a code outside its index or outside the scope (rule T5).

### 11.2 Index
Built from T0 `synonym`, **locally only** (licensed text):
- preferred terms;
- synonyms, and fully specified names without the semantic tag;
- LOINC long common names, short names and display names;
- other vocabularies' names;
- normalised forms (NFKD, accents stripped, lower-case, punctuation collapsed).

Rebuilt on every release.

### 11.3 Pipeline
1. **Normalise** the text, and expand abbreviations from a **curated list** (never guessed).
2. **Candidates:** lexical BM25 (OpenSearch, or an embedded Lucene or Tantivy index), plus, optionally, embedding
   nearest neighbours from a biomedical entity-linking model (SapBERT-style, trained on UMLS synonymy; check the
   model licence). Take the union of the top 50 from each.
3. **Scope filter:** by system, semantic tag, and ECL or value-set membership (checked against the SNOMED engine or
   HAPI).
4. **Rerank:** a cross-encoder, or a language model **constrained to the candidate list** that must choose or
   abstain.
5. **Score and abstain:** a calibrated score, and `abstain = true` below the threshold for the requested mode.

### 11.4 Known pitfalls
- **Near-synonyms are the main error source.** Prefer abstaining to a near match, especially across semantic tags
  (finding against disorder, substance against product).
- **Short phrases and abbreviations** are ambiguous. Require a scope, and abstain without one.
- **Laterality and specificity:** "hip pain" must not match "left hip pain", and vice versa, unless the scope allows
  broader or narrower matches, flagged.
- **Retired concepts** are excluded from the index unless the mode requests them.

### 11.5 Gold set
About **600 phrase → code pairs** across conditions, medicines (AMT levels), pathology (LOINC) and procedures, from:
- hand-checked bindings with verdicts;
- phrases experts write in the way users actually type (abbreviations, misspellings, brand names);
- an **abstention set** of about 100 phrases with no correct code in scope.

Split 70/30 (development/test), stratified by domain. The test split is used once per matcher version.

### 11.6 Evaluation and calibration
- **accuracy@1 and accuracy@5** per domain and scope, with Wilson intervals;
- **calibration:** choose the abstain threshold per mode where **precision@1 ≥ 0.95** (clinical); report the coverage
  that results (the share not abstained);
- **abstention precision:** of abstentions on the abstention set, the share correct;
- read and classify every test error (near-synonym, wrong tag, abbreviation, spelling, missing synonym). Feed missing
  synonyms back as review items.

---

## 12. Testing and validation

### 12.1 Conformance: the HL7 terminology ecosystem tests
The HL7 FHIR Terminology Ecosystem IG publishes test cases (package `hl7.fhir.uv.tx-ecosystem`). Its runner is built
into the FHIR validator:

```bash
java -jar validator_cli.jar txTests -tx <server-url> -test-version <version> -output <folder>
```

The test cases are written in R5, and the runner converts to and from R4 for R4 servers. Run the suites for the
operations you implement. Record passes, failures, and **justified exclusions** (tests for features out of scope).
Gate: every in-scope test passes.

### 12.2 Differential testing against a reference server
For about **200 expressions and value sets** (ECL across hierarchies and refinements; refset members; is-a;
graph-derived value sets), compare this server with a reference at the **same edition**: the national service, or
the HL7 Australia terminology server, which is authoritative for the Australian edition in the HL7 ecosystem.
- Compare counts, and the SHA-256 of the sorted codes.
- **Gate: 100% match, or a documented explanation** (for example a known difference in handling inactive concepts).

### 12.3 Property tests
- **Subsumption** is reflexive and transitive over sampled triples.
- `$validate-code` on every code of an extensional value set returns true; on random non-members it returns false.
- `$translate` forward, then reverse, returns the original code among its matches for `equivalent` maps.
- No clinical ConceptMap contains a target from a candidate route.

### 12.4 Regression
- **Expansion hashes** for a fixed set of 50 value sets are stored per release. An unexpected change fails the build.
  Expected changes (a new edition) are reviewed and accepted.
- The **T0 quiz items** tagged for translation and matching must pass.
- **Mapping audit:** 80 random `$translate` results from clinical maps are read by a terminologist per Appendix A.
  Gate: Wilson lower bound ≥ 0.80 (≥ 72/80). Every error becomes a NeverShow item and a report to the T0 producer.

### 12.5 Resource validation
Every generated CodeSystem, ValueSet and ConceptMap passes the FHIR validator (R4), including the custom extension
definitions.

---

## 13. Performance and scaling

### 13.1 Service-level objectives (defaults; set in `docs/slo.md`)

| Operation | 95th-percentile latency | Notes |
|---|---|---|
| `$lookup`, `$validate-code` | ≤ 50 ms | cached |
| `$subsumes` | ≤ 100 ms | |
| `$translate` | ≤ 100 ms | |
| `$expand` (paged, simple) | ≤ 300 ms per page | complex ECL may be slower; cache |
| `/match` | ≤ 300 ms (lexical), ≤ 800 ms (with rerank) | |
| Availability | 99.5% monthly | |

### 13.2 Load testing
Use k6 or Locust with a realistic mix: ETL batch validation bursts, interactive matching, and expansions. Test at 2×
the expected peak. Measure latency percentiles, error rates, and resource use.

### 13.3 Scaling levers
- The SNOMED engine: memory for Elasticsearch, and replicas.
- HAPI: connection pools, and read replicas for Postgres.
- The façade: stateless horizontal scaling.
- Caching: expansions, and `$lookup` results, keyed by version.
- Batch endpoints for ETL (FHIR `Bundle` of operations) to cut round-trips.

---

## 14. Security and licensing

### 14.1 Authentication and authorisation
- **Systems:** SMART Backend Services (OAuth2 client credentials with a signed JWT). **People:** OIDC.
- **Scopes per code system and operation:** for example `system/CodeSystem.read`, plus a per-system licence claim.
- **Candidate maps and the analyst mode** need an explicit role.

### 14.2 Licence gating (rule T6)
- **SNOMED CT-AU and AMT:** serve content only to clients whose organisation holds, or is covered by, the
  appropriate licence (the Australian national licence through NCTS registration). Record the basis per client.
  Confirm the terms for serving through a hosted API with the licensor.
- **LOINC:** follow its licence, including the required notice and attribution.
- **UMLS-derived content:** **not served** unless the licence permits redistribution to the specific users.
  Mappings derived through UMLS may need to be withheld from external clients.
- **Other sources:** per the T0 licence matrix. Publish the **attribution** required by each source in
  TerminologyCapabilities and on an attribution page.
- **Test:** an unlicensed test client is denied SNOMED content and gets a clear error.

### 14.3 Other controls
- TLS everywhere.
- Rate limits per client.
- Audit logs of client, operation, system and version. **No PHI**: the server needs none. Free text sent to `/match`
  may contain patient details, so screen it, keep only aggregate logs, and never log the raw text.

---

## 15. Release management

### 15.1 Cadence
- **SNOMED CT-AU and AMT:** monthly (follow NCTS releases).
- **PBS and MBS:** monthly.
- **LOINC:** twice a year.
- **Graph-Ontology mappings:** with each T0 release.

### 15.2 Release procedure
1. Pin the new T0 release, including the new source versions.
2. Load the new SNOMED edition into the engine, **alongside** the previous one.
3. Regenerate CodeSystems, ValueSets and ConceptMaps; validate them.
4. Rebuild the matcher index; re-run the matcher test split.
5. Run conformance, differential, property and regression tests.
6. Produce the **edition diff:** codes added, retired and remapped; value sets whose expansion changed; ConceptMap rows
   added, removed or re-tiered.
7. Canary (a small share of traffic or selected clients), then promote. Keep the previous version queryable for at
   least one cycle.
8. Publish release notes with the diff, and the notices for deprecated codes.

### 15.3 Deprecation and retirement
- Retired codes stay resolvable: `$lookup` returns `inactive`, and `$validate-code` returns false with a message and
  the replacement (from SNOMED historical associations, or the source's own replacement data).
- ConceptMaps from retired codes are kept, and flagged.

---

## 16. Scorecard

| # | Metric | Gate |
|---|---|---|
| C1 | HL7 ecosystem tests, in-scope operations | **all pass** (exclusions justified) |
| C2 | differential expansions vs reference | **100%** matched or explained |
| C3 | FHIR validation of generated resources | **100%** pass |
| C4 | tier and provenance on `$translate` targets | **100%** |
| C5 | clinical ConceptMaps free of candidate routes | **100%** |
| C6 | mapping audit (80 samples) | Wilson lower bound ≥ **0.80** |
| C7 | matcher precision@1, clinical scopes | ≥ **0.95** at the calibrated threshold; coverage reported |
| C8 | matcher abstention precision | ≥ 0.90 |
| C9 | SLOs | met at 2× peak |
| C10 | licence gating | unlicensed client denied; attribution published |
| C11 | regression hashes | no unexpected change |

---

## 17. Operations

### 17.1 Deployment
Containers: the façade, HAPI with Postgres, the SNOMED engine with Elasticsearch (or the national server
configuration), and the matcher. Hosted in an **Australian region**, behind an API gateway. Infrastructure as code;
backups of Postgres and of the release registry.

### 17.2 Monitoring
- **Health:** latency percentiles per operation, error rate, cache hit rate, and engine health.
- **Quality:** the matcher's abstain rate per scope, the top unmatched phrases (aggregated, no raw text beyond
  retention rules), and `$translate` no-match rates per system pair.
- **Alerts:** SLO breaches, error spikes, and differential-test failures in the nightly job.

### 17.3 Nightly jobs
- a differential sample (20 expressions) against the reference;
- regression hashes;
- a smoke run of the quiz items.

### 17.4 Feedback loop
- Unmatched-phrase clusters and mapping-audit errors become **review items for terminologists**. Confirmed fixes go to
  the T0 producer as mapping requests.
- New synonyms enter the index only through a T0 release, never by hot-patching.

### 17.5 Incidents
- **Severity 1:** a wrong clinical mapping served as `equivalent`, or licence gating bypassed. Disable the affected
  ConceptMap or client immediately, investigate, and add a regression test.
- **Severity 2:** an SLO breach or engine outage. Fail with clear errors; never serve stale versions silently.

---

## 18. Risk and failure-mode register

| # | Risk | Control | Detected by |
|---|---|---|---|
| X1 | expansions silently change between editions | version pinning; expansion hashes | C11 |
| X2 | candidate mappings leak into clinical use | separate ConceptMaps; authorisation | C5; tests |
| X3 | a conflicting equivalence served as `equivalent` | conflict downgrade rule (§10.2) | property tests |
| X4 | the matcher is confidently wrong on near-synonyms | abstain threshold; scope filters; curated abbreviations | C7; audit |
| X5 | licence breach through open endpoints | per-system gating; attribution | C10 |
| X6 | ECL behaviour differs from the specification | a proven engine; differential tests | C2 |
| X7 | retired codes treated as current | `$validate-code` inactive handling | property tests |
| X8 | URI churn breaks clients | never change published URIs | review |
| X9 | performance collapse on large expansions | paging; caching; limits | load tests |
| X10 | PHI in matcher logs | text screening; aggregate logging | log review |

---

## 19. Roles and effort

| Role | Responsibilities |
|---|---|
| Pipeline lead | scope, gates, releases |
| Engineer | façade, generators, matcher, operations |
| Terminologist | URIs, equivalence semantics, mapping audit, matcher gold set, review items |
| Security and licensing owner | gating, attribution, licence confirmation |

**Indicative effort** (one engineer, terminologist part-time):

| Stage | Effort |
|---|---|
| S1 | 3 days |
| S2 | 1 week |
| S3 | 1–2 weeks |
| S4 | 1–2 weeks |
| S5 | 2 weeks, including the gold set |
| S6 | 1 week |
| S7 | 1 week |

---

## 20. Prompt library

**20.1 ConceptMap generator review.**
> Review the SSSOM → ConceptMap generator against GO-TS `PLAYBOOK.md` §10: clinical and candidate separation, the
> equivalence mapping table, the conflict downgrade rule, the tier and provenance extensions, and FHIR R4 validity.
> List any path by which a candidate row could reach a clinical map.

**20.2 ECL differential set.**
> Write 200 ECL expressions covering SNOMED CT-AU hierarchies, AMT product levels, refinements, reverse attributes,
> refsets and dotted attributes, with the ECL version each needs. For each, give the purpose, and the expected
> behaviour for inactive concepts.

**20.3 Matcher error analysis.**
> Classify these matcher errors into near-synonym, wrong semantic tag, abbreviation, spelling, missing synonym, and
> scope. For each class, propose one index, normalisation or threshold change, and the test that proves it.

**20.4 Red team.**
> Attack this server against risks X1–X10 and rules T1–T9. For each, say whether it is controlled, where, and by
> which metric, and propose fixes for gaps.

---

## 21. Decisions to make

| # | Decision | Recommended default |
|---|---|---|
| SD1 | SNOMED engine | the national (Ontoserver-based) service where licensed; self-hosted Snowstorm for offline or controlled use |
| SD2 | FHIR store for other resources | HAPI FHIR JPA with Postgres |
| SD3 | ECL version supported | whatever the chosen engine implements, declared; at least 2.2 |
| SD4 | whether UMLS-derived mappings are served externally | no, unless the licence permits it for the specific users |
| SD5 | matcher embedding model | lexical first; add SapBERT-style embeddings only if C7 fails on lexical alone |
| SD6 | ICD-10-AM | include only with an IHACPA licence, if Australian hospital coding is in scope |
| SD7 | SLO targets | the §13.1 defaults, adjusted to clients' needs |

---

## 22. References

**Checked when this playbook was written (24 Sep 2026):**
- HL7, *FHIR Terminology Ecosystem IG*: server requirements, the test-case registry, and the test runner in the FHIR
  validator (`txTests`); R5 test cases converted for R4 servers; the HL7 Australia server authoritative for the AU
  edition ([IG](https://build.fhir.org/ig/HL7/fhir-tx-ecosystem-ig/); [requirements](http://hl7.org/fhir/uv/tx-ecosystem/requirements.html); [GitHub](https://github.com/HL7/fhir-tx-ecosystem-ig))
- SNOMED International, *Expression Constraint Language: Specification and Guide*: version 2.2 (November 2023), and
  version 2.3 (March 2026) ([docs.snomed.org](https://docs.snomed.org/snomed-ct-specifications/snomed-ct-expression-constraint-language))
- HL7 Australia, AU Core FHIR IG v2.0.0 ([hl7.org.au](https://hl7.org.au/fhir/core/))
- [IHTSDO/snowstorm](https://github.com/IHTSDO/snowstorm); [hapifhir/hapi-fhir-jpaserver-starter](https://github.com/hapifhir/hapi-fhir-jpaserver-starter); [SSSOM](https://github.com/mapping-commons/sssom)

**From the author's knowledge; not re-checked when written:**
- HL7 FHIR R4 terminology module: CodeSystem, ValueSet, ConceptMap (the `ConceptMapEquivalence` codes),
  TerminologyCapabilities, and the operations listed in §3.
- "Using SNOMED CT with FHIR" (HL7): system and version URIs, and implicit value sets.
- SNOMED International, *Terminology Services Guide*.
- The Australian Digital Health Agency's National Clinical Terminology Service (NCTS); the SNOMED CT-AU and AMT
  release cadence.
- HL7 SMART App Launch: Backend Services.
- Liu et al., SapBERT, NAACL 2021; Mohan & Li, MedMentions, AKBC 2019 (entity-linking evaluation practice).
- LOINC licence terms and notice requirements; the UMLS licence's redistribution restrictions.

---

## Appendix A: Hand-check protocol

**A.1 Sampling.** A simple random sample with a recorded seed. Stratify by system pair or domain when they differ,
and weight back. Targeted reads never count toward a grade.

**A.2 Sample size and grades.** The grade is the 95% Wilson lower bound:
lower bound = (p + z²/2n − z·√(p(1−p)/n + z²/4n²)) / (1 + z²/n), with z = 1.96.

| Target | Need |
|---|---|
| any grade | n ≥ 30 |
| lower bound ≥ 0.80 | at n = 80, **at least 72 correct** (71 fails) |
| lower bound ≥ 0.90 | at n = 80, at least 78 correct |
| lower bound ≥ 0.99 | at least 381 read with zero errors |

Fix n before reading. No optional stopping.

**A.3 Reading rules.**
- Judge each mapping against the **authoritative definitions** of both codes (the source terminology's own content),
  not from memory.
- Blind to tier and scores.
- Verdicts: `correct`, `wrong`, `ambiguous` (counts as wrong), with a reason for every wrong or ambiguous.
- A second reader covers 20%; κ ≥ 0.8 is required; a third person adjudicates.
- Models may pre-read but never give the verdict of record.

**A.4 Evidence file.**
- sample id, version, seed, population, number drawn;
- per row: source code, target code, equivalence, verdict, reason, readers, adjudication, date;
- κ, number correct, Wilson lower bound, outcome.

---

## Appendix B: Templates

**B.1 `docs/uris.md`:** per code system: canonical URI, source authority, version format, whether the URI is
published by an external authority or minted here, and date adopted.

**B.2 Extension StructureDefinition checklist:** canonical URL; context (`ConceptMap.group.element.target`); value
type; cardinality; description; examples; validation test.

**B.3 Release notes:**
- the T0 release;
- code-system versions;
- the edition diff summary;
- value sets whose expansion changed (with their hashes);
- ConceptMap changes by tier;
- matcher metrics;
- conformance and differential results;
- deprecations;
- known issues.

**B.4 Client onboarding:**
- organisation;
- licence basis per code system;
- scopes granted;
- rate limits;
- contact;
- date;
- reviewer.

**B.5 Release checklist:**
- C1–C11 passed;
- canary plan written;
- the previous version still available;
- release notes published;
- attribution page updated.
