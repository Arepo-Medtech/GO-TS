# GO-TS: Graph-Ontology → terminology server

Track **T4** of the Graph-Ontology next phases. A FHIR R4 terminology service for Australian clinical and medicines vocabularies: versioned lookup, validation, subsumption, expansion (SNOMED ECL) and translation, plus a free-text matcher, with tier and source on every mapping.

## Start here
1. **[T0.md](T0.md)**: the shared foundations, identical in all four track repos (GO-Harness, GO-HGT, GO-PJI, GO-TS):
   frozen snapshots, the licence matrix, algebraic properties on predicates, the validation register, shared views,
   the SSSOM export and the quiz suite. This track consumes a T0 release; it never reads the live graph.
2. The full recipe for this track is in `COMPENDIUM.md` (track T4), in
   [Arepo-Medtech/graph-ontology-compendium](https://github.com/Arepo-Medtech/graph-ontology-compendium).
3. Method (rules R1–R17, validation layers L0–L6) is governed by `PLAYBOOK.md` in
   [Arepo-Medtech/one-shot](https://github.com/Arepo-Medtech/one-shot).

## How this track uses T0
- generates FHIR ConceptMaps from the SSSOM export: Tier 1/2 and native routes into clinical maps, ungraded and inadmissible into candidate maps;
- builds its `/match` index from `synonym`, and translation equivalence from `equivalence_group`;
- uses the predicate properties for ConceptMap equivalence semantics;
- is tested on the quiz suite's translation and matching items;
- needs `serve_codes_to_clients` and `display_to_end_users` in the licence matrix, with attribution published.

## Pinning a release
Copy `t0.lock.example` to `t0.lock` once the first T0 release exists, and fill in its id and fingerprint (T0 §9).

Status: plan only. Nothing here has been built or run.
