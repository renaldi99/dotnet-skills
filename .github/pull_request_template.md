<!--
PR title: use Conventional Commits, for example `feat: add <skill-name> skill`,
`fix(efcore-pattern): correct interceptor example`, or `feat!: ...` for breaking changes.
The title becomes the commit message on `master` (squash merge).
Do not edit `CHANGELOG.md`; the maintainer records changes at release time.
-->

## Type of change

- [ ] New skill (Added)
- [ ] Change to existing rules or behavior (Changed)
- [ ] Breaking change (a rule, path, or behavior that changes how agents act for existing users)
- [ ] Fix in examples or documentation (Fixed)
- [ ] Removal of a rule, reference, or skill (Removed)

## Affected skills

<!-- For example: `dotnet-api-design`, `efcore-pattern` -->

## Summary for users

<!-- One or two sentences describing the change from a skill user's point of view.
The maintainer may reuse this text in the CHANGELOG. -->

## Checklist

- [ ] `SKILL.md` frontmatter has `name` (matching the folder name) and a `description` with triggers and scope.
- [ ] `SKILL.md` stays concise (`wc -l skills/*/SKILL.md`: target under ~180 lines, hard limit 500).
- [ ] The skill stands alone: it does not link to, require, or align with another skill.
- [ ] Examples within the skill are consistent (type names, namespaces, signatures, paths) and plausibly compile.
- [ ] Relative links in the changed files are valid.
- [ ] For a new skill: every `README.md` section listed under "Contributing" is updated.
- [ ] `CHANGELOG.md` is not edited.
