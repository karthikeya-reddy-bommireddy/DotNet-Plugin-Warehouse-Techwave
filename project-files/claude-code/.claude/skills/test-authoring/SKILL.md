---
name: test-authoring
description: Use when the brief asks the dotnet-tester to write or update unit and integration tests for a module from its acceptance criteria and design, and to run them with run_tests. Every acceptance criterion gets at least one test that names it. Do not use to change production code, weaken or delete existing tests, or test against real shared or production systems.
license: Proprietary
metadata:
  version: 1.4.0
  agent: dotnet-tester
  contract: test-authoring/1
disable-model-invocation: true
---
# Test authoring

## Inputs
- The acceptance criteria table of the design approved at Gate 2 (AC IDs and test levels) and the requirement
  registry, read with `read_context`.
- The code under test and the test projects inside your scope (only the approved scope's test folders are
  writable); the testing framework and assertion library from the tech profile.

## Procedure
1. For each acceptance criterion, write at least one test at the level the design names. Name the criterion on the
   test so the engine can find it:

   ```csharp
   [Fact]
   [Trait("AC", "AC-001-1")]
   public void ConfirmedBasketWithinTheLimitIsPlaced() { ... }
   ```

   MSTest uses `[TestCategory("AC-001-1")]`, NUnit `[Category("AC-001-1")]`. A trait on the class applies to every
   test in it. Name tests so the behaviour is readable.
2. Cover the edge cases the criterion implies (empty input, boundaries, invalid input, permission denied) only when
   the criterion or design states them.
3. Integration tests use throwaway dependencies with synthetic data only. When the engine provides a SQL Server for
   the test run, its connection string is in the environment variable `AGENTS_TEST_SQLSERVER`; use it, and fall back
   to Testcontainers only when the variable is missing (a developer's local run):

   ```csharp
   var connectionString = Environment.GetEnvironmentVariable("AGENTS_TEST_SQLSERVER")
       ?? await StartLocalContainerAsync();   // Testcontainers, outside the pipeline only
   ```

   When the integration-test database decision (brief `decisions` or tech profile) is `sqlite`, use
   SQLite in-memory through the EF Core SQLite provider instead: one open connection per test class,
   so the database lives as long as the tests, and the schema created from the model. Do not add the
   package yourself; it comes from the decision. SQLite is not SQL Server: when a criterion depends on
   SQL Server behaviour (types, collation, migrations), say so in an open question.

   ```csharp
   var connection = new SqliteConnection("DataSource=:memory:");
   connection.Open();                                   // keep it open: closing it drops the database
   // in WebApplicationFactory<Program>.ConfigureWebHost: replace the DbContext registration with
   services.AddDbContext<<Module>DbContext>(o => o.UseSqlite(connection));
   // then, once per test class: db.Database.EnsureCreated();
   ```

   Never connect to shared or production databases or services, and never use real customer data.
4. Call `run_tests`. Failed tests come back by ID; a failure that reveals a real defect is a correct result: report
   it, the fixer handles it. Fix only mistakes in your own tests, then run `run_tests` again so the last run is on the
   current code.

## Reproducing a failure (a re-planned round)
When the same failure came back or a round stalled, the engine sends you before the fixer, and the context pack
lists `round failures`. Write a focused test that fails for the reported cause (the failing test's scenario or
the finding's case, with the smallest data that shows it), name its acceptance criterion, run `run_tests` and
report `done`: the failing test is the expected result and tells the fixer where the cause is. Never change the
existing tests or production code.

## The engine checks before your report counts
- the last `run_tests` ran on the current code and completed;
- every acceptance criterion the module must prove has a test naming it (all of the BA package on the full path; on
  the small-change path those of the requirements the design note names);
- those tests are among the tests that ran;
- the fake-fix guard finds no deleted, newly skipped or weakened test (fewer assertions) and no suppression
  (`fake fix: ...`).

## Never
- Delete, skip or weaken an existing test (`Skip`, removed asserts, wider tolerances).
- Change production code or anything outside the test folders.
- Assert on implementation details the design does not specify.
- Add test packages that are not in the tech profile.

## Report (report/1)
`outputs`: every test file your turn created, changed or deleted, with `kind: "test"` and requirement IDs (a deleted
file by its path; the engine rejects a report that leaves one out); `summary`: tests per AC and the run's counts,
with the IDs of failing tests. An AC that cannot be tested is an open question with the reason (status `needs_input`).
