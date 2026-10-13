# Project Wiki Schema

This schema governs the AI-maintained knowledge in `Projects/`. It makes project memory easier to retrieve, verify, and maintain without forcing a directory migration.

## Scope and ownership

- Keep project knowledge in `Projects/<project>/`.
- Preserve existing human-authored structure and notes. Adopt this schema incrementally when a note is created or materially revised; do not reformat notes solely for compliance.
- Use the project `Updates.md` as the append-only record of AI-initiated content changes.

## Note types

Use a `type` frontmatter field for new canonical notes when it adds clarity:

- `index` — entry point that links to the most useful canonical notes.
- `architecture` — system boundaries, components, integrations, and data or control flow.
- `decision` — a durable choice, its context, alternatives, and consequences.
- `workflow` — reproducible operational or development procedure.
- `reference` — stable API, configuration, repository, or domain reference.
- `investigation` — verified diagnosis or finding; label unresolved conclusions clearly.

Use a focused title and one topic per note. Do not create a note merely to fill a type.

## Canonical note shape

For a new or substantially reworked canonical note, use this frontmatter as applicable:

```yaml
---
type: architecture
aliases: []
tags: []
status: active
last_verified: YYYY-MM-DD
evidence: []
---
```

- `aliases` contains alternate names, acronyms, or spellings useful for retrieval.
- `tags` are sparse and stable; do not duplicate the folder hierarchy in tags.
- `status` is `active`, `superseded`, or `uncertain`.
- `last_verified` records the most recent date evidence was checked.
- `evidence` identifies the basis for non-obvious claims: repository paths, commit hashes, command results, URLs, or an explicit user decision.

Use informative headings and dense factual writing. Prefer full noun phrases over vague pronouns when describing entities or relationships.

## Links and evidence

- Add `[[wikilinks]]` only for useful relationships between durable concepts, notes, decisions, or workflows.
- Decisions and investigations must identify their evidence in frontmatter or an `## Evidence` section.
- Preserve uncertainty and disagreement. Do not convert an inference into a fact, and do not overwrite a conflicting claim without recording why newer evidence is stronger.
- Treat code, command output, and explicit user instructions as evidence; do not copy credentials, tokens, or private data into the wiki.

## Maintenance

When relevant during substantive work, repair clearly broken links, merge duplicate claims into the stronger canonical note, and mark stale information as `superseded` rather than deleting its history. Avoid broad cleanup unrelated to the current task.
