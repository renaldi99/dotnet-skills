# Skill catalog maintenance instructions

This file applies only to this skill catalog repository, not to backend projects
that install the skills. The anchor for consuming projects is in
`skills/dotnet-api-design/references/enforcement.md`.

## Source of rules

- `skills/dotnet-api-design/SKILL.md` is the only source of G1-G7. References, README, and
  CHANGELOG only point to or illustrate them; do not write a different version of the rules.
- Each skill stands alone: do not link, require, or align the content of one
  skill with another. Users decide which skills to combine through their prompts.

## When changing a skill

- Keep `SKILL.md` concise (target under ~180 lines, hard limit 500 lines). Long code
  examples go in `references/` and are listed in the "Choose references by task" table.
- Frontmatter must have `name` (matching the folder name) and a `description` that includes
  triggers and scope.
- Examples across files within a single skill must be consistent: type names, namespaces,
  signatures, the `PagedResult<T>` shape, error codes, and paths in the structure diagram.
  Update every affected place at once. This consistency does not apply across skills.
- Do not add dependencies to examples (assertion, validation, mapping libraries) unless
  needed to demonstrate a rule; mention the "follow the existing project" alternative.
- Examples should plausibly compile: complete `using` directives, no helpers referenced
  without a definition, and unit tests that actually assert (not just call a mock).
- Write skill and documentation content in English.

## When adding a new skill

Follow the checklist in the "Contributing" section of `README.md`; every README section
listed there must be updated, not only the catalog table.

## Before declaring done

1. Check that relative links in `SKILL.md`, README, and CHANGELOG are still valid.
2. Check the `SKILL.md` line count (`wc -l skills/*/SKILL.md`).
3. Search for leftovers of removed patterns (for example `grep -rn "<old pattern>" .`).
4. Do not edit `CHANGELOG.md` in a contribution; the maintainer records changes at release
   time. Instead, use a Conventional Commits PR title and fill in the "Type of change" and
   "Summary for users" sections of `.github/pull_request_template.md`. Edit `CHANGELOG.md`
   only when the repository owner explicitly asks (for example, when preparing a release).
5. Do not commit/push without a request from the repository owner.
