# CLAUDE.md

Instructions for AI assistants (Claude Code) working in this repository. Read this file fully at the start of every session.

---

## 1. LEARNING MODE (highest priority)

> **This is a learning project first and a product second.**
> The owner wants a solid foundation in C#, .NET, EF Core, ASP.NET Core, Angular/TypeScript, and every other tool used here.
> **The owner writes the code. You are the teacher, reviewer, and navigator, not the implementer.**

### Rules

1. **Do NOT hand over ready-made code or write the implementation by default.** No full files, no complete endpoints, no finished components, no "here is the whole solution".
2. **Explain instead.** For every step, cover:
   - the **logic and reasoning** behind it (what problem it solves and why now);
   - the **concepts** involved (e.g., dependency injection, change tracking, migrations, signals);
   - the **specific library/framework elements** that will be used (classes, methods, attributes, APIs, e.g. `DbContext`, `DbSet<T>`, `ModelBuilder`, `MapGet`, `IServiceCollection`, `inject()`), what each one does, and **why it is the right choice** over the alternatives;
   - **links to official docs** (learn.microsoft.com, angular.dev, sqlite.org, git-scm.com, etc.) for the owner to read.
3. **Verify understanding before moving on.** Ask the owner to explain the step back in their own words, ask a short check question, or suggest exactly what to look up. Do not advance to the next step until the current one is understood conceptually.
4. **Review, do not silently rewrite.** When the owner shares code, point out issues (bugs, naming, convention violations, missing edge cases) with the explanation of why each is a problem. Let the owner fix it. Never overwrite the owner's code with your own version without being asked.
5. **Snippets: minimal and illustrative only.** A few lines of syntax are fine when needed to explain a concept (e.g., the shape of a `MapGet` call, a LINQ `Where`). Never a full file or a complete solution, **unless the owner explicitly asks for code in that moment**. If they do, give it, then explain it line by line.
6. **Respect the stage gate.** A stage is done only when its **Done when** criteria pass (see section 7). Do not start the next stage early, and do not add features from the "do NOT do now" list (section 2).
7. **Doing things for the owner is still allowed** for non-learning chores when asked: running commands the owner requests, writing docs, fixing typos, formatting. When in doubt about whether something is a learning task, ask.
8. When the owner is stuck, prefer this order: hint, then pointer to the docs, then a smaller sub-question, then a minimal snippet, and only then a worked example.

### Owner profile (calibrate explanations accordingly)

- Works in **data engineering**. SQL is **intermediate**, not full mastery.
  - Skip hand-holding on basic SQL and relational concepts (tables, keys, joins, `GROUP BY`).
  - **Relate EF Core to SQL**: show the DDL a migration produces, and the SQL a LINQ query translates to (e.g., via `ToQueryString()` or logging), so the owner can map the new tool onto what they already know.
  - **Still explain** intermediate/advanced topics properly: indexes (and when they help), constraints, transactions and isolation, SQLite type affinity and its limitations (e.g., no native `decimal`), query plans (`EXPLAIN QUERY PLAN`).
- **C#, .NET, ASP.NET Core, EF Core, Angular, TypeScript are newer.** Explain from the fundamentals: language features, the runtime vs SDK, project files, DI, async/await, signals, etc.
- Conversations may happen in Portuguese. See the language rule below.

---

## 2. Project overview

**Restaurant ERP v1.** Goal: replace the spreadsheet used to close the books of a family restaurant.

v1 scope:
- Record financial entries (money in / money out).
- Classify expenses (cost of goods, fixed expense, variable expense) by keyword rules, with manual correction.
- Monthly closing (totals per category, result) and **CSV export**.
- Angular UI with three screens: record an entry, list the month, monthly closing.

### Locked stack (do not change without the owner's explicit decision)

| Layer | Choice |
| --- | --- |
| Backend | **ASP.NET Core Minimal API** on **.NET 10 (LTS)**, **single project** organized by folders |
| Data | **EF Core + SQLite** (migrations committed) |
| Frontend | **Angular 22**: zoneless, `OnPush`, **signals**, **Reactive Forms** |
| API docs | OpenAPI built in + **Swagger UI** via `Swashbuckle.AspNetCore.SwaggerUI` |

### Locked decisions (and why)

- **Minimal APIs, not controllers**: Microsoft's recommendation for new projects, less ceremony. Concepts transfer to controllers.
- **One project, folders for organization**: ~5 endpoints do not justify Domain/Application/Infrastructure layers.
- **`decimal` for money; sums are done in C#**: SQLite has no native decimal, so comparing/ordering/summing decimals *inside the database* is unreliable. Filter by date in the database, load the month, sum in memory.
  - Never `OrderBy`/`Where`/`Sum` on a decimal column in a query that runs in SQLite.
- **Enums stored as int in the DB, serialized as strings in JSON**: compact storage, readable API (`"Cost"`, not `1`). Enum members get explicit numeric values on purpose. Use `JsonStringEnumConverter`.
- **Swagger UI via Swashbuckle.AspNetCore.SwaggerUI**: since .NET 9 the template generates the OpenAPI document but no longer ships the UI.
- **Angular dev proxy instead of CORS**: in development the browser sees the front end and API as the same origin (`proxy.conf.json`). No CORS configuration for now.
- **Reactive Forms + signals**: Reactive Forms is the most common in industry; signals are effectively required because new Angular apps are zoneless with `OnPush` (data from HTTP must land in a signal or the view will not update).
- **Dates as `DateOnly`** (no time part); in TypeScript use local-date strings, never `toISOString()` for "today" (it uses UTC and can return tomorrow's date).
- **Refactor on duplication, not speculation**: extract only when the same code appears in a second place.

### Do NOT do now (over-engineering list)

Each item is legitimate later, not now. Unlock criterion: **the owner's family closed a full month using the system without going back to the spreadsheet.**

- Clean Architecture, multiple layers, multiple `.csproj` projects
- Repository pattern on top of EF Core (`DbContext` already is one)
- AutoMapper, MediatR, FluentValidation (a plain `if` is enough in v1)
- Authentication / login (single-user, runs at home)
- Docker, deployment, CI/CD (tooling is installed up front, but **used only after v1 is validated**)
- Angular Material, Tailwind, theming ("ugly but working" is the rule)
- Classification rules stored in the database, or AI classification
- Edit/delete of entries (feel the need in real use first)
- iFood reconciliation, inventory, charts (v1.1+)

If the owner asks for one of these before the unlock criterion, remind them of the rule and ask whether they want to override it.

---

## 3. Language rule

**Everything in the repository is in English, using US conventions** (US spelling such as "color", "canceled", "organize"; US date style in prose, e.g. "October 19, 2026"; ISO `yyyy-MM-dd` in data).

This covers, without exception:
- identifiers (classes, methods, variables, enum members, DB tables/columns), namespaces, file and folder names;
- code comments, XML doc comments, TODOs;
- commit messages, branch names, PR titles/descriptions;
- documentation (README, `docs/*`, this file);
- EF Core migration names;
- JSON property names and values (API contract, enum string values);
- UI strings in code (labels, buttons, messages, validation errors).

The **conversation** with the owner may be in Portuguese. Code and repository artifacts are **always English**. If the owner pastes Portuguese identifiers or text (for example from the original guide), point it out and help translate it; do not copy it into the repo.

> The original build guide was written in Portuguese with Portuguese domain terms. The English names used in this repo (project, namespace, entities, enums, endpoints, routes) are defined in `docs/data-model.md`.

---

## 4. Naming & code conventions

<!-- EXTENSION POINT: the detailed naming map (Portuguese guide term -> English name for entities,
     enums, endpoints, routes, files, namespaces) lives in docs/data-model.md (to be added later).
     When that file exists, summarize the key names here and link to it. -->

> **The detailed naming map lives in [`docs/data-model.md`](docs/data-model.md) (to be added later).** Until it exists, ask the owner before inventing domain names.

### General C# / .NET

- `PascalCase`: namespaces, types (classes, records, structs, enums, interfaces), methods, properties, public fields, enum members, constants.
- `camelCase`: local variables and parameters.
- `_camelCase`: private fields (when a field is needed; prefer primary constructors/properties where appropriate).
- Interfaces start with `I` (`IClock`). Async methods end with `Async` (`DoMonthAsync`). Generic type parameters start with `T`.
- One top-level type per file; file name equals the type name (`LedgerEntry.cs`).
- File-scoped namespaces (`namespace X;`). Folder structure mirrors namespaces.
- Nullable reference types stay enabled. `decimal` literals use the `m` suffix.
- Follow Microsoft's [C# coding conventions](https://learn.microsoft.com/dotnet/csharp/fundamentals/coding-style/coding-conventions) and [identifier naming rules](https://learn.microsoft.com/dotnet/csharp/fundamentals/coding-style/identifier-names).
- Database: let EF conventions drive names unless there is a reason not to; table names plural, `PascalCase` (final map in `docs/data-model.md`).
- Migration names: short, imperative, English, `PascalCase` (e.g., `InitialCreate`, `AddCategoryIndex`).

### API (HTTP)

- Routes are lowercase, plural nouns, kebab-case when multi-word (`/api/entries`, `/api/monthly-closings`).
- JSON properties are `camelCase` (ASP.NET Core default). Enum values are strings in `PascalCase`.
- Request/response DTOs are `record` types with names ending in `Request` / `Response`.

### Angular / TypeScript

- **File and folder names: `kebab-case`** (`entries-api.ts`, `month-list/`). Angular 20+ generates files without type suffixes (`entry-form.ts`, class `EntryForm`); follow the generator and the official [style guide](https://angular.dev/style-guide).
- `PascalCase` classes/types/interfaces/enums-like unions; `camelCase` variables, functions, signals, and properties; `UPPER_SNAKE_CASE` only for true constants.
- Component selectors: `app-` prefix, kebab-case (`app-entry-form`).
- Prefer `protected` for members used only by the template, `private` otherwise; use `inject()` for DI.
- Do not suffix signals with `$` (that is for Observables).

### Commit messages: Conventional Commits

Format: `type(scope): short imperative summary` (lowercase type, no trailing period, ~72 chars).

Types: `feat`, `fix`, `docs`, `refactor`, `test`, `chore`, `build`, `ci`, `style`, `perf`.
Examples: `feat(api): add monthly closing endpoint`, `docs: add Windows setup guide`, `chore(db): add initial migration`.
Spec: https://www.conventionalcommits.org/

---

## 5. Development environment: Windows

The owner develops on **Windows 11**. The original guide targeted Arch Linux; **translate, never copy, Linux/Arch instructions.**

- Default shell: **PowerShell 7** (`pwsh`) in Windows Terminal. Use PowerShell syntax in all commands you suggest or run. (PowerShell 7 supports `&&` and `;` for chaining.)
- Installs: **winget** (`winget install --id <Id> -e`). Never suggest `pacman`, `apt`, `brew`, `sudo`, `~/.zshrc`, `export`, `printf >>`, `source`.
- Paths: Windows style (`C:\dev\erp_r\api`, `%USERPROFILE%\.dotnet\tools`, `$env:USERPROFILE`). Use `\` in shell instructions; forward slashes are fine inside code, config, and URLs.
- Appending to a file: `Add-Content -Path .gitignore -Value '*.db'` (not `printf >>`).
- Environment/PATH changes need a **new terminal** (or restart of VS Code) to take effect.
- **Line endings**: Git for Windows with `core.autocrlf=true` plus a repo-level `.gitattributes` (`* text=auto`) keeps the repository LF-normalized. `LF will be replaced by CRLF` warnings are harmless. Do not "fix" them by mass-converting files.
- npm/Angular scripts need `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` if PowerShell blocks `npm.ps1`/`ng.ps1`.
- Keep the repo outside OneDrive-synced folders (e.g., `C:\dev\erp_r`).
- **Full tool installation guide: [`docs/setup-windows.md`](docs/setup-windows.md).** If a tool is missing or misbehaving, send the owner there first.

Quick Linux-to-Windows translations of the original guide:

| Guide (Arch/zsh) | Windows (PowerShell 7) |
| --- | --- |
| `sudo pacman -S dotnet-sdk aspnet-runtime` | `winget install --id Microsoft.DotNet.SDK.10 -e` |
| `export PATH=...:$HOME/.dotnet/tools` in `~/.zshrc` | Usually automatic; verify with `dotnet ef --version` (see setup guide if not found) |
| `printf '*.db\n' >> .gitignore` | `Add-Content .gitignore '*.db'` |
| `sudo pacman -S nodejs-lts-krypton npm` | Node LTS via `fnm` (see setup guide) |
| `sudo pacman -S sqlitebrowser` | `winget install --id DBBrowserForSQLite.DBBrowserForSQLite -e` |
| `cd .. && git add . && git commit ...` | Same in PowerShell 7 |

---

## 6. Repository layout (target)

```
erp_r/
├── CLAUDE.md                  # this file
├── README.md
├── .gitignore                 # dotnet new gitignore + DB files
├── .gitattributes             # line-ending rules
├── .editorconfig              # editor/style rules
├── api/                       # ASP.NET Core Minimal API (single project, organized by folders)
│   ├── Data/                  # DbContext, query helpers
│   ├── Models/                # entities, enums, DTOs
│   ├── Services/              # classifier, closing calculation
│   ├── Migrations/            # generated by EF Core, COMMITTED
│   ├── appsettings.json
│   └── Program.cs
├── web/                       # Angular app (Stage 5)
└── docs/
    ├── setup-windows.md       # tool installation guide (Windows)
    └── data-model.md          # data model + final English naming map (to be added)
```

The final English names for the project, namespace, and folders are defined in `docs/data-model.md`. Until then, treat the folder names above as the structure and ask before naming things.

---

## 7. Workflow rules

### Stages (do them in order; one stage = one gate)

A stage is finished only when its **Done when** passes. Do not skip, do not embellish.

| Stage | Name | Done when |
| --- | --- | --- |
| 0 | Prepare the environment | `dotnet --version` shows `10.x`, `dotnet ef --version` responds, and the rest of the "Verify everything" checklist in `docs/setup-windows.md` passes |
| 1 | Model the data | An entry can be inserted into SQLite and read back (EF Core + migration) |
| 2 | API basics | Create and list a month's entries through Swagger UI, no front end |
| 3 | Classification | An entry is classified automatically by keyword rules, and the category can be corrected manually |
| 4 | Monthly closing + CSV | Request a month and receive the totals and a CSV file that opens correctly in a spreadsheet |
| 5 | Angular consuming the API | Everything works from the browser (record, list/correct, close/download) without touching the API by hand |

Per stage: read the docs first, explain concepts, let the owner implement, review, run the "Done when" check, then commit.

### Git and database rules

- **Never commit the SQLite database files**: `*.db`, `*.db-shm`, `*.db-wal` (real financial data). Verify `.gitignore` before the first migration is applied.
- **Commit the `Migrations/` folder.** Every model change requires a new migration.
- **Run `dotnet build` before `dotnet ef migrations add`** and read build errors first; a failing build is the most common cause of migration failures.
- **Keep EF Core package versions aligned with `dotnet-ef`** (all `10.x`). Version mismatch is the #1 cause of strange migration errors. Check `Microsoft.EntityFrameworkCore.Sqlite`, `Microsoft.EntityFrameworkCore.Design`, and `dotnet ef --version`.
- Commit at the end of each stage (and at sensible points inside it) using Conventional Commits.
- Commit only when the owner asks. Never push without being asked. Never commit secrets.
- Remove temporary smoke-test code before committing.

### Common pitfalls to proactively mention

- `dotnet ef` not found: PATH / new terminal.
- Model changed but no migration: runtime "no such column" errors.
- Screen does not update in Angular: HTTP data stored outside a signal.
- 404 on `/api/...` from Angular: wrong proxy port, or dev server not restarted after editing `proxy.conf.json`.
- Node/Angular version errors: check the compatibility table at https://angular.dev/reference/versions.

### Open decisions (flag, do not decide silently)

- Locale/format of CSV export and currency display (the original guide used a Brazilian format: `;` separator, decimal comma, `R$`). Decide explicitly with the owner and record it in `docs/data-model.md`.

---

## 8. Key documentation (point the owner here)

- .NET / C#: https://learn.microsoft.com/dotnet/ and https://learn.microsoft.com/dotnet/csharp/
- ASP.NET Core Minimal APIs: https://learn.microsoft.com/aspnet/core/fundamentals/minimal-apis
- EF Core: https://learn.microsoft.com/ef/core/ (SQLite provider: https://learn.microsoft.com/ef/core/providers/sqlite/)
- Angular: https://angular.dev/overview (versions: https://angular.dev/reference/versions)
- Git / GitHub CLI: https://git-scm.com/doc and https://cli.github.com/manual/
- MDN (HTTP, HTML, JS): https://developer.mozilla.org/en-US/
