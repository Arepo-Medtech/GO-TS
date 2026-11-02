# GO-TS: Graph-Ontology → terminology server

Track **T4** of the Graph-Ontology next phases. A FHIR R4 terminology service for Australian clinical and medicines vocabularies: versioned lookup, validation, subsumption, expansion (SNOMED ECL) and translation, plus a free-text matcher, with tier and source on every mapping.

## Start here
This repository is self-contained. Two documents hold everything this track needs:

1. **[PLAYBOOK.md](PLAYBOOK.md)**: the authoritative playbook for this track. It covers mission, rules, architecture,
   stages and gates, engineering guidance, evaluation, scorecard, operations, governance, risks, prompts and
   references, with the hand-check protocol and templates as appendices.
2. **[T0.md](T0.md)**: the shared foundations this track consumes (frozen snapshots, the licence matrix, predicate
   properties, the validation register, shared views, the SSSOM export and the quiz suite). It is identical across
   the four Graph-Ontology track repositories.

## How this track uses T0
- generates FHIR ConceptMaps from the SSSOM export: Tier 1/2 and native routes into clinical maps, ungraded and inadmissible into candidate maps;
- builds its `/match` index from `synonym`, and translation equivalence from `equivalence_group`;
- uses the predicate properties for ConceptMap equivalence semantics;
- is tested on the quiz suite's translation and matching items;
- needs `serve_codes_to_clients` and `display_to_end_users` in the licence matrix, with attribution published.

## Pinning a release
`t0.lock` pins Release 2, `GO-20261103-4e9f3709` (3 Nov 2026), the first full T0 release: every gate passed (docs/t0.md in Graph-Ontology). Upgrade by bumping `t0.lock` in a pull request that carries the release diff and this track's regression results (T0 §9).

Status: plan only. Nothing here has been built or run.
