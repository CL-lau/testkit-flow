# AI Test Workflow Design

## Overview

`testkit-flow` will provide a complete AI-assisted testing workflow inspired by Superpowers and ECC-style agentic development. It will combine a documentation-first workflow system with an executable CLI so a human tester or AI coding agent can move from requirements to test strategy, test execution, triage, gate decision, reporting, and reusable learning.

The first version will be an AI-ready workflow runner. It will not call an LLM directly. Instead, it will create structured artifacts, detect project test tooling, run configured commands, and preserve evidence so Codex, Claude Code, or another agent harness can fill in analysis and decision documents.

## Goals

- Provide a clear end-to-end AI testing process with explicit stages, inputs, outputs, and exit criteria.
- Ship both human-readable workflow documentation and a runnable CLI.
- Make each testing phase auditable through files under `.testkit-flow/`.
- Support common project ecosystems with automatic discovery of test commands.
- Produce final gate and report artifacts that can be pasted into a PR, release checklist, or test archive.
- Keep the first version harness-native and model-agnostic.

## Non-Goals

- Direct LLM API integration in the first version.
- Replacing project-specific test frameworks.
- Building a full test management system with users, dashboards, or hosted storage.
- Automatically fixing product bugs found during test execution.
- Guaranteeing coverage quality from line coverage alone.

## Architecture

The system has two layers.

### Workflow Layer

The workflow layer contains the written process:

- `docs/workflow.md` explains the full AI testing workflow.
- `docs/phases/` contains one phase document per workflow stage.
- `agents/` contains prompt files for specialized AI testing roles.
- `templates/` contains Markdown and JSON templates for phase outputs.
- `examples/` demonstrates a complete run.

This layer is optimized for AI agents and human operators. It makes expectations explicit enough that an agent can follow the process without inventing missing steps.

### Execution Layer

The execution layer contains the CLI and reusable logic:

- `src/cli/` handles command parsing and user-facing commands.
- `src/core/` defines workflow run state and artifact paths.
- `src/detectors/` identifies languages, package managers, test frameworks, and likely commands.
- `src/runners/` executes configured commands and captures structured results.
- `src/reporters/` renders final Markdown and JSON reports.

The CLI writes all runtime artifacts under `.testkit-flow/` in the target project.

## Runtime Workspace

A target project initialized with `testkit-flow init` will get:

```text
.testkit-flow/
  config.json
  runs/
    <run-id>/
      intake.md
      acceptance-criteria.json
      discovery.json
      project-map.md
      risk-map.md
      risk-register.json
      test-strategy.md
      test-cases.json
      test-cases.md
      generation-report.md
      execution.json
      logs/
      artifacts/
      triage.md
      failures.json
      coverage-gap.md
      gap-matrix.json
      gate.md
      gate.json
      report.md
      report.json
      learning.md
  rules/
    testing.md
```

Each run is immutable by default after report generation. Follow-up test cycles should create a new run ID so evidence stays traceable.

## CLI Commands

The initial CLI command set will mirror the workflow stages:

```bash
testkit-flow init
testkit-flow intake
testkit-flow discover
testkit-flow risk
testkit-flow strategy
testkit-flow cases
testkit-flow generate
testkit-flow run
testkit-flow triage
testkit-flow coverage
testkit-flow gate
testkit-flow report
testkit-flow learn
```

### Command Responsibilities

- `init`: create `.testkit-flow/`, default config, and the first run directory.
- `intake`: scaffold requirement intake and acceptance criteria files.
- `discover`: inspect the project and write detected tooling, commands, and project map.
- `risk`: scaffold a risk register from intake, discovery, and optional diff input.
- `strategy`: scaffold a strategy document that maps risk to test levels.
- `cases`: scaffold structured test cases from acceptance criteria and risk register.
- `generate`: create a generation report for test code or manual automation tasks.
- `run`: execute configured test commands and capture logs, exit codes, and durations.
- `triage`: scaffold failure analysis artifacts from execution output.
- `coverage`: scaffold coverage-gap analysis artifacts from coverage reports and cases.
- `gate`: calculate or scaffold the final quality gate decision.
- `report`: aggregate all phase artifacts into Markdown and JSON reports.
- `learn`: scaffold reusable project testing rules from the completed run.

## Workflow Phases

Each phase is defined by input, AI actions, human confirmation, output artifacts, and exit criteria.

### 1. Init

Input:
- Target project path.

CLI actions:
- Create `.testkit-flow/`.
- Create `config.json`.
- Create a run directory.
- Copy or render starter templates.

Outputs:
- `.testkit-flow/config.json`
- `.testkit-flow/runs/<run-id>/`

Exit criteria:
- The config parses successfully.
- The active run directory exists.
- Required subdirectories for logs and artifacts exist.

### 2. Intake

Input:
- Feature request, PRD, issue, commit diff, user story, or free-form test objective.

AI actions:
- Extract functional claims.
- Extract acceptance criteria.
- Identify implicit assumptions.
- Identify open questions.
- Convert requirements into testable assertions.

Human confirmation:
- Confirm the testing target and acceptance criteria.

Outputs:
- `intake.md`
- `acceptance-criteria.json`

Exit criteria:
- Every requirement has at least one testable assertion.
- Open questions are explicitly marked as risks or blockers.

### 3. Discovery

Input:
- Target project directory.

CLI actions:
- Detect language and package manager.
- Detect test framework.
- Detect likely unit, integration, E2E, lint, type-check, build, and coverage commands.
- Identify important project directories and existing test locations.

AI actions:
- Summarize project testing conventions.
- Note missing or ambiguous commands.

Outputs:
- `discovery.json`
- `project-map.md`

Exit criteria:
- At least one runnable validation command is known, or a blocker is recorded.
- Existing test locations and major source locations are documented.

### 4. Risk Mapping

Input:
- Intake artifacts.
- Discovery artifacts.
- Optional diff or changed-file list.

AI actions:
- Map functional risk.
- Map data risk.
- Map permission and role risk.
- Map integration risk.
- Map state and workflow risk.
- Map performance and scalability risk.
- Map security and privacy risk.
- Map regression risk.

Outputs:
- `risk-map.md`
- `risk-register.json`

Exit criteria:
- Every high-risk item has an owner stage in the test strategy.
- Risk severity and rationale are recorded.

### 5. Strategy

Input:
- Risk register.
- Project testing capabilities.
- Acceptance criteria.

AI actions:
- Choose test levels: unit, integration, contract, E2E, snapshot, AI eval, and manual exploratory.
- Define what each test level should prove.
- Define command sequence and evidence expectations.
- Define what will be considered out of scope for the current run.

Outputs:
- `test-strategy.md`

Exit criteria:
- Every high and medium risk item maps to at least one verification method.
- The strategy identifies what must pass before release.

### 6. Case Design

Input:
- Acceptance criteria.
- Risk register.
- Test strategy.

AI actions:
- Generate test cases with priority, preconditions, data, steps, expected result, and automation recommendation.
- Cover happy paths, boundary cases, error paths, permission paths, state transitions, and regression paths.
- Identify cases that require manual exploration.

Outputs:
- `test-cases.json`
- `test-cases.md`

Exit criteria:
- Each acceptance criterion maps to one or more test cases.
- Each high-risk item maps to one or more test cases.
- Test cases separate behavior expectations from implementation details.

### 7. Test Generation

Input:
- Test cases.
- Project conventions.
- Existing test patterns.

AI actions:
- Propose or generate test files following project conventions.
- For new feature work, follow test-first behavior where applicable.
- Identify test data and fixtures needed.
- Mark cases that should remain manual.

Outputs:
- Test file drafts or implementation notes.
- `generation-report.md`

Exit criteria:
- Generated tests are discoverable by the target test framework, or manual implementation steps are explicit.
- The report records which cases were automated, skipped, or deferred.

### 8. Execution

Input:
- Test strategy.
- Configured commands.
- Generated or existing tests.

CLI actions:
- Run configured validation commands.
- Capture stdout, stderr, exit code, start time, end time, and duration.
- Store logs and artifact paths.

Outputs:
- `execution.json`
- `logs/`
- `artifacts/`

Exit criteria:
- Every configured command has a structured result.
- Failures preserve enough evidence for triage.

### 9. Failure Triage

Input:
- Execution output.
- Logs and artifacts.
- Test cases and strategy.

AI actions:
- Classify each failure as product defect, test defect, environment issue, data issue, dependency issue, flaky behavior, or unknown.
- Identify likely root cause and supporting evidence.
- Recommend next action.

Outputs:
- `triage.md`
- `failures.json`

Exit criteria:
- Every failure has a category, evidence, and next step.
- Unknown failures are explicitly marked with required investigation.

### 10. Coverage Gap

Input:
- Coverage reports when available.
- Test case matrix.
- Risk register.

AI actions:
- Identify untested behavior.
- Identify missing assertions.
- Identify missing error-path and permission-path tests.
- Compare line coverage with behavioral coverage.

Outputs:
- `coverage-gap.md`
- `gap-matrix.json`

Exit criteria:
- Critical risks cannot be closed by line coverage alone.
- Remaining gaps have severity and recommended remediation.

### 11. Regression Gate

Input:
- Execution results.
- Failure triage.
- Coverage gap analysis.
- Risk register.

AI actions:
- Recommend a gate decision: `PASS`, `PASS_WITH_RISK`, `FAIL`, or `BLOCKED`.
- Summarize blocking failures.
- Summarize accepted risks.
- Summarize required follow-up.

CLI actions:
- Validate that required artifacts exist.
- Render `gate.json` from available evidence and decision fields.

Outputs:
- `gate.md`
- `gate.json`

Exit criteria:
- The gate decision includes evidence.
- `FAIL` and `BLOCKED` include concrete unblock steps.
- `PASS_WITH_RISK` includes explicit risk acceptance language.

### 12. Report

Input:
- All phase artifacts.

CLI actions:
- Aggregate phase outputs into final Markdown and JSON.
- Include command results, failures, risks, coverage gaps, and gate decision.

AI actions:
- Summarize results for PR, release, or test archive audiences.

Outputs:
- `report.md`
- `report.json`

Exit criteria:
- The report can be read without opening raw logs.
- Raw evidence remains linked by path.
- Gate decision and rationale are prominent.

### 13. Learning

Input:
- Final report.
- Triage output.
- Coverage gaps.
- Project testing conventions.

AI actions:
- Extract reusable test rules.
- Record common failure modes.
- Record recommended fixtures, selectors, factories, and command patterns.
- Propose improvements to future testing prompts.

Outputs:
- `learning.md`
- `.testkit-flow/rules/testing.md`

Exit criteria:
- At least one reusable project-specific testing rule is recorded, or the run explicitly states that no new learning was found.
- Future runs can reference the learning artifact.

## Agent Prompts

The first version will include prompt files for these roles:

- `test-planner.md`: turns intake and discovery into test strategy.
- `test-case-designer.md`: turns criteria and risks into structured test cases.
- `test-runner.md`: interprets execution setup and evidence collection requirements.
- `failure-triager.md`: classifies failures and recommends next actions.
- `coverage-analyst.md`: compares coverage output against behavioral expectations.
- `regression-gatekeeper.md`: makes gate recommendations from evidence.

Each prompt will define role, inputs, process, output format, and safety constraints.

## Detection Rules

Initial discovery will support:

- Node package managers: npm, pnpm, yarn, bun.
- Node test signals: Jest, Vitest, Playwright.
- Python signals: `pyproject.toml`, `requirements.txt`, pytest.
- Go signals: `go.mod`, `go test`.
- Rust signals: `Cargo.toml`, `cargo test`.

Discovery will prefer explicit project scripts over inferred commands. When multiple commands exist, it will record all candidates and mark likely defaults.

## Reporting Model

Reports will preserve both machine-readable and human-readable output.

Important JSON objects:

- Acceptance criterion.
- Risk item.
- Test case.
- Command execution result.
- Failure classification.
- Coverage gap.
- Gate decision.

Important Markdown documents:

- Intake.
- Project map.
- Risk map.
- Test strategy.
- Test cases.
- Triage.
- Coverage gap.
- Gate.
- Final report.
- Learning.

## Acceptance Criteria

Version 1 is complete when:

- `testkit-flow init` creates the `.testkit-flow/` workspace.
- `testkit-flow discover` detects common Node, Python, Go, and Rust test commands.
- `testkit-flow run` executes configured commands and saves structured results.
- `testkit-flow report` generates final Markdown and JSON reports from run artifacts.
- Templates exist for all 13 workflow phases.
- Agent prompt files exist for the core testing roles.
- Documentation explains how an AI agent should move through the workflow.
- Automated tests cover CLI initialization, discovery, execution result capture, and report rendering.

## Implementation Notes

The preferred implementation is a small CLI with conservative dependencies. The implementation language can be chosen during planning based on repository conventions, but the first version should optimize for:

- Simple local installation.
- Cross-platform command execution.
- JSON artifact stability.
- Easy testability.
- Clear file layout.

The CLI should avoid hidden global state. All run state should live under the target project's `.testkit-flow/` directory.

## Open Decisions Resolved

- The first version will include both documentation and CLI support.
- The first version will not directly call LLM APIs.
- Each workflow phase will have explicit inputs, actions, outputs, and exit criteria.
- Test evidence will be stored as local files, not in a hosted service.
- Gate decisions will be evidence-based and represented in both Markdown and JSON.
