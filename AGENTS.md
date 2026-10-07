# AGENTS.md — @linked.cm/access-templates

Staged in the **linked-cm** org (npm scope `@linked.cm`) pending René's review; moves to linked-fw (`@_linked`) only after approval. The first npm publish is approved manually by René; until then the repo has no `publish.yml`, so merging to `main` cannot release it (same as `linked-cm/safety`).

Canonical working copy: `/Volumes/LIGHT/linked-cm/main/access-templates`. Build and tests follow `@linked.cm/access`: `npm run build` (`linked build`), `npm test` (jest + ts-jest, ESM).

Network role templates for `@linked.cm/access`: role templates as shapes, per-organization roles based on them, adjustments that survive template upgrades, materialized into ordinary grants. Start with `docs/plans/001-role-templates-as-shapes.md`.

## Agent docs (`docs/`)

Same convention as linked-fw packages: `docs/ideas`, `docs/plans`, `docs/reports`, each file numbered with a 3-digit prefix and starting with YAML frontmatter (`summary`, `packages`).
