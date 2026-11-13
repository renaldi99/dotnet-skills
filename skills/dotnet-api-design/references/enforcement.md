# Enforcement and installation

Read when installing the skill or setting up project guardrails, not for every implementation.
An on-demand skill reduces context, but does not guarantee activation/compliance on every prompt.
Host instructions are not a substitute for deterministic checks either.

## Three layers

| Layer | Purpose |
| :--- | :--- |
| Repository instructions recognized by the host | Route API tasks to the skill before changes; short so they are cheap to load repeatedly. |
| SKILL.md and on-demand references | The primary source of G1-G7, workflows, and examples as needed. |
| Analyzers/tests/CI | Reject measurable violations instead of only asking the AI to remember them. |

## Anchor for the user's backend repository

Merge the snippet below into the repository instructions of **the backend that uses the skill**,
not only the AGENTS.md of the skill catalog repository. Do not overwrite the user's instructions.
Choose the instruction file/skill path according to the host documentation and the actual install location.
This example assumes the host supports `.agents/skills` and the repository instructions you choose.

```markdown
To create, fix, review, or maintain a controller-based .NET API:
- Before editing, read `.agents/skills/dotnet-api-design/SKILL.md`.
- Apply G1-G7; read only the references relevant to the change.
- Preserve existing contracts and patterns. Ask for confirmation if a change
  requires a breaking change or a rule exception that has not been approved.
- After context is summarized/lost, reread the rules before continuing.
- Before finishing, check the diff against G1-G7 and run the narrowest
  relevant checks; report limitations, do not claim success.
```

There is no need to copy all guardrails and templates into every instruction file.
If you use another path, change the anchor; if the host does not support repository instructions,
use the host's rule mechanism or invoke the skill explicitly. Verify discovery on the target host
after installation, including when using nested submodules.

## Enforcement you can add per project

| Rule | Suggested evidence |
| :--- | :--- |
| G1 | A signature analyzer plus a propagation test from the request to the dependency; the signature alone does not prove the token is used. |
| G2/G3 | HTTP integration tests through the real host: body/status/schema, route/version, invalid JSON, and special endpoints without the envelope. |
| G4 | Architecture tests for dependency boundaries/interfaces; avoid string name checks as the only evidence. |
| G5 | Relational integration tests for tracking, query translation, pagination, constraints, and concurrent writes; measure queries for N+1 cases. |
| G6 | An analyzer for static clock usage and time tests with an injected FakeTimeProvider. |
| G7 | Integration tests for domain errors, unexpected errors, auth failures if any, and request cancellation; verify logs without leaking internal details into the response. |

Use the project's existing tooling. Adding a new analyzer/test framework requires
a real need; it is not a prerequisite for installing the skill. Grep/checklists help reviews but are not
semantic proof that every rule is followed.

## Skill evaluation scenarios

Try in a clean session: create a new module, add one field, fix an endpoint bug,
review without editing, and continue maintenance after context compaction.
Check whether references are chosen only as needed, contracts are preserved, and violations are reported.
Do not treat one successful prompt as a guarantee for every model/host.
