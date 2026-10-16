# Azure Pipelines security stage

> **Placeholders:** file paths, the solution path and the branch names are illustrations. Who owns the pipeline, and where its YAML lives, are project facts; ask if they are not recorded.

The repo is on GitHub, and CI/CD runs in Azure Pipelines. This stage runs on every pull request. Once a repo admin marks it as a required check, a failing stage blocks the merge.

## Template (for example `eng/pipelines/templates/security-gates.yml`)
```yaml
parameters:
  - name: solution
    type: string
  - name: buildConfiguration
    type: string
    default: Release

jobs:
  - job: security_gates
    displayName: Security gates
    steps:
      - task: UseDotNet@2
        displayName: Install .NET 10 SDK
        inputs:
          packageType: sdk
          version: 10.0.x

      - script: dotnet restore "${{ parameters.solution }}"
        displayName: Restore (NuGet audit fails on vulnerable packages)

      - script: dotnet build "${{ parameters.solution }}" --configuration ${{ parameters.buildConfiguration }} --no-restore
        displayName: Build (security analyzers are errors)

      - script: >-
          dotnet test "${{ parameters.solution }}"
          --configuration ${{ parameters.buildConfiguration }} --no-build
          --logger trx --results-directory "$(Agent.TempDirectory)/TestResults"
        displayName: Test (guardrail tests and forbidden-pattern scan)

      - task: PublishTestResults@2
        displayName: Publish test results
        condition: succeededOrFailed()
        inputs:
          testResultsFormat: VSTest
          testResultsFiles: '**/*.trx'
          searchFolder: $(Agent.TempDirectory)/TestResults
          failTaskOnFailedTests: true

      - pwsh: |
          $json = (dotnet list "${{ parameters.solution }}" package --vulnerable --include-transitive --format json) -join "`n"
          if ($LASTEXITCODE -ne 0) { Write-Host "##vso[task.logissue type=error]dotnet list package failed"; exit 1 }
          $report = $json | ConvertFrom-Json
          $findings = @(
            $report.projects | ForEach-Object { $_.frameworks } |
              ForEach-Object { @($_.topLevelPackages) + @($_.transitivePackages) } |
              Where-Object { $_ -and $_.vulnerabilities }
          )
          if ($findings.Count -gt 0) {
            $findings | ForEach-Object { Write-Host "##vso[task.logissue type=error]Vulnerable package: $($_.id) $($_.resolvedVersion)" }
            exit 1
          }
          Write-Host "No vulnerable packages found."
        displayName: Vulnerable package check (direct and transitive)

      # Secret scanning: add the step for the scanner chosen at setup, pinned to a version,
      # unless GitHub secret scanning with push protection is used instead (github-protection.md).
```
Every step fails closed: if a check cannot run, the job fails rather than passing silently.

## Wiring it into the main pipeline (only with the pipeline owner's approval)
```yaml
pr:
  branches:
    include: [ main ]

stages:
  - stage: SecurityGates
    jobs:
      - template: eng/pipelines/templates/security-gates.yml
        parameters:
          solution: <path to the .sln or .slnx file>
```
- Merge with the existing `trigger`, `pr` and `stages` sections; never replace them.
- The pipeline must be connected to the GitHub repo (Azure Pipelines GitHub App) so its result appears as a check on pull requests. The repo admin then marks that check as required (github-protection.md).

## When someone else owns the pipeline
Do not edit it. Write a cross-repo change request (`api-security-baseline/references/change-request.md`) with:
- Target: the pipeline and its owner.
- Requested change: add the stage above for pull requests to the protected branches, and report it as a GitHub check.
- Why: security checks must block merges from day one (project-facts).
- Proof: a pull request with a deliberately failing guardrail test is blocked.

Until the stage runs, the guardrail tests still run in every local `dotnet test`. Report the CI gate as NOT RUN, not PASS.
