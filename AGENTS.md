# Codex Instructions

## Context Efficiency
- Do not scan the entire repository unless necessary.
- Search for relevant files, symbols, and references first.
- Read the minimum file ranges required to complete the task.
- Do not reread unchanged files.
- Summarize large command outputs rather than reproducing them.
- Prefer targeted tests during iteration.
- Keep explanations concise unless detailed reasoning is requested.
- Avoid redundant comments and documentation.
- Reuse existing abstractions before creating new ones.

## Code Quality
- Prefer simple, readable implementations over clever abstractions.
- Keep functions and components focused.
- Avoid unnecessary dependencies.
- Do not duplicate logic that already exists.
- Remove dead code encountered during relevant edits when safe.

## SQL / Database
- Optimize for PostgreSQL.
- Avoid SELECT * in application queries.
- Check indexes for filtering, joins, ordering, and frequently searched columns.
- Avoid N+1 queries.
- Prefer set-based operations and batching.
- Use EXPLAIN / EXPLAIN ANALYZE when investigating slow queries.
- Do not denormalize without a measured reason.
- Preserve source provenance and evidence relationships.
- Keep migrations reversible when practical.
- Do not change schemas without considering existing data.

## Frontend / UI
- Reuse existing components before creating new ones.
- Avoid unnecessary state, effects, and rerenders.
- Keep data fetching narrow.
- Paginate or virtualize large datasets.
- Lazy-load expensive secondary UI when appropriate.
- Maintain responsive layouts.
- Maintain keyboard accessibility and semantic HTML.
- Keep technical/data-heavy interfaces information-dense but readable.
- Prefer clarity and usability over decorative effects.

## Performance
- Measure before optimizing.
- Identify whether bottlenecks are database, CPU, network, rendering, memory, or I/O.
- Fix high-impact bottlenecks before micro-optimizations.
- Avoid unnecessary API/database round trips.
- Cache only where there is a clear benefit and correct invalidation strategy.

## Workflow
Before making a significant change:
1. Identify the smallest relevant set of files.
2. Inspect existing architecture and conventions.
3. Make the smallest coherent change.
4. Run targeted validation/tests.
5. Report what changed and any remaining concerns concisely.

## Deployment

This repository is connected to Netlify and deploys automatically from the GitHub `main` branch.

Production URL:
`https://acop-database.netlify.app`

The production entry point is:
`index.html`

Deployment flow:

1. Modify the existing project files in place.
2. Validate the application.
3. Commit changes to Git.
4. Push to `main`.
5. Netlify automatically redeploys the same production URL.

Do not:

- rename `index.html`
- create alternate production HTML files
- create versioned deployment copies
- change the Netlify deployment structure unless explicitly requested
- assume a new URL should be created for a new version

The production URL should remain stable across updates.
