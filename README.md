# 🤖 .NET Agent Skills

A collection of standard **Agent Skills** for AI Coding Assistants ([Google Antigravity](https://antigravity.google), [Claude Code](https://claude.com/product/claude-code), [Cursor](https://cursor.com/), [GitHub Copilot](https://github.com/features/copilot), etc.) designed to speed up building, refactoring, and standardizing backend architecture on **.NET 10 (C# 14)** and **Entity Framework Core**.

---

## 📦 Skill Catalog

This repository contains modular skills that an AI agent can load _on demand_:

<!-- prettier-ignore -->
| Skill | Scope & Focus | Target Platform | Document |
| :--- | :--- | :--- | :---: |
| **`dotnet-api-design`** | Scaffolding, new features, bug fixes, reviews, and maintenance of **Modular Monolith** REST APIs with G1-G7 guardrails and on-demand references. | New projects: .NET 10 / C# 14; maintenance follows the project | [`SKILL.md`](./skills/dotnet-api-design/SKILL.md) |
| **`efcore-pattern`** | High-performance persistence layer design, separate `IEntityTypeConfiguration<T>` configuration, N+1 query elimination, automatic audit interceptors, native SQL bulk update/delete, and standard pagination. | EF Core 9 / 10 | [`SKILL.md`](./skills/efcore-pattern/SKILL.md) |

---

## 🎯 Core Guardrails Summary

Each skill stands alone with its own _core guardrails_ and does not depend on the others.
To use more than one skill, name those skills explicitly in your prompt. Compliance still needs checking;
text instructions alone do not guarantee the AI always activates a skill or follows every rule.

### 1. `dotnet-api-design`

The primary source of rules is **G1-G7 in [SKILL.md](./skills/dotnet-api-design/SKILL.md)**:
cancellation, HTTP contract, versioning, specific repositories, database queries, C#/time,
and error handling. HTTP exceptions and the difference between read-only and tracked mutation
are defined there, not duplicated in the README.

Workflows are split into **scaffolding**, **maintenance**, and **review without edits**.
Long templates are loaded only when needed.

### 2. `efcore-pattern`

- **`.AsNoTracking()` Required**: Every read operation (GET) must use `.AsNoTracking()` or project directly via `.Select()`.
- **Separate Configuration**: Schema, indexes, and table relationships must live in separate `IEntityTypeConfiguration<T>` classes (not Data Annotations on the model).
- **No N+1 Queries**: Use eager loading (`.Include()`) or `.Select()` projection. Do not run database queries inside loops.
- **Native Bulk Operations**: Use `.ExecuteUpdateAsync()` and `.ExecuteDeleteAsync()` for bulk mutations without loading entities into memory.
- **TimeProvider Interceptors**: Automatic audit columns (`CreatedAt`, `UpdatedAt`) must inject the `TimeProvider` abstraction (not a static `DateTime.UtcNow`).
- **`AppDbContext` Only in Repositories**: Services and controllers never inject `AppDbContext`; they depend on specific repository interfaces.

---

## 🚀 Installation

Choose the installation method that best fits your workflow:

### Option 1: Per-Project (Workspace Level) — Recommended

Install the skills in your backend project so the standard is shared by the team and the AI.

The example below uses `.agents/skills`. Make sure your AI host supports that location;
adjust to the skill directory documented by your host if it differs.
Copy the whole skill folder, including `references/`:

```bash
# Run at the root of your .NET project repository:
mkdir -p .agents/skills

# Clone this repo into a temporary folder, then copy the skills you need:
git clone --depth 1 https://github.com/renaldi99/dotnet-skills.git /tmp/dotnet-skills
cp -r /tmp/dotnet-skills/skills/dotnet-api-design .agents/skills/
cp -r /tmp/dotnet-skills/skills/efcore-pattern .agents/skills/
rm -rf /tmp/dotnet-skills
```

The commands above fetch the latest version from the default branch. For a specific release,
add `--branch <tag>` to `git clone`; see the [tags](https://github.com/renaldi99/dotnet-skills/tags)
for available versions.

To install every skill at once, replace the two `cp` lines (before `rm -rf`) with:

```bash
cp -r /tmp/dotnet-skills/skills/* .agents/skills/
```

> **Folder Structure in Your Project:**
>
> ```text
> my-dotnet-project/
> ├── .agents/
> │   └── skills/
> │       ├── dotnet-api-design/
> │       │   ├── SKILL.md
> │       │   └── references/
> │       └── efcore-pattern/
> │           └── SKILL.md
> ├── src/
> └── MyProject.sln
> ```

---

### Option 2: Git Submodule (Easy to Update)

If you want to sync skill updates from this repository into multiple projects easily:

```bash
# At the root of your project:
git submodule add https://github.com/renaldi99/dotnet-skills.git .agents/skills/dotnet-skills
```

**Note:** with a submodule, each skill ends up two levels deeper than hosts usually expect:

```text
      .agents/skills/dotnet-skills/skills/dotnet-api-design/SKILL.md   ← submodule layout
      .agents/skills/dotnet-api-design/SKILL.md                        ← layout most hosts scan
```

Many hosts only discover skills directly under `.agents/skills/<skill-name>/`, so check that your
host finds them. If it does not, use Option 1 (copy each skill from `skills/<skill-name>`) or a
location your host supports; having a submodule does not guarantee discovery.

---

### Option 3: Global Install (Available in Every Project on Your Machine)

If you use **Google Antigravity** and want these skills active in every project without copying them into each workspace:

```bash
# Antigravity Global Customizations
git clone --depth 1 https://github.com/renaldi99/dotnet-skills.git /tmp/dotnet-skills
mkdir -p ~/.gemini/config/skills
cp -r /tmp/dotnet-skills/skills/dotnet-api-design ~/.gemini/config/skills/
cp -r /tmp/dotnet-skills/skills/efcore-pattern ~/.gemini/config/skills/
rm -rf /tmp/dotnet-skills
```

---

## 💡 Usage Trigger Examples

These skills apply **Progressive Disclosure**: metadata helps skill selection,
`SKILL.md` holds the core rules/workflows, and references are read per task. Automatic activation
depends on the host/model; verify the skill is available and invoke it explicitly when needed.

### Scenario 1: Scaffolding a New Module (.NET 10)

> _"Create a new vertical module 'Orders' with POST, GET by id, and paginated GET list endpoints. Make sure it follows our modular monolith architecture."_

### Scenario 2: EF Core Performance Optimization & Refactoring

> _"Review this `CustomerService.cs` service. Why is the query on the get-all endpoint slow? Please optimize it and remove any potential N+1 queries."_

### Scenario 3: Implementing Soft-Delete & Audit Trail

> _"Please create an EF Core interceptor that automatically fills the `CreatedAt` and `UpdatedAt` audit columns using `TimeProvider`."_

### Scenario 4: Maintenance Without Changing the Contract

> _"Use dotnet-api-design to fix the validation on this Orders endpoint. Keep the route, response schema, and framework version; add relevant regression tests."_

### Scenario 5: Combining Two Skills Explicitly

> _"Use dotnet-api-design and efcore-pattern to add a monthly Orders report endpoint. Add a bulk status update via ExecuteUpdateAsync."_

### Strengthening Rule Compliance

Use the [enforcement guide](./skills/dotnet-api-design/references/enforcement.md) to add a short
instruction anchor in **the user's backend repository** and choose the relevant
analyzers/tests/CI. The `AGENTS.md` in this skill catalog does not automatically apply to consuming projects.
Do not copy every template into permanent instructions; point to G1-G7 and read references as needed.

---

## 📁 Repository Structure

```text
dotnet-skills/
├── README.md                           # Main documentation and skill catalog
├── CHANGELOG.md                        # Change history
├── AGENTS.md                           # Catalog maintenance instructions (not for consuming projects)
├── LICENSE                             # MIT License
├── .gitignore                          # Ignore rules for OS files and temporary caches
├── .github/
│   └── pull_request_template.md        # PR checklist and change type for contributors
│
└── skills/                             # All skills, one folder per skill
    ├── dotnet-api-design/              # .NET 10 RESTful API architecture skill
    │   ├── SKILL.md                    # G1-G7, reference routing, and concise workflows
    │   └── references/
    │       ├── foundation.md           # Envelope, error codes, exception handler
    │       ├── module-scaffolding.md   # Employees module example
    │       ├── bootstrap.md            # DI, automatic validation, HTTP pipeline
    │       ├── database-performance.md # Queries, pagination, uniqueness, bulk
    │       └── enforcement.md          # Host instructions and automated checks
    │
    └── efcore-pattern/                 # EF Core 9/10 design pattern skill
        └── SKILL.md                    # Entity configuration, interceptor, & bulk SQL guide
```

---

## 🤝 Contributing

Want to add a new skill or update an existing guide?

1. Fork this repository.
2. Create a new skill directory: `skills/<skill-name>/SKILL.md`.
3. Make sure `SKILL.md` includes YAML frontmatter in this format:
   ```markdown
   ---
   name: skill-name
   description: >-
     Describe when this skill is used (activation triggers) and its boundaries.
   ---
   ```
4. Put long guides in a `references/` subfolder.
5. Make sure the skill stands alone: do not link to or require other skills.
6. Register the new skill in every one of these `README.md` sections:
   - [ ] **Skill Catalog**: a table row (name, scope, target platform, `SKILL.md` link).
   - [ ] **Core Guardrails Summary**: a `### N. <skill-name>` subsection that points to
         `SKILL.md` instead of copying every rule.
   - [ ] **Installation**: `cp -r` commands in Option 1 and Option 3, plus the skill folder in
         the "Folder Structure in Your Project" diagram.
   - [ ] **Usage Trigger Examples**: at least one scenario; add a combined prompt example
         when the skill is commonly used together with another skill.
   - [ ] **Repository Structure**: the skill folder with `SKILL.md` and `references/` if any.
7. Do not edit `CHANGELOG.md`; the maintainer records changes there at release time.
8. Open a Pull Request with a [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/)
   title (for example `feat: add <skill-name> skill`) and fill in the
   [pull request template](./.github/pull_request_template.md), including the
   "Summary for users" section.

---

## 📄 License

Distributed under the [MIT License](LICENSE). Free to use, modify, and distribute for commercial and non-commercial projects.
