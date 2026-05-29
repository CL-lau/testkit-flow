# AI Test Workflow Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the first working version of `testkit-flow`: a documentation-backed, model-agnostic AI testing workflow CLI with phase templates, discovery, command execution, gate/report rendering, and agent prompts.

**Architecture:** Implement a small Node.js ESM CLI with no runtime dependencies. The CLI owns deterministic filesystem operations, project discovery, command execution, and report aggregation; AI agents fill and review the generated Markdown/JSON artifacts.

**Tech Stack:** Node.js 20+, ESM modules, built-in `node:test`, built-in `assert`, built-in `fs`, `path`, `child_process`, and Markdown/JSON artifacts.

---

## File Structure

- Create `package.json`: package metadata, CLI bin, and scripts.
- Create `bin/testkit-flow.js`: executable entrypoint.
- Create `src/cli/main.js`: argument parsing and command dispatch.
- Create `src/core/paths.js`: project workspace and run path helpers.
- Create `src/core/json.js`: JSON read/write helpers.
- Create `src/core/templates.js`: template registry and phase scaffolding.
- Create `src/core/commands.js`: shell command execution helper.
- Create `src/detectors/discover.js`: package manager, language, test framework, and command detection.
- Create `src/reporters/report.js`: final report and gate rendering helpers.
- Create `src/commands/*.js`: one module per CLI command group.
- Create `templates/`: source templates for all 13 workflow phases.
- Create `agents/`: prompt files for testing roles.
- Create `docs/workflow.md`: full operator workflow.
- Create `docs/phases/*.md`: phase-specific docs.
- Modify `README.md`: quickstart and command reference.
- Create `test/*.test.js`: Node test coverage for init, discovery, execution, and report rendering.

## Task 1: Package and CLI Entrypoint

**Files:**
- Create: `package.json`
- Create: `bin/testkit-flow.js`
- Create: `src/cli/main.js`
- Test: `test/cli.test.js`

- [ ] **Step 1: Write the failing CLI help test**

Create `test/cli.test.js`:

```js
import test from 'node:test';
import assert from 'node:assert/strict';
import { spawnSync } from 'node:child_process';
import { fileURLToPath } from 'node:url';
import path from 'node:path';

const root = path.resolve(path.dirname(fileURLToPath(import.meta.url)), '..');
const bin = path.join(root, 'bin', 'testkit-flow.js');

test('prints help when no command is provided', () => {
  const result = spawnSync(process.execPath, [bin], {
    cwd: root,
    encoding: 'utf8'
  });

  assert.equal(result.status, 0);
  assert.match(result.stdout, /Usage: testkit-flow <command>/);
  assert.match(result.stdout, /init/);
  assert.match(result.stdout, /discover/);
  assert.match(result.stdout, /report/);
});

test('returns a useful error for an unknown command', () => {
  const result = spawnSync(process.execPath, [bin, 'unknown-command'], {
    cwd: root,
    encoding: 'utf8'
  });

  assert.equal(result.status, 1);
  assert.match(result.stderr, /Unknown command: unknown-command/);
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run:

```bash
npm test -- test/cli.test.js
```

Expected: command fails because `package.json` or `bin/testkit-flow.js` does not exist.

- [ ] **Step 3: Add package metadata and CLI entrypoint**

Create `package.json`:

```json
{
  "name": "testkit-flow",
  "version": "0.1.0",
  "description": "AI-ready testing workflow CLI and documentation kit.",
  "type": "module",
  "bin": {
    "testkit-flow": "./bin/testkit-flow.js"
  },
  "scripts": {
    "test": "node --test",
    "test:watch": "node --test --watch",
    "check": "node --test"
  },
  "engines": {
    "node": ">=20"
  },
  "license": "MIT"
}
```

Create `bin/testkit-flow.js`:

```js
#!/usr/bin/env node
import { main } from '../src/cli/main.js';

main(process.argv.slice(2)).catch((error) => {
  console.error(error instanceof Error ? error.message : String(error));
  process.exitCode = 1;
});
```

Create `src/cli/main.js`:

```js
const COMMANDS = [
  'init',
  'intake',
  'discover',
  'risk',
  'strategy',
  'cases',
  'generate',
  'run',
  'triage',
  'coverage',
  'gate',
  'report',
  'learn'
];

export function helpText() {
  return [
    'Usage: testkit-flow <command> [options]',
    '',
    'Commands:',
    ...COMMANDS.map((command) => `  ${command}`),
    '',
    'Options:',
    '  --project <path>   Target project directory, defaults to current directory',
    '  --run <id>         Workflow run id, defaults to active run in config',
    '  --help            Show this help'
  ].join('\n');
}

export async function main(argv) {
  const [command] = argv;

  if (!command || command === '--help' || command === '-h') {
    console.log(helpText());
    return;
  }

  if (!COMMANDS.includes(command)) {
    console.error(`Unknown command: ${command}`);
    console.error('');
    console.error(helpText());
    process.exitCode = 1;
    return;
  }

  console.log(`${command} command is not implemented yet.`);
}
```

- [ ] **Step 4: Run the CLI tests to verify green**

Run:

```bash
npm test -- test/cli.test.js
```

Expected: both tests pass.

- [ ] **Step 5: Commit**

```bash
git add package.json bin/testkit-flow.js src/cli/main.js test/cli.test.js
git commit -m "feat: add testkit-flow CLI entrypoint"
```

## Task 2: Workspace Paths and JSON Helpers

**Files:**
- Create: `src/core/paths.js`
- Create: `src/core/json.js`
- Test: `test/core.test.js`

- [ ] **Step 1: Write failing tests for workspace path helpers**

Create `test/core.test.js`:

```js
import test from 'node:test';
import assert from 'node:assert/strict';
import fs from 'node:fs';
import os from 'node:os';
import path from 'node:path';
import { workflowPaths, createRunId } from '../src/core/paths.js';
import { readJsonFile, writeJsonFile } from '../src/core/json.js';

test('workflowPaths returns stable project-local paths', () => {
  const project = path.join(os.tmpdir(), 'testkit-flow-paths');
  const paths = workflowPaths(project, 'run-abc');

  assert.equal(paths.projectRoot, project);
  assert.equal(paths.workspace, path.join(project, '.testkit-flow'));
  assert.equal(paths.config, path.join(project, '.testkit-flow', 'config.json'));
  assert.equal(paths.runDir, path.join(project, '.testkit-flow', 'runs', 'run-abc'));
  assert.equal(paths.logsDir, path.join(project, '.testkit-flow', 'runs', 'run-abc', 'logs'));
  assert.equal(paths.artifactsDir, path.join(project, '.testkit-flow', 'runs', 'run-abc', 'artifacts'));
});

test('createRunId is timestamp based and filesystem friendly', () => {
  const runId = createRunId(new Date('2026-05-30T01:02:03.004Z'));
  assert.equal(runId, '2026-05-30T01-02-03-004Z');
});

test('JSON helpers create parent directories and preserve data', () => {
  const dir = fs.mkdtempSync(path.join(os.tmpdir(), 'testkit-flow-json-'));
  const file = path.join(dir, 'nested', 'data.json');
  const value = { name: 'testkit-flow', commands: ['init', 'run'] };

  writeJsonFile(file, value);

  assert.deepEqual(readJsonFile(file), value);
});
```

- [ ] **Step 2: Run the tests to verify they fail**

Run:

```bash
npm test -- test/core.test.js
```

Expected: fail with missing modules.

- [ ] **Step 3: Implement path and JSON helpers**

Create `src/core/paths.js`:

```js
import path from 'node:path';

export function createRunId(date = new Date()) {
  return date.toISOString().replaceAll(':', '-').replace('.', '-');
}

export function workflowPaths(projectRoot, runId) {
  const workspace = path.join(projectRoot, '.testkit-flow');
  const runsDir = path.join(workspace, 'runs');
  const runDir = path.join(runsDir, runId);

  return {
    projectRoot,
    workspace,
    config: path.join(workspace, 'config.json'),
    runsDir,
    rulesDir: path.join(workspace, 'rules'),
    runDir,
    logsDir: path.join(runDir, 'logs'),
    artifactsDir: path.join(runDir, 'artifacts')
  };
}
```

Create `src/core/json.js`:

```js
import fs from 'node:fs';
import path from 'node:path';

export function readJsonFile(filePath) {
  return JSON.parse(fs.readFileSync(filePath, 'utf8'));
}

export function writeJsonFile(filePath, value) {
  fs.mkdirSync(path.dirname(filePath), { recursive: true });
  fs.writeFileSync(filePath, `${JSON.stringify(value, null, 2)}\n`);
}
```

- [ ] **Step 4: Run the tests to verify green**

Run:

```bash
npm test -- test/core.test.js
```

Expected: all tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/core/paths.js src/core/json.js test/core.test.js
git commit -m "feat: add workflow path and JSON helpers"
```

## Task 3: Init Command

**Files:**
- Modify: `src/cli/main.js`
- Create: `src/commands/init.js`
- Test: `test/init.test.js`

- [ ] **Step 1: Write failing init command tests**

Create `test/init.test.js`:

```js
import test from 'node:test';
import assert from 'node:assert/strict';
import fs from 'node:fs';
import os from 'node:os';
import path from 'node:path';
import { spawnSync } from 'node:child_process';
import { fileURLToPath } from 'node:url';

const root = path.resolve(path.dirname(fileURLToPath(import.meta.url)), '..');
const bin = path.join(root, 'bin', 'testkit-flow.js');

test('init creates workflow workspace, config, and active run', () => {
  const project = fs.mkdtempSync(path.join(os.tmpdir(), 'testkit-flow-init-'));
  const result = spawnSync(process.execPath, [bin, 'init', '--project', project, '--run', 'run-test'], {
    cwd: root,
    encoding: 'utf8'
  });

  assert.equal(result.status, 0, result.stderr);
  assert.match(result.stdout, /Initialized testkit-flow/);

  const configPath = path.join(project, '.testkit-flow', 'config.json');
  const config = JSON.parse(fs.readFileSync(configPath, 'utf8'));

  assert.equal(config.version, 1);
  assert.equal(config.activeRun, 'run-test');
  assert.deepEqual(config.commands, []);
  assert.ok(fs.existsSync(path.join(project, '.testkit-flow', 'runs', 'run-test', 'logs')));
  assert.ok(fs.existsSync(path.join(project, '.testkit-flow', 'runs', 'run-test', 'artifacts')));
  assert.ok(fs.existsSync(path.join(project, '.testkit-flow', 'rules', 'testing.md')));
});
```

- [ ] **Step 2: Run the init test to verify it fails**

Run:

```bash
npm test -- test/init.test.js
```

Expected: fail because `init` still prints the not-implemented message and creates no files.

- [ ] **Step 3: Implement argument parsing and init command**

Replace `src/cli/main.js` with:

```js
import path from 'node:path';
import { initCommand } from '../commands/init.js';

const COMMANDS = [
  'init',
  'intake',
  'discover',
  'risk',
  'strategy',
  'cases',
  'generate',
  'run',
  'triage',
  'coverage',
  'gate',
  'report',
  'learn'
];

export function helpText() {
  return [
    'Usage: testkit-flow <command> [options]',
    '',
    'Commands:',
    ...COMMANDS.map((command) => `  ${command}`),
    '',
    'Options:',
    '  --project <path>   Target project directory, defaults to current directory',
    '  --run <id>         Workflow run id, defaults to active run in config',
    '  --help            Show this help'
  ].join('\n');
}

export function parseArgs(argv) {
  const [command, ...rest] = argv;
  const options = { project: process.cwd(), run: undefined };

  for (let index = 0; index < rest.length; index += 1) {
    const arg = rest[index];
    if (arg === '--project') {
      options.project = path.resolve(rest[index + 1]);
      index += 1;
    } else if (arg === '--run') {
      options.run = rest[index + 1];
      index += 1;
    } else if (arg === '--help' || arg === '-h') {
      options.help = true;
    } else {
      throw new Error(`Unknown option: ${arg}`);
    }
  }

  return { command, options };
}

export async function main(argv) {
  const { command, options } = parseArgs(argv);

  if (!command || command === '--help' || command === '-h' || options.help) {
    console.log(helpText());
    return;
  }

  if (!COMMANDS.includes(command)) {
    console.error(`Unknown command: ${command}`);
    console.error('');
    console.error(helpText());
    process.exitCode = 1;
    return;
  }

  if (command === 'init') {
    await initCommand(options);
    return;
  }

  console.log(`${command} command is not implemented yet.`);
}
```

Create `src/commands/init.js`:

```js
import fs from 'node:fs';
import { createRunId, workflowPaths } from '../core/paths.js';
import { writeJsonFile } from '../core/json.js';

export function initCommand(options) {
  const runId = options.run ?? createRunId();
  const paths = workflowPaths(options.project, runId);

  fs.mkdirSync(paths.logsDir, { recursive: true });
  fs.mkdirSync(paths.artifactsDir, { recursive: true });
  fs.mkdirSync(paths.rulesDir, { recursive: true });

  writeJsonFile(paths.config, {
    version: 1,
    activeRun: runId,
    commands: [],
    createdAt: new Date().toISOString()
  });

  fs.writeFileSync(
    `${paths.rulesDir}/testing.md`,
    [
      '# Project Testing Rules',
      '',
      '- Record durable testing lessons from completed testkit-flow runs.',
      '- Prefer behavioral assertions over implementation-only checks.',
      ''
    ].join('\n')
  );

  console.log(`Initialized testkit-flow at ${paths.workspace}`);
  console.log(`Active run: ${runId}`);
}
```

- [ ] **Step 4: Run CLI and init tests**

Run:

```bash
npm test -- test/cli.test.js test/init.test.js
```

Expected: all tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/cli/main.js src/commands/init.js test/init.test.js
git commit -m "feat: initialize workflow workspace"
```

## Task 4: Phase Templates and Scaffold Commands

**Files:**
- Create: `src/core/templates.js`
- Create: `src/commands/scaffold.js`
- Modify: `src/cli/main.js`
- Create: `templates/*.md`
- Create: `templates/*.json`
- Test: `test/scaffold.test.js`

- [ ] **Step 1: Write failing scaffold tests**

Create `test/scaffold.test.js`:

```js
import test from 'node:test';
import assert from 'node:assert/strict';
import fs from 'node:fs';
import os from 'node:os';
import path from 'node:path';
import { spawnSync } from 'node:child_process';
import { fileURLToPath } from 'node:url';

const root = path.resolve(path.dirname(fileURLToPath(import.meta.url)), '..');
const bin = path.join(root, 'bin', 'testkit-flow.js');

test('intake scaffolds intake and acceptance criteria artifacts', () => {
  const project = fs.mkdtempSync(path.join(os.tmpdir(), 'testkit-flow-scaffold-'));
  spawnSync(process.execPath, [bin, 'init', '--project', project, '--run', 'run-test'], { encoding: 'utf8' });

  const result = spawnSync(process.execPath, [bin, 'intake', '--project', project], {
    cwd: root,
    encoding: 'utf8'
  });

  assert.equal(result.status, 0, result.stderr);
  assert.ok(fs.existsSync(path.join(project, '.testkit-flow', 'runs', 'run-test', 'intake.md')));
  assert.ok(fs.existsSync(path.join(project, '.testkit-flow', 'runs', 'run-test', 'acceptance-criteria.json')));
});

test('all non-execution phase commands scaffold their configured artifacts', () => {
  const project = fs.mkdtempSync(path.join(os.tmpdir(), 'testkit-flow-phases-'));
  spawnSync(process.execPath, [bin, 'init', '--project', project, '--run', 'run-test'], { encoding: 'utf8' });

  for (const command of ['risk', 'strategy', 'cases', 'generate', 'triage', 'coverage', 'learn']) {
    const result = spawnSync(process.execPath, [bin, command, '--project', project], {
      cwd: root,
      encoding: 'utf8'
    });
    assert.equal(result.status, 0, `${command}: ${result.stderr}`);
  }

  const runDir = path.join(project, '.testkit-flow', 'runs', 'run-test');
  for (const file of [
    'risk-map.md',
    'risk-register.json',
    'test-strategy.md',
    'test-cases.json',
    'test-cases.md',
    'generation-report.md',
    'triage.md',
    'failures.json',
    'coverage-gap.md',
    'gap-matrix.json',
    'learning.md'
  ]) {
    assert.ok(fs.existsSync(path.join(runDir, file)), `${file} should exist`);
  }
});
```

- [ ] **Step 2: Run scaffold tests to verify they fail**

Run:

```bash
npm test -- test/scaffold.test.js
```

Expected: fail because scaffold commands are not implemented.

- [ ] **Step 3: Implement template registry and scaffold command**

Create `src/core/templates.js`:

```js
export const PHASE_ARTIFACTS = {
  intake: [
    ['intake.md', '# Intake\n\n## Testing Target\n\nRecord the feature, change, PR, issue, or release under test.\n\n## Acceptance Summary\n\nRecord the observable behavior that must be true.\n\n## Open Questions\n\nRecord questions that affect test scope.\n'],
    ['acceptance-criteria.json', { criteria: [] }]
  ],
  risk: [
    ['risk-map.md', '# Risk Map\n\n## High Risk\n\n## Medium Risk\n\n## Low Risk\n'],
    ['risk-register.json', { risks: [] }]
  ],
  strategy: [
    ['test-strategy.md', '# Test Strategy\n\n## Scope\n\n## Test Levels\n\n## Required Gates\n\n## Out of Scope\n']
  ],
  cases: [
    ['test-cases.json', { cases: [] }],
    ['test-cases.md', '# Test Cases\n\n## Case Matrix\n']
  ],
  generate: [
    ['generation-report.md', '# Test Generation Report\n\n## Automated\n\n## Manual\n\n## Deferred\n']
  ],
  triage: [
    ['triage.md', '# Failure Triage\n\n## Summary\n\n## Failure Analysis\n'],
    ['failures.json', { failures: [] }]
  ],
  coverage: [
    ['coverage-gap.md', '# Coverage Gap Analysis\n\n## Behavioral Gaps\n\n## Coverage Evidence\n'],
    ['gap-matrix.json', { gaps: [] }]
  ],
  learn: [
    ['learning.md', '# Testing Learning\n\n## Reusable Rules\n\n## Failure Patterns\n\n## Future Improvements\n']
  ]
};

export function artifactContent(content) {
  if (typeof content === 'string') {
    return content.endsWith('\n') ? content : `${content}\n`;
  }
  return `${JSON.stringify(content, null, 2)}\n`;
}
```

Create `src/commands/scaffold.js`:

```js
import fs from 'node:fs';
import path from 'node:path';
import { readJsonFile } from '../core/json.js';
import { workflowPaths } from '../core/paths.js';
import { artifactContent, PHASE_ARTIFACTS } from '../core/templates.js';

export function activeRunId(projectRoot, explicitRun) {
  if (explicitRun) {
    return explicitRun;
  }
  const configPath = path.join(projectRoot, '.testkit-flow', 'config.json');
  return readJsonFile(configPath).activeRun;
}

export function scaffoldPhaseCommand(phase, options) {
  const runId = activeRunId(options.project, options.run);
  const paths = workflowPaths(options.project, runId);
  const artifacts = PHASE_ARTIFACTS[phase];

  if (!artifacts) {
    throw new Error(`No scaffold artifacts configured for phase: ${phase}`);
  }

  fs.mkdirSync(paths.runDir, { recursive: true });

  for (const [fileName, content] of artifacts) {
    const filePath = path.join(paths.runDir, fileName);
    if (!fs.existsSync(filePath)) {
      fs.writeFileSync(filePath, artifactContent(content));
    }
  }

  console.log(`Scaffolded ${phase} artifacts in ${paths.runDir}`);
}
```

Modify `src/cli/main.js` so the import block and dispatch include scaffolding:

```js
import path from 'node:path';
import { initCommand } from '../commands/init.js';
import { scaffoldPhaseCommand } from '../commands/scaffold.js';
```

Add before the not-implemented fallback:

```js
  if (['intake', 'risk', 'strategy', 'cases', 'generate', 'triage', 'coverage', 'learn'].includes(command)) {
    scaffoldPhaseCommand(command, options);
    return;
  }
```

- [ ] **Step 4: Run scaffold tests and existing tests**

Run:

```bash
npm test -- test/cli.test.js test/init.test.js test/scaffold.test.js
```

Expected: all tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/core/templates.js src/commands/scaffold.js src/cli/main.js test/scaffold.test.js
git commit -m "feat: scaffold workflow phase artifacts"
```

## Task 5: Discovery Command

**Files:**
- Create: `src/detectors/discover.js`
- Create: `src/commands/discover.js`
- Modify: `src/cli/main.js`
- Test: `test/discover.test.js`

- [ ] **Step 1: Write failing discovery tests**

Create `test/discover.test.js`:

```js
import test from 'node:test';
import assert from 'node:assert/strict';
import fs from 'node:fs';
import os from 'node:os';
import path from 'node:path';
import { spawnSync } from 'node:child_process';
import { fileURLToPath } from 'node:url';

const root = path.resolve(path.dirname(fileURLToPath(import.meta.url)), '..');
const bin = path.join(root, 'bin', 'testkit-flow.js');

test('discover detects npm scripts and JavaScript test framework signals', () => {
  const project = fs.mkdtempSync(path.join(os.tmpdir(), 'testkit-flow-discover-node-'));
  fs.writeFileSync(path.join(project, 'package-lock.json'), '{}\n');
  fs.writeFileSync(path.join(project, 'vitest.config.js'), 'export default {};\n');
  fs.writeFileSync(path.join(project, 'package.json'), JSON.stringify({
    scripts: {
      test: 'vitest run',
      coverage: 'vitest run --coverage',
      build: 'vite build'
    },
    devDependencies: {
      vitest: '^1.0.0'
    }
  }, null, 2));

  spawnSync(process.execPath, [bin, 'init', '--project', project, '--run', 'run-test'], { encoding: 'utf8' });
  const result = spawnSync(process.execPath, [bin, 'discover', '--project', project], { cwd: root, encoding: 'utf8' });

  assert.equal(result.status, 0, result.stderr);

  const discovery = JSON.parse(fs.readFileSync(path.join(project, '.testkit-flow', 'runs', 'run-test', 'discovery.json'), 'utf8'));
  assert.equal(discovery.packageManager, 'npm');
  assert.deepEqual(discovery.languages, ['javascript']);
  assert.ok(discovery.testFrameworks.includes('vitest'));
  assert.ok(discovery.commands.some((command) => command.name === 'test' && command.command === 'npm run test'));
});

test('discover detects Python pytest projects', () => {
  const project = fs.mkdtempSync(path.join(os.tmpdir(), 'testkit-flow-discover-python-'));
  fs.writeFileSync(path.join(project, 'pyproject.toml'), '[tool.pytest.ini_options]\ntestpaths = ["tests"]\n');

  spawnSync(process.execPath, [bin, 'init', '--project', project, '--run', 'run-test'], { encoding: 'utf8' });
  const result = spawnSync(process.execPath, [bin, 'discover', '--project', project], { cwd: root, encoding: 'utf8' });

  assert.equal(result.status, 0, result.stderr);

  const discovery = JSON.parse(fs.readFileSync(path.join(project, '.testkit-flow', 'runs', 'run-test', 'discovery.json'), 'utf8'));
  assert.deepEqual(discovery.languages, ['python']);
  assert.ok(discovery.testFrameworks.includes('pytest'));
  assert.ok(discovery.commands.some((command) => command.command === 'python -m pytest'));
});
```

- [ ] **Step 2: Run discovery tests to verify they fail**

Run:

```bash
npm test -- test/discover.test.js
```

Expected: fail because `discover` is not implemented.

- [ ] **Step 3: Implement detector and command**

Create `src/detectors/discover.js`:

```js
import fs from 'node:fs';
import path from 'node:path';

function exists(projectRoot, fileName) {
  return fs.existsSync(path.join(projectRoot, fileName));
}

function readPackageJson(projectRoot) {
  const filePath = path.join(projectRoot, 'package.json');
  if (!fs.existsSync(filePath)) {
    return undefined;
  }
  return JSON.parse(fs.readFileSync(filePath, 'utf8'));
}

export function detectPackageManager(projectRoot) {
  if (exists(projectRoot, 'pnpm-lock.yaml')) return 'pnpm';
  if (exists(projectRoot, 'yarn.lock')) return 'yarn';
  if (exists(projectRoot, 'bun.lockb')) return 'bun';
  if (exists(projectRoot, 'package-lock.json')) return 'npm';
  if (exists(projectRoot, 'package.json')) return 'npm';
  if (exists(projectRoot, 'go.mod')) return 'go';
  if (exists(projectRoot, 'Cargo.toml')) return 'cargo';
  if (exists(projectRoot, 'pyproject.toml') || exists(projectRoot, 'requirements.txt')) return 'python';
  return 'unknown';
}

export function discoverProject(projectRoot) {
  const packageJson = readPackageJson(projectRoot);
  const packageManager = detectPackageManager(projectRoot);
  const languages = new Set();
  const testFrameworks = new Set();
  const commands = [];

  if (packageJson) {
    languages.add('javascript');
    const runner = packageManager === 'pnpm' ? 'pnpm' : packageManager === 'yarn' ? 'yarn' : packageManager === 'bun' ? 'bun' : 'npm run';
    for (const [name] of Object.entries(packageJson.scripts ?? {})) {
      const command = runner === 'yarn' || runner === 'bun' ? `${runner} ${name}` : `${runner} ${name}`;
      commands.push({ name, command, source: 'package.json scripts' });
    }
    const deps = { ...(packageJson.dependencies ?? {}), ...(packageJson.devDependencies ?? {}) };
    if (deps.vitest || exists(projectRoot, 'vitest.config.js') || exists(projectRoot, 'vitest.config.ts')) testFrameworks.add('vitest');
    if (deps.jest || exists(projectRoot, 'jest.config.js') || exists(projectRoot, 'jest.config.ts')) testFrameworks.add('jest');
    if (deps['@playwright/test'] || exists(projectRoot, 'playwright.config.js') || exists(projectRoot, 'playwright.config.ts')) testFrameworks.add('playwright');
  }

  if (exists(projectRoot, 'pyproject.toml') || exists(projectRoot, 'requirements.txt')) {
    languages.add('python');
    testFrameworks.add('pytest');
    commands.push({ name: 'test', command: 'python -m pytest', source: 'python project signal' });
  }

  if (exists(projectRoot, 'go.mod')) {
    languages.add('go');
    commands.push({ name: 'test', command: 'go test ./...', source: 'go.mod' });
  }

  if (exists(projectRoot, 'Cargo.toml')) {
    languages.add('rust');
    commands.push({ name: 'test', command: 'cargo test', source: 'Cargo.toml' });
  }

  return {
    packageManager,
    languages: [...languages].sort(),
    testFrameworks: [...testFrameworks].sort(),
    commands,
    sourceDirectories: ['src', 'lib', 'app'].filter((dir) => fs.existsSync(path.join(projectRoot, dir))),
    testDirectories: ['test', 'tests', '__tests__', 'spec'].filter((dir) => fs.existsSync(path.join(projectRoot, dir)))
  };
}
```

Create `src/commands/discover.js`:

```js
import fs from 'node:fs';
import path from 'node:path';
import { workflowPaths } from '../core/paths.js';
import { writeJsonFile } from '../core/json.js';
import { activeRunId } from './scaffold.js';
import { discoverProject } from '../detectors/discover.js';

export function discoverCommand(options) {
  const runId = activeRunId(options.project, options.run);
  const paths = workflowPaths(options.project, runId);
  const discovery = discoverProject(options.project);

  fs.mkdirSync(paths.runDir, { recursive: true });
  writeJsonFile(path.join(paths.runDir, 'discovery.json'), discovery);
  fs.writeFileSync(path.join(paths.runDir, 'project-map.md'), renderProjectMap(discovery));

  console.log(`Wrote discovery artifacts for ${options.project}`);
}

function renderProjectMap(discovery) {
  return [
    '# Project Map',
    '',
    `Package manager: ${discovery.packageManager}`,
    `Languages: ${discovery.languages.join(', ') || 'none detected'}`,
    `Test frameworks: ${discovery.testFrameworks.join(', ') || 'none detected'}`,
    '',
    '## Commands',
    '',
    ...discovery.commands.map((command) => `- ${command.name}: \`${command.command}\` (${command.source})`),
    ''
  ].join('\n');
}
```

Modify `src/cli/main.js`:

```js
import { discoverCommand } from '../commands/discover.js';
```

Add before scaffold dispatch:

```js
  if (command === 'discover') {
    discoverCommand(options);
    return;
  }
```

- [ ] **Step 4: Run discovery and existing tests**

Run:

```bash
npm test -- test/discover.test.js test/scaffold.test.js test/init.test.js
```

Expected: all tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/detectors/discover.js src/commands/discover.js src/cli/main.js test/discover.test.js
git commit -m "feat: discover project test tooling"
```

## Task 6: Run Command and Execution Capture

**Files:**
- Create: `src/core/commands.js`
- Create: `src/commands/run.js`
- Modify: `src/cli/main.js`
- Test: `test/run.test.js`

- [ ] **Step 1: Write failing execution tests**

Create `test/run.test.js`:

```js
import test from 'node:test';
import assert from 'node:assert/strict';
import fs from 'node:fs';
import os from 'node:os';
import path from 'node:path';
import { spawnSync } from 'node:child_process';
import { fileURLToPath } from 'node:url';

const root = path.resolve(path.dirname(fileURLToPath(import.meta.url)), '..');
const bin = path.join(root, 'bin', 'testkit-flow.js');

test('run executes configured commands and captures structured results', () => {
  const project = fs.mkdtempSync(path.join(os.tmpdir(), 'testkit-flow-run-'));
  spawnSync(process.execPath, [bin, 'init', '--project', project, '--run', 'run-test'], { encoding: 'utf8' });

  const configPath = path.join(project, '.testkit-flow', 'config.json');
  const config = JSON.parse(fs.readFileSync(configPath, 'utf8'));
  config.commands = [
    { name: 'pass', command: `${process.execPath} -e "console.log('pass output')"` },
    { name: 'fail', command: `${process.execPath} -e "console.error('fail output'); process.exit(7)"` }
  ];
  fs.writeFileSync(configPath, `${JSON.stringify(config, null, 2)}\n`);

  const result = spawnSync(process.execPath, [bin, 'run', '--project', project], {
    cwd: root,
    encoding: 'utf8'
  });

  assert.equal(result.status, 1);
  assert.match(result.stdout, /Executed 2 command/);

  const execution = JSON.parse(fs.readFileSync(path.join(project, '.testkit-flow', 'runs', 'run-test', 'execution.json'), 'utf8'));
  assert.equal(execution.results.length, 2);
  assert.equal(execution.results[0].status, 'passed');
  assert.equal(execution.results[1].status, 'failed');
  assert.equal(execution.results[1].exitCode, 7);
  assert.ok(fs.existsSync(path.join(project, '.testkit-flow', 'runs', 'run-test', 'logs', 'pass.stdout.log')));
  assert.ok(fs.existsSync(path.join(project, '.testkit-flow', 'runs', 'run-test', 'logs', 'fail.stderr.log')));
});
```

- [ ] **Step 2: Run execution test to verify it fails**

Run:

```bash
npm test -- test/run.test.js
```

Expected: fail because `run` is not implemented.

- [ ] **Step 3: Implement command execution**

Create `src/core/commands.js`:

```js
import { spawnSync } from 'node:child_process';

export function executeShellCommand(command, cwd) {
  const startedAt = new Date();
  const result = spawnSync(command, {
    cwd,
    shell: true,
    encoding: 'utf8'
  });
  const finishedAt = new Date();

  return {
    command,
    exitCode: result.status ?? 1,
    status: result.status === 0 ? 'passed' : 'failed',
    startedAt: startedAt.toISOString(),
    finishedAt: finishedAt.toISOString(),
    durationMs: finishedAt.getTime() - startedAt.getTime(),
    stdout: result.stdout ?? '',
    stderr: result.stderr ?? ''
  };
}
```

Create `src/commands/run.js`:

```js
import fs from 'node:fs';
import path from 'node:path';
import { executeShellCommand } from '../core/commands.js';
import { readJsonFile, writeJsonFile } from '../core/json.js';
import { workflowPaths } from '../core/paths.js';
import { activeRunId } from './scaffold.js';

export function runCommand(options) {
  const runId = activeRunId(options.project, options.run);
  const paths = workflowPaths(options.project, runId);
  const config = readJsonFile(paths.config);
  const commands = config.commands ?? [];

  fs.mkdirSync(paths.logsDir, { recursive: true });

  const results = commands.map((commandConfig) => {
    const result = executeShellCommand(commandConfig.command, options.project);
    const safeName = commandConfig.name.replace(/[^a-zA-Z0-9_-]/g, '-');
    const stdoutPath = path.join(paths.logsDir, `${safeName}.stdout.log`);
    const stderrPath = path.join(paths.logsDir, `${safeName}.stderr.log`);

    fs.writeFileSync(stdoutPath, result.stdout);
    fs.writeFileSync(stderrPath, result.stderr);

    return {
      name: commandConfig.name,
      command: commandConfig.command,
      status: result.status,
      exitCode: result.exitCode,
      startedAt: result.startedAt,
      finishedAt: result.finishedAt,
      durationMs: result.durationMs,
      stdoutPath,
      stderrPath
    };
  });

  writeJsonFile(path.join(paths.runDir, 'execution.json'), {
    runId,
    results,
    summary: {
      total: results.length,
      passed: results.filter((result) => result.status === 'passed').length,
      failed: results.filter((result) => result.status === 'failed').length
    }
  });

  console.log(`Executed ${results.length} command(s).`);

  if (results.some((result) => result.status === 'failed')) {
    process.exitCode = 1;
  }
}
```

Modify `src/cli/main.js`:

```js
import { runCommand } from '../commands/run.js';
```

Add before scaffold dispatch:

```js
  if (command === 'run') {
    runCommand(options);
    return;
  }
```

- [ ] **Step 4: Run execution tests**

Run:

```bash
npm test -- test/run.test.js
```

Expected: test passes and the CLI process exits `1` when any configured command fails.

- [ ] **Step 5: Commit**

```bash
git add src/core/commands.js src/commands/run.js src/cli/main.js test/run.test.js
git commit -m "feat: capture test command execution"
```

## Task 7: Gate and Report Commands

**Files:**
- Create: `src/reporters/report.js`
- Create: `src/commands/report.js`
- Modify: `src/cli/main.js`
- Test: `test/report.test.js`

- [ ] **Step 1: Write failing report tests**

Create `test/report.test.js`:

```js
import test from 'node:test';
import assert from 'node:assert/strict';
import fs from 'node:fs';
import os from 'node:os';
import path from 'node:path';
import { spawnSync } from 'node:child_process';
import { fileURLToPath } from 'node:url';

const root = path.resolve(path.dirname(fileURLToPath(import.meta.url)), '..');
const bin = path.join(root, 'bin', 'testkit-flow.js');

test('gate and report aggregate execution evidence', () => {
  const project = fs.mkdtempSync(path.join(os.tmpdir(), 'testkit-flow-report-'));
  spawnSync(process.execPath, [bin, 'init', '--project', project, '--run', 'run-test'], { encoding: 'utf8' });

  const runDir = path.join(project, '.testkit-flow', 'runs', 'run-test');
  fs.writeFileSync(path.join(runDir, 'execution.json'), JSON.stringify({
    runId: 'run-test',
    results: [
      { name: 'unit', command: 'npm test', status: 'passed', exitCode: 0, durationMs: 12 },
      { name: 'e2e', command: 'npx playwright test', status: 'failed', exitCode: 1, durationMs: 30 }
    ],
    summary: { total: 2, passed: 1, failed: 1 }
  }, null, 2));

  const gate = spawnSync(process.execPath, [bin, 'gate', '--project', project], { cwd: root, encoding: 'utf8' });
  assert.equal(gate.status, 1);

  const report = spawnSync(process.execPath, [bin, 'report', '--project', project], { cwd: root, encoding: 'utf8' });
  assert.equal(report.status, 0, report.stderr);

  const gateJson = JSON.parse(fs.readFileSync(path.join(runDir, 'gate.json'), 'utf8'));
  assert.equal(gateJson.decision, 'FAIL');

  const reportMd = fs.readFileSync(path.join(runDir, 'report.md'), 'utf8');
  assert.match(reportMd, /# Testkit Flow Report/);
  assert.match(reportMd, /Gate Decision: FAIL/);
  assert.match(reportMd, /e2e/);
});
```

- [ ] **Step 2: Run report test to verify it fails**

Run:

```bash
npm test -- test/report.test.js
```

Expected: fail because `gate` and `report` are not implemented.

- [ ] **Step 3: Implement gate and report rendering**

Create `src/reporters/report.js`:

```js
export function gateDecision(execution) {
  if (!execution || !Array.isArray(execution.results)) {
    return {
      decision: 'BLOCKED',
      rationale: 'No execution evidence was found.',
      blockingFailures: []
    };
  }

  const failed = execution.results.filter((result) => result.status === 'failed');
  if (failed.length > 0) {
    return {
      decision: 'FAIL',
      rationale: `${failed.length} configured command(s) failed.`,
      blockingFailures: failed.map((result) => ({
        name: result.name,
        command: result.command,
        exitCode: result.exitCode
      }))
    };
  }

  return {
    decision: 'PASS',
    rationale: 'All configured commands passed.',
    blockingFailures: []
  };
}

export function renderGateMarkdown(gate) {
  return [
    '# Regression Gate',
    '',
    `Decision: ${gate.decision}`,
    '',
    `Rationale: ${gate.rationale}`,
    '',
    '## Blocking Failures',
    '',
    ...(gate.blockingFailures.length === 0
      ? ['None']
      : gate.blockingFailures.map((failure) => `- ${failure.name}: \`${failure.command}\` exited ${failure.exitCode}`)),
    ''
  ].join('\n');
}

export function renderReportMarkdown({ runId, gate, execution }) {
  const results = execution?.results ?? [];

  return [
    '# Testkit Flow Report',
    '',
    `Run: ${runId}`,
    `Gate Decision: ${gate.decision}`,
    `Rationale: ${gate.rationale}`,
    '',
    '## Command Results',
    '',
    ...(results.length === 0
      ? ['No command execution results were recorded.']
      : results.map((result) => `- ${result.name}: ${result.status} (${result.durationMs}ms) - \`${result.command}\``)),
    '',
    '## Evidence',
    '',
    '- `execution.json` contains structured command results.',
    '- `gate.json` contains the machine-readable gate decision.',
    ''
  ].join('\n');
}
```

Create `src/commands/report.js`:

```js
import fs from 'node:fs';
import path from 'node:path';
import { readJsonFile, writeJsonFile } from '../core/json.js';
import { workflowPaths } from '../core/paths.js';
import { activeRunId } from './scaffold.js';
import { gateDecision, renderGateMarkdown, renderReportMarkdown } from '../reporters/report.js';

function readJsonIfExists(filePath) {
  return fs.existsSync(filePath) ? readJsonFile(filePath) : undefined;
}

export function gateCommand(options) {
  const runId = activeRunId(options.project, options.run);
  const paths = workflowPaths(options.project, runId);
  const execution = readJsonIfExists(path.join(paths.runDir, 'execution.json'));
  const gate = gateDecision(execution);

  writeJsonFile(path.join(paths.runDir, 'gate.json'), gate);
  fs.writeFileSync(path.join(paths.runDir, 'gate.md'), renderGateMarkdown(gate));
  console.log(`Gate decision: ${gate.decision}`);

  if (gate.decision === 'FAIL' || gate.decision === 'BLOCKED') {
    process.exitCode = 1;
  }
}

export function reportCommand(options) {
  const runId = activeRunId(options.project, options.run);
  const paths = workflowPaths(options.project, runId);
  const execution = readJsonIfExists(path.join(paths.runDir, 'execution.json'));
  const existingGate = readJsonIfExists(path.join(paths.runDir, 'gate.json'));
  const gate = existingGate ?? gateDecision(execution);

  writeJsonFile(path.join(paths.runDir, 'gate.json'), gate);
  fs.writeFileSync(path.join(paths.runDir, 'gate.md'), renderGateMarkdown(gate));
  fs.writeFileSync(path.join(paths.runDir, 'report.md'), renderReportMarkdown({ runId, gate, execution }));
  writeJsonFile(path.join(paths.runDir, 'report.json'), { runId, gate, execution: execution ?? null });

  console.log(`Wrote report for ${runId}`);
}
```

Modify `src/cli/main.js`:

```js
import { gateCommand, reportCommand } from '../commands/report.js';
```

Add before scaffold dispatch:

```js
  if (command === 'gate') {
    gateCommand(options);
    return;
  }

  if (command === 'report') {
    reportCommand(options);
    return;
  }
```

- [ ] **Step 4: Run report tests and full test suite**

Run:

```bash
npm test
```

Expected: all tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/reporters/report.js src/commands/report.js src/cli/main.js test/report.test.js
git commit -m "feat: render gate and test reports"
```

## Task 8: Templates, Agent Prompts, and Workflow Docs

**Files:**
- Create: `agents/test-planner.md`
- Create: `agents/test-case-designer.md`
- Create: `agents/test-runner.md`
- Create: `agents/failure-triager.md`
- Create: `agents/coverage-analyst.md`
- Create: `agents/regression-gatekeeper.md`
- Create: `docs/workflow.md`
- Create: `docs/phases/01-init.md` through `docs/phases/13-learning.md`
- Modify: `README.md`
- Test: `test/docs.test.js`

- [ ] **Step 1: Write failing docs surface tests**

Create `test/docs.test.js`:

```js
import test from 'node:test';
import assert from 'node:assert/strict';
import fs from 'node:fs';
import path from 'node:path';
import { fileURLToPath } from 'node:url';

const root = path.resolve(path.dirname(fileURLToPath(import.meta.url)), '..');

test('workflow docs and agent prompts exist', () => {
  for (const file of [
    'docs/workflow.md',
    'agents/test-planner.md',
    'agents/test-case-designer.md',
    'agents/test-runner.md',
    'agents/failure-triager.md',
    'agents/coverage-analyst.md',
    'agents/regression-gatekeeper.md'
  ]) {
    const content = fs.readFileSync(path.join(root, file), 'utf8');
    assert.match(content, /Input/);
    assert.match(content, /Output/);
  }
});

test('all thirteen phase docs exist', () => {
  const phaseFiles = [
    '01-init.md',
    '02-intake.md',
    '03-discovery.md',
    '04-risk-mapping.md',
    '05-strategy.md',
    '06-case-design.md',
    '07-test-generation.md',
    '08-execution.md',
    '09-failure-triage.md',
    '10-coverage-gap.md',
    '11-regression-gate.md',
    '12-report.md',
    '13-learning.md'
  ];

  for (const file of phaseFiles) {
    const content = fs.readFileSync(path.join(root, 'docs', 'phases', file), 'utf8');
    assert.match(content, /Exit Criteria/);
  }
});
```

- [ ] **Step 2: Run docs tests to verify they fail**

Run:

```bash
npm test -- test/docs.test.js
```

Expected: fail because docs and agents do not exist.

- [ ] **Step 3: Create docs and prompts from the approved spec**

Create `agents/test-planner.md`:

```markdown
# Test Planner

## Input

- `intake.md`
- `acceptance-criteria.json`
- `discovery.json`
- `risk-register.json`

## Process

1. Confirm the testing objective.
2. Map high and medium risks to test levels.
3. Separate automated checks from manual exploration.
4. Define the required gate evidence.

## Output

- `test-strategy.md`

## Quality Bar

- Every high-risk item has a verification method.
- Every required gate has evidence.
- Out-of-scope items are explicit.
```

Create the other five agent files with this exact role mapping:

- `agents/test-case-designer.md`
  - Title: `# Test Case Designer`
  - Input: `acceptance-criteria.json`, `risk-register.json`, `test-strategy.md`
  - Process: map each criterion to cases; cover happy path, boundary, error, permission, state transition, and regression paths; mark automation fit.
  - Output: `test-cases.json`, `test-cases.md`
  - Quality Bar: every high-risk item has at least one case; expected results are behavioral.
- `agents/test-runner.md`
  - Title: `# Test Runner`
  - Input: `test-strategy.md`, `.testkit-flow/config.json`, generated or existing tests
  - Process: verify command intent; run commands through the CLI; preserve stdout, stderr, exit code, duration, and artifact paths.
  - Output: `execution.json`, `logs/`, `artifacts/`
  - Quality Bar: every configured command has structured evidence.
- `agents/failure-triager.md`
  - Title: `# Failure Triager`
  - Input: `execution.json`, `logs/`, `artifacts/`, `test-cases.json`
  - Process: classify failures as product defect, test defect, environment issue, data issue, dependency issue, flaky behavior, or unknown; attach evidence.
  - Output: `triage.md`, `failures.json`
  - Quality Bar: every failure has category, evidence, and next action.
- `agents/coverage-analyst.md`
  - Title: `# Coverage Analyst`
  - Input: coverage reports, `test-cases.json`, `risk-register.json`
  - Process: compare line coverage with behavior coverage; identify missing assertions, error paths, permission paths, and critical gaps.
  - Output: `coverage-gap.md`, `gap-matrix.json`
  - Quality Bar: critical risks are not closed by line coverage alone.
- `agents/regression-gatekeeper.md`
  - Title: `# Regression Gatekeeper`
  - Input: `execution.json`, `triage.md`, `coverage-gap.md`, `risk-register.json`
  - Process: recommend `PASS`, `PASS_WITH_RISK`, `FAIL`, or `BLOCKED`; summarize evidence and risk acceptance.
  - Output: `gate.md`, `gate.json`
  - Quality Bar: gate decision includes evidence and unblock steps for blocking outcomes.

Create `docs/workflow.md` with the 13 phases from the approved spec and sections named `Input`, `Action`, `Output`, and `Exit Criteria` for each phase.

Create each `docs/phases/*.md` with sections named exactly:

```markdown
# Phase Title

## Input

Input artifacts or project state from the phase mapping.

## Action

CLI action and AI operator action from the phase mapping.

## Output

Exact artifacts written by this phase from the phase mapping.

## Exit Criteria

Conditions required before moving to the next phase from the phase mapping.
```

Phase mapping:

- `01-init.md`: title `Init`; input target project path; action create workspace, config, active run, logs, artifacts, and rules directory; output `config.json`, active run directory, `rules/testing.md`; exit criteria config parses and required directories exist.
- `02-intake.md`: title `Intake`; input feature request, PRD, issue, diff, user story, or test objective; action extract functional claims, acceptance criteria, assumptions, open questions, and testable assertions; output `intake.md`, `acceptance-criteria.json`; exit criteria every requirement has a testable assertion.
- `03-discovery.md`: title `Discovery`; input target project directory; action detect language, package manager, test framework, commands, source directories, and test directories; output `discovery.json`, `project-map.md`; exit criteria at least one runnable validation command is known or a blocker is recorded.
- `04-risk-mapping.md`: title `Risk Mapping`; input intake, discovery, and optional diff; action classify functional, data, permission, integration, state, performance, security, and regression risks; output `risk-map.md`, `risk-register.json`; exit criteria high risks map to owner stages.
- `05-strategy.md`: title `Strategy`; input risk register, project capabilities, and acceptance criteria; action choose test levels and required gate evidence; output `test-strategy.md`; exit criteria high and medium risks map to verification methods.
- `06-case-design.md`: title `Case Design`; input acceptance criteria, risk register, and test strategy; action produce prioritized cases with preconditions, data, steps, expected results, and automation fit; output `test-cases.json`, `test-cases.md`; exit criteria criteria and high risks map to cases.
- `07-test-generation.md`: title `Test Generation`; input test cases, project conventions, and existing test patterns; action propose or create tests and record automation status; output test file drafts or implementation notes, `generation-report.md`; exit criteria generated tests are discoverable or manual steps are explicit.
- `08-execution.md`: title `Execution`; input test strategy and configured commands; action run commands and capture evidence; output `execution.json`, `logs/`, `artifacts/`; exit criteria every configured command has structured results.
- `09-failure-triage.md`: title `Failure Triage`; input execution output, logs, artifacts, test cases, and strategy; action classify failures and recommend next action; output `triage.md`, `failures.json`; exit criteria every failure has category, evidence, and next step.
- `10-coverage-gap.md`: title `Coverage Gap`; input coverage reports, test case matrix, and risk register; action identify untested behavior and missing assertions; output `coverage-gap.md`, `gap-matrix.json`; exit criteria remaining gaps have severity and remediation.
- `11-regression-gate.md`: title `Regression Gate`; input execution, triage, coverage gaps, and risks; action recommend gate decision with evidence; output `gate.md`, `gate.json`; exit criteria decision includes evidence and unblock or risk acceptance language.
- `12-report.md`: title `Report`; input all phase artifacts; action aggregate results, evidence, risks, failures, and recommendations; output `report.md`, `report.json`; exit criteria report is readable without raw logs and links evidence paths.
- `13-learning.md`: title `Learning`; input report, triage, coverage gaps, and project testing conventions; action extract reusable rules and failure patterns; output `learning.md`, `.testkit-flow/rules/testing.md`; exit criteria at least one reusable rule is recorded or the run states no new learning was found.

Modify `README.md` to include:

```markdown
# testkit-flow

A full AI-ready testing workflow tool based on harness-native testing evidence.

## Quickstart

```bash
npm install
npm test
node bin/testkit-flow.js init --project /path/to/project
node bin/testkit-flow.js discover --project /path/to/project
node bin/testkit-flow.js run --project /path/to/project
node bin/testkit-flow.js gate --project /path/to/project
node bin/testkit-flow.js report --project /path/to/project
```

## Workflow

The workflow has thirteen phases: init, intake, discovery, risk mapping, strategy, case design, test generation, execution, failure triage, coverage gap analysis, regression gate, report, and learning.

Read `docs/workflow.md` for the complete operator process.
```

- [ ] **Step 4: Run docs tests**

Run:

```bash
npm test -- test/docs.test.js
```

Expected: tests pass.

- [ ] **Step 5: Commit**

```bash
git add agents docs README.md test/docs.test.js
git commit -m "docs: add workflow docs and testing agents"
```

## Task 9: End-to-End CLI Smoke Test

**Files:**
- Create: `test/e2e.test.js`
- Modify: any files needed to fix integration issues found by the smoke test.

- [ ] **Step 1: Write failing end-to-end smoke test**

Create `test/e2e.test.js`:

```js
import test from 'node:test';
import assert from 'node:assert/strict';
import fs from 'node:fs';
import os from 'node:os';
import path from 'node:path';
import { spawnSync } from 'node:child_process';
import { fileURLToPath } from 'node:url';

const root = path.resolve(path.dirname(fileURLToPath(import.meta.url)), '..');
const bin = path.join(root, 'bin', 'testkit-flow.js');

function run(args, project) {
  return spawnSync(process.execPath, [bin, ...args, '--project', project], {
    cwd: root,
    encoding: 'utf8'
  });
}

test('complete workflow creates core artifacts and final report', () => {
  const project = fs.mkdtempSync(path.join(os.tmpdir(), 'testkit-flow-e2e-'));
  fs.writeFileSync(path.join(project, 'package.json'), JSON.stringify({
    scripts: {
      test: `${process.execPath} -e "console.log('unit ok')"`
    }
  }, null, 2));

  assert.equal(run(['init', '--run', 'run-test'], project).status, 0);
  for (const command of ['intake', 'discover', 'risk', 'strategy', 'cases', 'generate', 'run', 'triage', 'coverage', 'gate', 'report', 'learn']) {
    const result = run([command], project);
    assert.equal(result.status, 0, `${command}: ${result.stderr}`);
  }

  const runDir = path.join(project, '.testkit-flow', 'runs', 'run-test');
  for (const file of ['discovery.json', 'execution.json', 'gate.json', 'report.md', 'learning.md']) {
    assert.ok(fs.existsSync(path.join(runDir, file)), `${file} should exist`);
  }

  const gate = JSON.parse(fs.readFileSync(path.join(runDir, 'gate.json'), 'utf8'));
  assert.equal(gate.decision, 'PASS');
});
```

- [ ] **Step 2: Run smoke test to verify behavior**

Run:

```bash
npm test -- test/e2e.test.js
```

Expected before fixes: fail if any command dispatch, scaffolding, or default config behavior is incomplete.

- [ ] **Step 3: Fix integration issues**

Likely required fix: after `discover`, if config has no commands, populate `config.commands` with discovered test commands so `run` has something to execute.

Add this logic to `src/commands/discover.js` after writing discovery output:

```js
const config = readJsonFile(paths.config);
if (!Array.isArray(config.commands) || config.commands.length === 0) {
  config.commands = discovery.commands.filter((command) => ['test', 'coverage', 'build'].includes(command.name));
  writeJsonFile(paths.config, config);
}
```

Also update the import in `src/commands/discover.js`:

```js
import { readJsonFile, writeJsonFile } from '../core/json.js';
```

- [ ] **Step 4: Run full verification**

Run:

```bash
npm test
```

Expected: every test passes.

- [ ] **Step 5: Commit**

```bash
git add src/commands/discover.js test/e2e.test.js
git commit -m "test: add end-to-end workflow smoke coverage"
```

## Task 10: Final Verification and Release Readiness

**Files:**
- Modify: `README.md` if command examples drifted.
- Modify: `docs/workflow.md` if implementation behavior differs from the spec.

- [ ] **Step 1: Run incomplete marker scan**

Run:

```bash
rg -n "TB[D]|TO[D]O|FIXM[E]|placeholde[r]|implement late[r]|fill in detail[s]" README.md docs agents templates src test
```

Expected: no matches.

- [ ] **Step 2: Run full test suite**

Run:

```bash
npm test
```

Expected: all tests pass.

- [ ] **Step 3: Run local CLI smoke commands against the repository**

Run:

```bash
node bin/testkit-flow.js init --project . --run local-smoke
node bin/testkit-flow.js discover --project . --run local-smoke
node bin/testkit-flow.js run --project . --run local-smoke
node bin/testkit-flow.js gate --project . --run local-smoke
node bin/testkit-flow.js report --project . --run local-smoke
```

Expected: commands complete and `.testkit-flow/runs/local-smoke/report.md` exists.

- [ ] **Step 4: Clean local smoke artifacts if they should not be committed**

Run:

```bash
rm -rf .testkit-flow
git status --short
```

Expected: no `.testkit-flow/` artifacts remain in git status.

- [ ] **Step 5: Commit verification adjustments**

If docs changed:

```bash
git add README.md docs
git commit -m "docs: align quickstart with implemented CLI"
```

If no files changed, record the verification output in the final implementation report instead of creating an empty commit.

## Spec Coverage Checklist

- Documentation workflow layer: Task 8.
- CLI execution layer: Tasks 1 through 7.
- `.testkit-flow/` runtime workspace: Tasks 2, 3, 4, 6, and 7.
- Thirteen workflow phases: Tasks 4 and 8.
- Node, Python, Go, Rust discovery: Task 5.
- Command execution and structured evidence: Task 6.
- Gate and report artifacts: Task 7.
- Agent prompts: Task 8.
- End-to-end workflow verification: Task 9.
- Final verification and cleanup: Task 10.

## Validation Commands

Run these before claiming implementation complete:

```bash
npm test
rg -n "TB[D]|TO[D]O|FIXM[E]|placeholde[r]|implement late[r]|fill in detail[s]" README.md docs agents templates src test
node bin/testkit-flow.js --help
```

Expected:

- `npm test` passes.
- Placeholder scan returns no matches.
- CLI help prints command list and exits `0`.
