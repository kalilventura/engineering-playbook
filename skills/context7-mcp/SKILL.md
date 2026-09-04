---
name: context7-mcp
description: Use when answering questions about a library, framework, SDK, or API, or generating code that uses one - fetches current documentation via Context7 instead of relying on training data
---

# Context7 Documentation Lookup

## Overview

Library APIs change faster than training data. Before answering setup/config questions, writing code against a library, or citing an API reference, fetch current documentation via Context7's `resolve-library-id` + `query-docs` instead of relying on memory.

## When to Use

- Setup or configuration questions ("How do I configure Next.js middleware?").
- Writing code that calls a specific library ("Write a Prisma query for...").
- API reference questions ("What are the Supabase auth methods?").
- Any mention of a specific framework/library (React, Vue, Next.js, Prisma, Supabase, Express, Tailwind, etc.).

## Workflow

1. **Resolve the library ID** — call `resolve-library-id` with the library name and what to look up (improves relevance ranking).
2. **Select the best match** — prefer exact name match, higher benchmark score, official/primary package over community forks, and a version-specific ID if the user named a version (e.g. "React 19").
3. **Fetch the docs** — call `query-docs` with the selected library ID and a query scoped to a single concept. If the question spans multiple distinct concepts (routing, auth, caching), make one `query-docs` call per concept with the same library ID — combined queries dilute ranking and return shallow results, unless the question is specifically about how the concepts interact.
4. **Answer from the fetched docs** — use the current, accurate information and code examples returned, citing the library version when relevant.

## Quick Reference

| Situation | Do |
|---|---|
| User names a library/framework | Resolve its ID before answering, even if you think you already know the answer |
| Question spans multiple concepts | One `query-docs` call per concept, same library ID |
| User names a version | Prefer the version-specific library ID |
| Multiple libraries match the name | Prefer the official/primary package over a community fork |
| Refactoring, business logic, or general programming | Don't use this skill — Context7 is for library docs, not code review |

## Common Mistakes

- Answering a library-specific question from memory instead of fetching current docs.
- One `query-docs` call bundling unrelated concepts, diluting the results for all of them.
- Ignoring a version the user mentioned and resolving to the latest library ID anyway.
- Using this skill for refactoring, business logic debugging, or general programming questions it isn't meant for.
