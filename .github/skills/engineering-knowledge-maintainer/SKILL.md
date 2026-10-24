---
name: engineering-knowledge-maintainer
description: "Use when adding, researching, organizing, explaining, or improving engineering knowledge in this repository, including rough notes, code, papers, links, experiments, and requests to audit or navigate existing material."
---

# Engineering Knowledge Maintainer

Maintain this repository as a connected engineering knowledge base across all its domains. Use the existing indexes and files as the source of truth; do not assume that a planned note, map, or glossary entry already exists.

## When to use

- The user provides technical content, rough notes, code, research papers, links, or experiment results to incorporate.
- The user asks a technical question that should be answered or recorded using this repository's knowledge.
- The user asks where information belongs, how to find related material, or how to follow a learning path.
- The user asks to reorganize, improve, audit, or cross-link existing material.

## Add new knowledge

1. Understand the subject, audience, and intended depth. Preserve useful user insights.
2. Inspect the root [README](../../../README.md), relevant domain README files, and existing notes. Check `MAP.md` or `GLOSSARY.md` only if present; check `shared/glossary/` when relevant.
3. Search for existing canonical notes and overlapping material before creating files. Extend or link to an existing canonical explanation instead of creating a duplicate.
4. Choose the domain and a focused destination based on the repository's actual structure. Ask only if an essential choice cannot be inferred.
5. Separate established facts, claims from cited sources, and the author's interpretations.
6. Write a clear, original explanation at the appropriate depth. Add examples, diagrams, citations, and tests where useful; do not fabricate details, sources, benchmark results, or test results.
7. Link to relevant prerequisites, canonical concepts, implementations, experiments, and further reading using relative Markdown links.
8. Update affected domain indexes and root navigation. Update `MAP.md` or `GLOSSARY.md` only if those files exist or the user asks to create them; otherwise do not claim they were updated.
9. Validate local links affected by the change and run relevant tests for code examples when practical. Report what changed, what was checked, and any limitations.

## Explain repository navigation

1. Consult the actual root and domain indexes and verify any note before linking to it.
2. Explain the recommended domain and navigation path, with prerequisites and related concepts that exist.
3. Suggest a learning sequence when it helps. Do not guess paths or present planned topics as existing notes.

## Improve existing notes

1. Read the target note and relevant references before editing.
2. Preserve accurate information and useful original explanations; make focused improvements to clarity, organization, examples, diagrams, and technical precision.
3. Update indexes and cross-references when titles, locations, or conceptual scope change.
4. Fix broken links in the affected area, and avoid unrelated rewrites.

## Research and source handling

- Inspect supplied sources with available tools when possible; prefer primary sources, official documentation, and original papers.
- Record source title, URL, and publication or access date where useful. Distinguish source findings from interpretation.
- If a source is inaccessible, state the limitation and use only information actually available.
- Respect copyright: summarize in original language rather than copying substantial passages.

## Repository review and maintenance

For audits or maintenance requests, inspect the relevant files and report confirmed issues separately from suggestions. Check for broken links, duplicate concepts, orphaned notes, missing cross-references, stale claims, code examples, and whether existing indexes reflect actual files. Do not delete material simply because it appears redundant.

When substantive content is added, changed, renamed, or removed, update the relevant domain index and root navigation as needed. Maintain a single canonical explanation when practical, and update map or glossary resources only when they exist or are explicitly requested.

## Safe operating rules

- Inspect before editing; never overwrite unrelated files.
- Prefer minimal, focused changes and preserve user-authored insights.
- Use relative links and validate them; never invent paths or claim validation that was not performed.
- Never publish, push, or create releases without explicit authorization.
- Proceed without unnecessary questions when the request is clear.
- Report created and changed files, checks actually run, and unresolved issues.

## Example requests

- “Turn these rough notes about KV caches into a clear note and connect it to related inference topics.”
- “Where should I put notes about database indexes, and what should I read first?”
- “Summarize this paper in my own words and link it to the existing concepts.”
- “Audit this domain for broken links and notes that are not connected to its index.”
