# Data Model and Naming Map

Stage 1 reference for the Restaurant ERP v1: what the data looks like, what everything is called in English, and how to build and verify it on Windows.

> **Learning mode.** This document describes *what* each file must contain, *why*, and which library elements are involved. It deliberately does **not** contain the finished C# files. You write the code; use the linked official docs to learn each element, then compare your result against the "Done when" checklist in section 11.

**Contents**

1. [Naming map (Portuguese guide to English)](#1-naming-map-portuguese-guide-to-english)
2. [The `Transaction` entity](#2-the-transaction-entity)
3. [Enums](#3-enums)
4. [`AppDbContext` and the Fluent API](#4-appdbcontext-and-the-fluent-api)
5. [Connection string, `Program.cs` changes, `.gitignore`](#5-connection-string-programcs-changes-gitignore)
6. [Expected SQLite schema and EF Core to SQL mapping](#6-expected-sqlite-schema-and-ef-core-to-sql-mapping)
7. [SQLite specifics you must understand](#7-sqlite-specifics-you-must-understand)
8. [Business rules and invariants](#8-business-rules-and-invariants)
9. [Rejected extra fields](#9-rejected-extra-fields)
10. [Target folder structure](#10-target-folder-structure)
11. [Stage 1: Done when and Windows checklist](#11-stage-1-done-when-and-windows-checklist)
12. [Locale and currency policy](#12-locale-and-currency-policy)
13. [Open decisions](#13-open-decisions)

---

## 1. Naming map (Portuguese guide to English)

The original build guide was written in Portuguese. This table is the single source of truth for the English names used in the repository. If a name is missing here, add it here first.

### 1.1 Project, folders, and files

| Guide (PT) | English | Why |
| --- | --- | --- |
| Repo `erp-restaurante/` | Repo root `erp_r/` | The repository already exists under this name. |
| `ErpRestaurante.Api` (project, `.csproj`, root namespace) | `RestaurantErp.Api` | English, modifier first. .NET naming guidelines PascalCase acronyms of three or more letters (`Erp`, not `ERP`). |
| `api/`, `web/` | same | Already English. |
| `Data/`, `Models/`, `Services/`, `Migrations/` | same | Standard .NET folder names. |
| `Models/Enums.cs` | `Models/TransactionType.cs` and `Models/ExpenseCategory.cs` | One public type per file. |
| `Models/Lancamento.cs` | `Models/Transaction.cs` | See the domain table. |
| `Models/Dtos.cs` (Stage 2) | `Models/Dtos.cs` | Allowed exception to one-type-per-file: a few small request records. |
| `Data/ConsultasLancamento.cs` (Stage 4) | `Data/TransactionQueries.cs` | Convention: `<Entity>Queries` static class holding extension methods. |
| `Services/Classificador.cs` (Stage 3) | `Services/CategoryClassifier.cs` | |
| `Services/CalculoFechamento.cs` (Stage 4) | `Services/MonthlyClosingCalculator.cs` | "Monthly closing" (month-end close) is the standard accounting term. |
| record `FechamentoMensal` | `Models/MonthlyClosing.cs` containing `record MonthlyClosing` | Shape of the closing response. |
| Migration `Inicial` | `InitialCreate` | The name used throughout the EF Core documentation. |
| DB file `erp.db` | `erp.db` | Already neutral. |
| Connection string key `Default` | `DefaultConnection` | Matches the Microsoft template convention. |

### 1.2 Domain

| Guide (PT) | English | Why |
| --- | --- | --- |
| Entity `Lancamento` | **`Transaction`** | The common US small-business bookkeeping term (QuickBooks, Wave). Rejected: `Entry` (collides with `DbContext.Entry()`), `JournalEntry` (implies double-entry accounting), `LedgerEntry` (verbose). No clash with `System.Transactions` because that namespace is not imported by default. |
| `DbSet`/table `Lancamentos` | `DbSet<Transaction> Transactions` / table `Transactions` | EF Core takes the table name from the `DbSet` property name. |
| Enum `TipoLancamento` | `TransactionType` | |
| `Entrada` / `Saida` | `Income` / `Expense` | The closing becomes a simple cash-basis P&L: income minus expenses equals net income. |
| Enum `Categoria` | `ExpenseCategory` (the property is still called `Category`) | The type name itself encodes the rule "categories apply to expenses only". |
| `Custo` | `CostOfGoodsSold` | COGS, the US term for what the guide calls CMV. UI label: "Cost of goods sold (COGS)". |
| `DespesaFixa` | `FixedExpense` | |
| `DespesaVariavel` | `VariableExpense` | |
| `Id, Data, Valor, Descricao, Tipo, Categoria` | `Id, Date, Amount, Description, Type, Category` | |
| `AppDbContext` | `AppDbContext` | Already neutral. |

> **Terminology rule: "Transaction" always means a financial transaction** (one row of income or expense). A *database* transaction is always called by its full name, `IDbContextTransaction` (or "DB transaction"). Never use the bare word for both, and do not import `System.Transactions` in this project.

### 1.3 Later-stage names (fixed now so nothing gets renamed later)

| Guide (PT) | English |
| --- | --- |
| `CriarLancamentoRequest(Data, Valor, Descricao, Tipo, Categoria)` | `CreateTransactionRequest(Date, Amount, Description, Type, Category)` |
| `CorrigirCategoriaRequest(Categoria)` | `UpdateCategoryRequest(ExpenseCategory Category)` |
| `ConsultasLancamento.DoMesAsync(ano, mes)` | `TransactionQueries.ForMonthAsync(year, month)`, used as `db.Transactions.ForMonthAsync(2026, 9)` |
| `Classificador.Sugerir(descricao)` | `CategoryClassifier.Suggest(description)`, returns `ExpenseCategory?` |
| `Regras`, `Chave`, `Separadores`, `Normalizar` | `Rules`, `(string Keyword, ExpenseCategory Category)`, `Separators`, `Normalize` |
| `CalculoFechamento.Calcular` / `ParaCsv` / `Br` | `MonthlyClosingCalculator.Calculate` / `ToCsv` / `FormatAmount` |
| `FechamentoMensal(Ano, Mes, TotalEntradas, TotalSaidas, Custo, DespesaFixa, DespesaVariavel, SemCategoria, Resultado)` | `MonthlyClosing(Year, Month, TotalIncome, TotalExpenses, CostOfGoodsSold, FixedExpenses, VariableExpenses, Uncategorized, NetIncome)` |
| Routes `/api/lancamentos`, `PUT /{id:int}/categoria` | `/api/transactions`, `PUT /api/transactions/{id:int}/category` |
| Query string `?ano=&mes=` | `?year=&month=` |
| `/api/fechamentos/{ano}/{mes}` and `.../csv` | `/api/monthly-closings/{year:int}/{month:int}` and `.../csv` |
| `fechamento-2026-09.csv` | `monthly-closing-2026-09.csv` |
| Swagger title "ERP v1" | "Restaurant ERP v1" |
| Angular `lancamentos-api` / `LancamentosApi` | `transactions-api.ts` / `TransactionsApi` |
| TS methods `criar`, `listar`, `corrigirCategoria`, `fechamento` | `create`, `listByMonth`, `updateCategory`, `getMonthlyClosing` |
| TS `Tipo`, `Categoria`, `Lancamento`, `NovoLancamento`, `ResumoFechamento`, `hojeISO()` | `TransactionType`, `ExpenseCategory`, `Transaction`, `NewTransaction`, `MonthlyClosing`, `todayIso()` |
| Pages `lancar`, `lista-mes`, `fechamento` | `pages/new-transaction` (class `NewTransaction`), `pages/monthly-transactions` (`MonthlyTransactions`), `pages/monthly-closing` (`MonthlyClosingPage`, to avoid a clash with the `MonthlyClosing` interface) |
| Angular routes `lancar`, `lancamentos`, `fechamento` | `new`, `transactions`, `closing` |

---

## 2. The `Transaction` entity

One row = one money movement (income or expense). Table name: `Transactions`.

| Field | C# type | Nullable | SQLite column | Constraints / default | Why |
| --- | --- | --- | --- | --- | --- |
| `Id` | `int` | no | `INTEGER` primary key, `AUTOINCREMENT` | Generated by the database (EF Core convention: a property named `Id` is the key). | Clients never send it. Gives a stable tie-break when sorting (`ThenBy(t => t.Id)`). |
| `Date` | `DateOnly` | no | `TEXT` in `yyyy-MM-dd` form | Required. Stage 2 rejects `default` (0001-01-01). | The business date has no time or time zone. ISO text sorts chronologically, so `>=` / `<` month filters work correctly inside the database. |
| `Amount` | `decimal` | no | `TEXT` | Always greater than 0, at most 2 decimal places (checked by the app). `HasPrecision(18, 2)` documents the intent. | No binary rounding error (unlike `double`). SQLite stores decimals as TEXT, so **never filter, sort, or sum by `Amount` in SQL**; do it in C#. |
| `Description` | `string` | no | `TEXT` | Required, trimmed, at most 200 characters (`HasMaxLength(200)`). C# default `""`. | SQLite does not enforce max length, so the Stage 2 endpoint must check it. |
| `Type` | `TransactionType` | no | `INTEGER` | Must be a defined value (1 or 2). | Sets the direction (income or expense) so `Amount` can stay positive. |
| `Category` | `ExpenseCategory?` | **yes** | `INTEGER NULL` | `null` for income, and `null` for an expense that has not been categorized. | An uncategorized expense shows up as "Uncategorized" in the monthly closing, which is a useful warning. |

Other facts:

- **Index:** `IX_Transactions_Date` on `Date`. The most common query is "transactions for a month".
- **Enums as int:** EF Core stores enums as integers by default; no value converter is needed. JSON uses strings through `JsonStringEnumConverter` (configured in Stage 2).
- **Why enum values start at 1:** a missing enum value in a JSON request deserializes to `0`. Because no member has the value 0, `Enum.IsDefined` catches it.

### Concepts and library elements for the entity file

Read these before writing `Models/Transaction.cs`:

| Element | What it is and why it is used here | Docs |
| --- | --- | --- |
| Entity class (POCO) | A plain C# class that EF Core maps to a table; each public property with a getter and setter maps to a column. | [EF Core: creating a model](https://learn.microsoft.com/ef/core/modeling/) |
| Key by convention (`Id`) | A property named `Id` or `<TypeName>Id` becomes the primary key, and an `int` key is generated by the database. | [Keys](https://learn.microsoft.com/ef/core/modeling/keys) |
| `DateOnly` | .NET type for a calendar date without time. Fits "business date". EF Core's SQLite provider stores it as ISO `TEXT`. | [DateOnly and TimeOnly](https://learn.microsoft.com/dotnet/standard/datetime/how-to-use-dateonly-timeonly) |
| `decimal` and the `m` suffix | Base-10 floating type for money. A literal such as `150.00m` is a `decimal`; `150.00` alone is a `double`. | [Floating-point numeric types](https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/floating-point-numeric-types) |
| Nullable value type (`ExpenseCategory?`) | `Nullable<T>`: the enum **or** `null`. EF Core maps it to a nullable column. | [Nullable value types](https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/nullable-value-types) |
| Nullable reference types | Why `string Description` needs an initializer (the compiler warns about a possible `null`). | [Nullable reference types](https://learn.microsoft.com/dotnet/csharp/nullable-references) |
| File-scoped namespace | `namespace RestaurantErp.Api.Models;` applies to the whole file. | [Namespaces](https://learn.microsoft.com/dotnet/csharp/fundamentals/types/namespaces) |

Questions to be able to answer before moving on: Why is `Amount` a `decimal` and not a `double`? Why is `Category` nullable while `Type` is not? Why is there no `Id` in a request later?

---

## 3. Enums

Two files, one public enum each, with **explicit numeric values starting at 1**. The number is what SQLite stores, so explicit values make stored data safe if someone reorders the members later.

### `TransactionType`

| Member | Value | Meaning and examples |
| --- | --- | --- |
| `Income` | 1 | Money in: daily sales, card settlements. |
| `Expense` | 2 | Money out: purchases, bills, payroll. |
| (unused) | 0 | Intentionally has no member; 0 means "missing". |

### `ExpenseCategory` (expenses only)

| Member | Value | Meaning and examples |
| --- | --- | --- |
| `CostOfGoodsSold` | 1 | Goes into a dish or is resold: meat, tomatoes, beverages for resale. |
| `FixedExpense` | 2 | Keeps the doors open, stable amount: rent, base salaries, accountant, internet. |
| `VariableExpense` | 3 | Keeps the doors open, fluctuating amount: electricity, gas, card fees. |

Docs: [Enumeration types](https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/enum) and [EF Core: enum mapping and value conversions](https://learn.microsoft.com/ef/core/modeling/value-conversions).

---

## 4. `AppDbContext` and the Fluent API

`Data/AppDbContext.cs` is the class that represents your database session. Its required contents:

- It **inherits from `DbContext`**.
- A **constructor** that accepts `DbContextOptions<AppDbContext>` and passes it to the base constructor. Use the classic constructor syntax, not a C# 12 primary constructor, to stay consistent with the learning material.
- One `DbSet<Transaction>` property named `Transactions`.
- An override of `OnModelCreating(ModelBuilder modelBuilder)` configuring the entity:

| Configuration | Purpose |
| --- | --- |
| `Description`: `HasMaxLength(200)` and `IsRequired()` | Documents the length and `NOT NULL`. The SQLite provider does not enforce the length. |
| `Amount`: `HasPrecision(18, 2)` | Documents the intent (money, two decimals). Enforced by PostgreSQL / SQL Server later; **not enforced by SQLite**. This is the only addition to the original guide's model. |
| `HasIndex` on `Date` | Creates `IX_Transactions_Date` for the month filter. |

### Library elements

| Element | What it does and why it is the right choice | Docs |
| --- | --- | --- |
| `DbContext` | The unit-of-work session with the database: tracks entities you load or add, and writes the changes on `SaveChanges`. Registered with a *scoped* lifetime (one per request). | [DbContext lifetime, configuration, and initialization](https://learn.microsoft.com/ef/core/dbcontext-configuration/) |
| `DbSet<T>` | The table as seen from C#. You query and add through it. The property name becomes the table name. | [DbSet<TEntity> API](https://learn.microsoft.com/dotnet/api/microsoft.entityframeworkcore.dbset-1) |
| `DbContextOptions<T>` | Carries the provider and connection settings into the context through the constructor, so the context itself does not know about SQLite. This is how dependency injection configures it. | [DbContextOptions<TContext> API](https://learn.microsoft.com/dotnet/api/microsoft.entityframeworkcore.dbcontextoptions-1) |
| `OnModelCreating` and `ModelBuilder` | The hook where you configure the model with the **Fluent API**; `ModelBuilder` is the entry point (`modelBuilder.Entity<T>()`). Fluent API wins over conventions and data annotations. | [ModelBuilder API](https://learn.microsoft.com/dotnet/api/microsoft.entityframeworkcore.modelbuilder), [Modeling overview](https://learn.microsoft.com/ef/core/modeling/) |
| `EntityTypeBuilder<T>` | Returned by `Entity<T>()`. Exposes `Property(...)`, `HasIndex(...)`, `HasKey(...)`, `ToTable(...)`. | [EntityTypeBuilder<T> API](https://learn.microsoft.com/dotnet/api/microsoft.entityframeworkcore.metadata.builders.entitytypebuilder-1) |
| `Property(...)`, `HasMaxLength`, `IsRequired`, `HasPrecision` | Per-property configuration. | [Entity properties](https://learn.microsoft.com/ef/core/modeling/entity-properties) |
| `HasIndex` | Declares a (non-unique) index; migrations generate the `CREATE INDEX`. | [Indexes](https://learn.microsoft.com/ef/core/modeling/indexes) |

A tiny illustration of the *shape* (not your file): the lambda `t => t.Date` tells EF Core which property is being configured, with compile-time checking instead of a magic string.

---

## 5. Connection string, `Program.cs` changes, `.gitignore`

### `appsettings.json`

Add a `ConnectionStrings` section at the root with key **`DefaultConnection`** and the value `Data Source=erp.db`. For SQLite the "connection string" is just the file path. The path is relative to the **working directory**, so always run `dotnet run` and `dotnet ef` from `api/`; otherwise a second `erp.db` appears somewhere else.

Docs: [Connection strings](https://learn.microsoft.com/ef/core/miscellaneous/connection-strings), [SQLite connection strings](https://learn.microsoft.com/dotnet/standard/data/sqlite/connection-strings), [Configuration in ASP.NET Core](https://learn.microsoft.com/aspnet/core/fundamentals/configuration/).

### `Program.cs` (Stage 1 changes)

1. Add the `using` directives for `Microsoft.EntityFrameworkCore`, `RestaurantErp.Api.Data`, and `RestaurantErp.Api.Models`.
2. Right after creating the builder, register the context with `AddDbContext<AppDbContext>`, whose lambda calls `UseSqlite` with the value read through `GetConnectionString("DefaultConnection")`.
3. Delete the template's WeatherForecast sample and the `app.UseHttpsRedirection();` line (HTTPS returns when you deploy). Keep `AddOpenApi()` and `MapOpenApi()`.

| Element | What it does | Docs |
| --- | --- | --- |
| `IServiceCollection` / `builder.Services` | The dependency-injection registry that you fill before `Build()`. | [Dependency injection in ASP.NET Core](https://learn.microsoft.com/aspnet/core/fundamentals/dependency-injection) |
| `AddDbContext<T>` | Registers your context with a scoped lifetime and wires its `DbContextOptions<T>`. | [AddDbContext API](https://learn.microsoft.com/dotnet/api/microsoft.extensions.dependencyinjection.entityframeworkservicecollectionextensions.adddbcontext) |
| `UseSqlite` | Selects the SQLite provider (from `Microsoft.EntityFrameworkCore.Sqlite`) and passes the connection string. | [SQLite provider](https://learn.microsoft.com/ef/core/providers/sqlite/) |
| `GetConnectionString(name)` | Shortcut for reading `ConnectionStrings:<name>` from configuration. | [API](https://learn.microsoft.com/dotnet/api/microsoft.extensions.configuration.configurationextensions.getconnectionstring) |

### `.gitignore`

Create the repository-root file with `dotnet new gitignore` (the official .NET template), then append these four lines with `Add-Content`:

```
*.db
*.db-shm
*.db-wal
*.db-journal
```

Why four: the database is real financial data and must never be committed. EF Core opens SQLite in **WAL mode**, which creates `-wal` and `-shm` side files next to the main file; `-journal` appears in non-WAL (rollback journal) mode. **`Migrations/` is committed.**

Docs: [`dotnet new` gitignore template](https://learn.microsoft.com/dotnet/core/tools/dotnet-new-sdk-templates#gitignore), [`git check-ignore`](https://git-scm.com/docs/git-check-ignore).

---

## 6. Expected SQLite schema and EF Core to SQL mapping

The `InitialCreate` migration should produce (formatting may differ slightly by EF Core version):

```sql
CREATE TABLE "Transactions" (
    "Id"          INTEGER NOT NULL CONSTRAINT "PK_Transactions" PRIMARY KEY AUTOINCREMENT,
    "Date"        TEXT    NOT NULL,
    "Amount"      TEXT    NOT NULL,
    "Description" TEXT    NOT NULL,
    "Type"        INTEGER NOT NULL,
    "Category"    INTEGER NULL
);

CREATE INDEX "IX_Transactions_Date" ON "Transactions" ("Date");

-- plus EF's own bookkeeping table:
-- "__EFMigrationsHistory" ("MigrationId" TEXT PK, "ProductVersion" TEXT)
```

Use this to check your result in a SQLite tool or in the `Up` method of the generated migration.

### EF Core concept to SQL concept

| EF Core / C# | SQL equivalent | Notes |
| --- | --- | --- |
| `DbContext` | A connection plus a unit of work | Tracks changes; `SaveChanges` sends them in **one DB transaction** (`IDbContextTransaction` under the hood). |
| `DbSet<Transaction>` | Table `Transactions` | Table name comes from the property name. |
| Property | Column | Name = property name; type from the provider's type mapping. |
| `ExpenseCategory?` | `INTEGER NULL` | Nullable C# type means nullable column. |
| `Id` convention | `PRIMARY KEY AUTOINCREMENT` | Database-generated. |
| `HasIndex(t => t.Date)` | `CREATE INDEX` | Name defaults to `IX_<Table>_<Column>`. |
| `IsRequired()` | `NOT NULL` | |
| `HasMaxLength` / `HasPrecision` | `VARCHAR(n)` / `DECIMAL(p,s)` on other databases | SQLite accepts the type names but ignores the limits. |
| Migration (`Up`/`Down`) | Versioned DDL script | History is recorded in `__EFMigrationsHistory`. `migrations add` generates the C# file; `database update` runs the DDL. |
| `db.Transactions.Where(...).OrderBy(...)` | `SELECT ... WHERE ... ORDER BY ...` | Inspect the SQL with `query.ToQueryString()` or [simple logging](https://learn.microsoft.com/ef/core/logging-events-diagnostics/simple-logging). |
| `Add` + `SaveChanges` | `INSERT` | EF also runs `SELECT last_insert_rowid()` style logic to read back the generated `Id`. |
| Changing a tracked property + `SaveChanges` | `UPDATE` of the changed column only | This is *change tracking*. |

Docs: [Migrations overview](https://learn.microsoft.com/ef/core/managing-schemas/migrations/), [`dotnet ef migrations add`](https://learn.microsoft.com/ef/core/cli/dotnet#dotnet-ef-migrations-add), [Querying data](https://learn.microsoft.com/ef/core/querying/), [Saving data](https://learn.microsoft.com/ef/core/saving/basic).

---

## 7. SQLite specifics you must understand

You know relational basics; SQLite behaves differently from the servers you work with day to day.

1. **Type affinity, not strict types.** SQLite columns have an *affinity* (TEXT, INTEGER, REAL, NUMERIC, BLOB), not a rigid type. A value can be stored in a column whose affinity does not match it, and the declared type name (`VARCHAR(200)`, `DECIMAL(18,2)`) only determines the affinity by naming rules. (SQLite has an opt-in `STRICT` table mode, but the default tables EF Core creates are not strict; check the provider docs before relying on it.) Read: [SQLite datatypes](https://www.sqlite.org/datatype3.html), [EF Core SQLite type mappings](https://learn.microsoft.com/ef/core/providers/sqlite/type-mappings).
2. **`decimal` is stored as TEXT.** SQLite has no fixed-point type; the NUMERIC/REAL alternatives would reintroduce floating-point error, so EF Core's SQLite provider stores `decimal` as TEXT. Consequences: `ORDER BY Amount` sorts as text (`"100"` before `"20"`), `WHERE Amount > 100` compares text, and SQL `SUM` is not reliable. **That is why this project filters by date in SQL, loads the month, and sums in C#.** A month of a restaurant is hundreds of rows, not millions, so this is the simple and correct choice. Read: [SQLite limitations](https://learn.microsoft.com/ef/core/providers/sqlite/limitations) and [client vs server evaluation](https://learn.microsoft.com/ef/core/querying/client-eval).
   - A visible side effect: a stored `150.00m` may print as `150.0` or `150`, because trailing zeros are not preserved in the text form. The value is the same.
3. **`DateOnly` is ISO TEXT.** `yyyy-MM-dd` sorts lexicographically in date order, so `Date >= '2026-09-01' AND Date < '2026-10-01'` is both correct and index-friendly.
4. **Max length is not enforced.** `HasMaxLength(200)` does not make SQLite reject a 500-character string. The API must validate it (Stage 2). Same for precision.
5. **`AUTOINCREMENT`.** Guarantees that ids are never reused after deletes (it keeps a counter in `sqlite_sequence`). Plain `INTEGER PRIMARY KEY` would already alias the rowid. See [SQLite autoincrement](https://www.sqlite.org/autoinc.html).
6. **WAL mode and the extra files.** The default journaling for EF Core databases is write-ahead logging: `erp.db-wal` and `erp.db-shm` appear while the database is open. They belong to the database; copying only `erp.db` while the app runs can lose recent writes. Read: [Write-ahead logging](https://www.sqlite.org/wal.html).
7. **No `CHECK` constraint on `Amount`, deliberately.** A `CHECK (Amount > 0)` would compare TEXT, not numbers, and give wrong results (`'9' > '0'` but also `'100' < '20'`). The rule is enforced in the API instead (section 8). On PostgreSQL later, a real `numeric` column makes a CHECK reasonable; it is listed in the rejected fields below as a future candidate.
8. **Query plans.** `EXPLAIN QUERY PLAN SELECT ... WHERE Date >= ... AND Date < ...` should show `SEARCH Transactions USING INDEX IX_Transactions_Date`. Try it in DB Browser for SQLite or the `sqlite3` shell to see the index at work; remove the index in a scratch copy and compare (`SCAN Transactions`). See [EXPLAIN QUERY PLAN](https://www.sqlite.org/eqp.html).
9. **Limited `ALTER TABLE`.** Historically SQLite cannot alter or drop many column details in place; EF Core migrations work around it by rebuilding tables. Keep that in mind when you change the model later.

---

## 8. Business rules and invariants

| # | Rule | Enforced where |
| --- | --- | --- |
| 1 | `Amount > 0`; `Type` alone sets the direction. | Stage 2 `POST` returns 400 "Amount must be greater than zero."; Stage 5 `Validators.min(0.01)` (UX only). No DB `CHECK` (section 7.7). |
| 2 | At most 2 decimal places. | Optional Stage 2 check comparing the amount with `decimal.Round(amount, 2)` returns 400. |
| 3 | `Type` must be `Income` or `Expense`; 0 means missing. | Stage 2 `Enum.IsDefined`; `JsonStringEnumConverter` with `allowIntegerValues: false`. |
| 4 | `Category` is only for expenses; income is always `null`. | Stage 2/3 `POST` forces `null` for income; Stage 3 `PUT` returns 400 "Income transactions cannot have a category."; the classifier is only called for expenses. |
| 5 | Expense category precedence: value sent by the client, then `CategoryClassifier.Suggest`, then `null` ("Uncategorized"). A provided category must be defined. | Stage 3 `POST` (`req.Category ?? CategoryClassifier.Suggest(...)`); `Enum.IsDefined` in Stage 2 `POST` and Stage 3 `PUT`. |
| 6 | `Type` is immutable after creation; only `Category` can change. | No edit endpoint in v1. |
| 7 | `Description` is required, trimmed, and at most 200 characters. | Stage 2 `POST`: `IsNullOrWhiteSpace`, trim, length check (SQLite will not). |
| 8 | `Date` is required and date-only. A month is `[first day, first day of next month)`. | Stage 2 rejects `Date == default`; month bounds live only in `TransactionQueries.ForMonthAsync`; Stage 5 uses a local `todayIso()`, never `toISOString()`. |
| 9 | Query year 2000-2100 and month 1-12. | Stage 2 `GET` and Stage 4 return 400 "Invalid year/month." |
| 10 | All math on `Amount` happens in C#, in memory. The database only filters and sorts by `Date` and `Id`. | Stages 2 and 4. |
| 11 | `TotalExpenses = COGS + Fixed + Variable + Uncategorized`; `NetIncome = TotalIncome - TotalExpenses`. | Stage 4 calculator; verify by hand in Stage 4 testing. |
| 12 | Ids are generated by the database; clients never send them. | `CreateTransactionRequest` has no `Id`. |

Errors are plain English strings through `Results.BadRequest("...")`. No ProblemDetails or FluentValidation in v1.

---

## 9. Rejected extra fields

Anti-over-engineering: each of these was considered and **deliberately left out of v1**. Add one only if real use proves the need (see the unlock rule in `CLAUDE.md`).

| Field / feature | Why rejected |
| --- | --- |
| `CreatedAt` / `UpdatedAt` | `Id` already gives insertion order; no edit endpoint exists. |
| `IsCategoryManual` / `CategorySource` | The classifier runs only at creation; a manual correction simply overwrites `Category`. |
| `Currency` | Single currency (BRL). Documented assumption. |
| `Notes`, `Supplier`, `PaymentMethod` | Scope creep; nothing in v1 reads them. |
| Soft delete, `RowVersion` (concurrency token) | There is no edit or delete, and a single user. |
| Database `CHECK` constraints | `Amount` is TEXT in SQLite (unreliable comparison), and they would duplicate the endpoint rules. A candidate when moving to PostgreSQL. |

Accepted addition to the original guide: `HasPrecision(18, 2)` on `Amount` (documentation of intent, zero runtime cost).

---

## 10. Target folder structure

```
erp_r/
├── .gitignore
├── .gitattributes
├── .editorconfig
├── CLAUDE.md
├── README.md
├── docs/
│   ├── setup-windows.md
│   └── data-model.md                        (this file)
├── api/                                     project RestaurantErp.Api
│   ├── Data/
│   │   ├── AppDbContext.cs
│   │   └── TransactionQueries.cs            (Stage 4)
│   ├── Models/
│   │   ├── ExpenseCategory.cs
│   │   ├── Transaction.cs
│   │   ├── TransactionType.cs
│   │   ├── Dtos.cs                          (Stage 2)
│   │   └── MonthlyClosing.cs                (Stage 4)
│   ├── Services/
│   │   ├── CategoryClassifier.cs            (Stage 3)
│   │   └── MonthlyClosingCalculator.cs      (Stage 4)
│   ├── Migrations/                          (generated; committed)
│   ├── Properties/launchSettings.json
│   ├── appsettings.json
│   ├── appsettings.Development.json
│   ├── Program.cs
│   ├── RestaurantErp.Api.csproj
│   ├── RestaurantErp.Api.http
│   └── erp.db                               (generated; git-ignored)
└── web/                                     Angular 22 (Stage 5)
    ├── proxy.conf.json
    └── src/app/
        ├── app.routes.ts
        ├── transactions-api.ts
        └── pages/{new-transaction, monthly-transactions, monthly-closing}/
```

---

## 11. Stage 1: Done when and Windows checklist

### Done when

1. `dotnet build` succeeds with 0 errors.
2. `Microsoft.EntityFrameworkCore.Sqlite` and `Microsoft.EntityFrameworkCore.Design` are both `10.x`, matching `dotnet ef --version`.
3. `api\Migrations\` contains `<timestamp>_InitialCreate.cs`, `<timestamp>_InitialCreate.Designer.cs`, and `AppDbContextModelSnapshot.cs`.
4. `api\erp.db` has the table `Transactions` with the columns from section 6 and the index `IX_Transactions_Date`.
5. Smoke test: the first `dotnet run` prints 1 row; the second run prints 2 rows (proof of persistence).
6. The smoke-test block is removed and the project still builds; `git check-ignore api/erp.db` prints the path; no `.db` file shows up in `git status`.
7. No Portuguese identifiers remain in `api\**\*.cs`.
8. Committed with a Conventional Commit, for example `feat(api): add transaction model with SQLite persistence`.

### Step-by-step checklist (PowerShell 7, from the repository root `C:\dev\erp_r`)

Each step says what to do and why. You write the C#; the commands are yours to run.

1. **Verify the toolchain.** `dotnet --version` should print `10.x`; `dotnet ef --version` should respond. If not, go back to [`setup-windows.md`](setup-windows.md).
2. **Create the root `.gitignore`.** Run `dotnet new gitignore` in the repo root (git is already initialized; skip `git init`). Check it exists with `Test-Path .gitignore`.
3. **Create the API project.** `dotnet new webapi -n RestaurantErp.Api -o api`, then `Set-Location api`. Read [`dotnet new` templates](https://learn.microsoft.com/dotnet/core/tools/dotnet-new-sdk-templates#web-api) to see what was generated.
4. **Clean `Program.cs`.** Remove the WeatherForecast sample and `app.UseHttpsRedirection();`. Keep the OpenAPI lines. Run `dotnet build` to confirm you did not break anything.
5. **Add the EF Core packages** (both, same major version as `dotnet-ef`):
   ```powershell
   dotnet add package Microsoft.EntityFrameworkCore.Sqlite
   dotnet add package Microsoft.EntityFrameworkCore.Design
   ```
   Docs: [`dotnet add package`](https://learn.microsoft.com/dotnet/core/tools/dotnet-add-package).
6. **Verify the versions** (replaces `grep`):
   ```powershell
   Select-String -Path .\RestaurantErp.Api.csproj -Pattern 'PackageReference'
   dotnet list package
   ```
   Both EF Core packages must show `10.x`, matching `dotnet ef --version`. A mismatch is the number one cause of strange migration errors.
7. **Write the model files** in `Models\`: `TransactionType.cs`, `ExpenseCategory.cs`, `Transaction.cs` (sections 2 and 3). Use the namespace `RestaurantErp.Api.Models`.
8. **Write `Data\AppDbContext.cs`** (section 4). Namespace `RestaurantErp.Api.Data`.
9. **Add the connection string** to `appsettings.json` (section 5).
10. **Register the context** in `Program.cs` with `AddDbContext` and `UseSqlite` (section 5).
11. **Build first.** `dotnet build`. Read any error before continuing; a failing build makes `migrations add` fail.
12. **Create the migration and the database** (still inside `api\`):
    ```powershell
    dotnet ef migrations add InitialCreate
    dotnet ef database update
    ```
    Open the generated `Migrations\*_InitialCreate.cs` and compare its `Up` method with the DDL in section 6.
13. **Append the DB ignore rules** (translation of `printf '...' >> ../.gitignore`):
    ```powershell
    Add-Content -Path ..\.gitignore -Value '*.db', '*.db-shm', '*.db-wal', '*.db-journal'
    ```
14. **Confirm the database is ignored:**
    ```powershell
    git check-ignore -v erp.db     # prints the matching rule and path
    git status --short             # must not list erp.db
    ```
15. **Smoke test (temporary code, written by you).** Right after `builder.Build()` in `Program.cs`, write a block that: opens a DI scope with `CreateScope`; gets the context with `GetRequiredService<AppDbContext>`; adds one `Transaction` (an expense "Beef purchase", `150.00m`, category `CostOfGoodsSold`, today's date via `DateOnly.FromDateTime(DateTime.Today)`); calls `SaveChanges`; then loops over `db.Transactions` and prints every row (format the date with `{t.Date:yyyy-MM-dd}`). Mark it clearly with a comment so you remember to delete it.

    | Element | Why | Docs |
    | --- | --- | --- |
    | `CreateScope` | The `DbContext` is *scoped* (one per request). Outside a request you must create a scope by hand. | [Service lifetimes](https://learn.microsoft.com/aspnet/core/fundamentals/dependency-injection#service-lifetimes), [`CreateScope`](https://learn.microsoft.com/dotnet/api/microsoft.extensions.dependencyinjection.serviceproviderserviceextensions.createscope) |
    | `GetRequiredService<T>` | Asks the DI container for an instance; throws if it is not registered (unlike `GetService`). | [`GetRequiredService`](https://learn.microsoft.com/dotnet/api/microsoft.extensions.dependencyinjection.serviceproviderserviceextensions.getrequiredservice) |
    | `Add` / `SaveChanges` | `Add` marks the entity as pending; `SaveChanges` writes the `INSERT` in one DB transaction and fills `Id`. | [Basic saving](https://learn.microsoft.com/ef/core/saving/basic) |
    | `DateOnly.FromDateTime` | Converts today's `DateTime` to a date without time. | [DateOnly](https://learn.microsoft.com/dotnet/standard/datetime/how-to-use-dateonly-timeonly) |

16. **Run it twice.** `dotnet run`, stop with `Ctrl+C`, run again. First run: 1 row. Second run: 2 rows. The value of `Amount` may print as `150.0`; that is the TEXT storage from section 7.
17. **Inspect the file.** Open `erp.db` in DB Browser for SQLite, or use the CLI if installed: `sqlite3 erp.db ".schema Transactions"` and `sqlite3 erp.db "SELECT * FROM Transactions;"`. Compare with section 6. Try `EXPLAIN QUERY PLAN` on a date-range query.
18. **Remove the smoke-test block**, then `dotnet build` again.
19. **Check for leftover Portuguese identifiers** (replaces `grep -r`):
    ```powershell
    Get-ChildItem -Path . -Recurse -Filter *.cs |
      Where-Object { $_.FullName -notmatch '\\(bin|obj)\\' } |
      Select-String -Pattern 'Lancamento|Entrada|Saida|Valor|Descricao|Categoria|Custo|Despesa|Fechamento'
    ```
    No output means clean.
20. **Commit** from the repository root (only when you are ready; Conventional Commits):
    ```powershell
    Set-Location ..
    git add .
    git commit -m "feat(api): add transaction model with SQLite persistence"
    ```
    Before committing, `git status --short` must list `api/Migrations/` but no `.db*` files.

### Pitfalls

- `dotnet ef` not found: PATH, or you did not open a new terminal after installing it.
- "Build failed" during `migrations add`: run `dotnet build` and read the first error.
- Changed the model but forgot a migration: runtime error "no such column".
- Running `dotnet ef` or `dotnet run` from a different folder: a second `erp.db` appears there; the one you inspect is not the one the app uses.
- Package version drift: re-check step 6 after any `dotnet add package`.

---

## 12. Locale and currency policy

- **Storage:** plain `decimal Amount`, no currency column. Single currency (BRL), documented assumption.
- **API JSON:** numbers use `.` as the decimal separator; dates are ISO-8601 (`"2026-09-16"`); enum values are the PascalCase member names; property names are `camelCase`.
- **Server-side formatting** (CSV, logs): always `CultureInfo.InvariantCulture`, so the output never depends on the machine's regional settings.
- **UI:** use Angular's `currency: 'BRL'` pipe on the default `en-US` locale, which renders `R$1,234.56`. The guide's optional pt-BR locale step (5.9) is skipped unless the policy changes (open decision 3).
- **CSV (US rules):** comma separator, `.` decimal, UTF-8 with BOM (so Excel reads accents), header such as `Item,Amount (BRL)`. See open decision 2.

---

## 13. Open decisions

Record the outcome here (and in `CLAUDE.md`) once decided.

| # | Decision | Options | Decide before |
| --- | --- | --- | --- |
| 1 | **Classifier keyword language** | Keywords must match the language you type descriptions in. English keywords (`beef`, `rent`, `electricity`) vs Portuguese keywords (`carne`, `aluguel`, `luz`), or both. Code and comments stay English either way; the keyword list is user data. | Stage 3 |
| 2 | **CSV format** | US rules (comma separator, `.` decimal) as in section 12, vs the guide's Brazilian style (`;` separator, decimal comma). A pt-BR Excel opens a comma-separated CSV as a single column. Either accept that, or keep `;` and the decimal comma as a deliberate exception for the end user. | Stage 4 |
| 3 | **UI locale** | Angular default `en-US` (`R$1,234.56`) vs pt-BR number format (`R$ 1.234,56`, the guide's optional step 5.9). | Stage 5 |

Status: all three are **open**.
