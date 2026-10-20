# ✅ SFDX Run Tests

This repository implements a unified GitHub composite action to run Salesforce **Apex**, **LWC (Jest)** and **Flow** tests from a single step. Each test type can be toggled on or off, coverage is highlighted in the CLI logs and the GitHub step summary, and the coverage reports are written to the paths expected by quality tools like **SonarQube/SonarCloud** or **Codecov**.

Apex tests are enabled by default (they are mandatory in Salesforce); LWC and Flow tests are opt-in.

## Usage

### Run all test types and report to Sonar

This streamlines the otherwise separate, manual test steps into one action and produces the coverage reports a subsequent SonarCloud scan consumes:

```yaml
# Least-privilege token: this workflow only reads the repository.
permissions:
  contents: read

jobs:
  validation:
    name: Validation
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v7.0.1
        with:
          fetch-depth: 0 # required for the Sonar analysis
          persist-credentials: false # the GITHUB_TOKEN is not needed after the checkout - don't leave it in .git/config

      - name: Select Node Version
        uses: svierk/get-node-version@v1.5.2

      - name: Install Dependencies
        run: npm ci --ignore-scripts

      - name: Install SF CLI
        uses: svierk/sfdx-cli-setup@v1.2.0

      - name: Salesforce Org Login
        uses: svierk/sfdx-login@v1.4.3
        with:
          client-id: ${{ secrets.SFDX_CONSUMER_KEY }}
          jwt-secret-key: ${{ secrets.SFDX_JWT_SECRET_KEY }}
          username: ${{ vars.SFDX_USERNAME }}

      - name: Deploy Metadata
        run: sf project deploy start

      - name: Run Tests
        uses: svierk/sfdx-run-tests@v1.2.0
        with:
          apex: true
          lwc: true
          flow: false

      - name: SonarCloud Scan
        uses: SonarSource/sonarqube-scan-action@v8.2.1
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

> LWC tests run locally with Jest, so make sure the Node.js dependencies are installed first (e.g. `npm install`). Apex and Flow tests run against the authenticated org, so log in (and deploy the metadata) beforehand. When Flow tests are enabled, the required `@salesforce/plugin-flow` CLI plugin is installed automatically if it is not already present.

In the workflow logs, each test run reports a human-readable result - one line per test plus a run summary - folded into a collapsible section, with a concise pass/fail line that stays visible. Test failures are always printed unfolded (including message and stack trace), so they can be inspected without expanding anything.

### Apex tests only (default)

```yaml
      - name: Run Tests
        uses: svierk/sfdx-run-tests@v1.2.0
        with:
          target-org: ci
          test-level: RunLocalTests
```

### Run specific tests (RunSpecifiedTests)

To run only part of the tests, e.g. a fast smoke test on every push, select them via one of the selection inputs. They map 1:1 to the selection flags of the Salesforce CLI:

| Test type | Input              | CLI flag        | Values                                         |
| --------- | ------------------ | --------------- | ---------------------------------------------- |
| Apex      | `apex-class-names` | `--class-names` | Test classes, all methods of each class run    |
| Apex      | `apex-suite-names` | `--suite-names` | Test suites                                    |
| Apex      | `apex-tests`       | `--tests`       | `Class` or `Class.method` (`ns.Class.method` for namespaced tests) |
| Flow      | `flow-class-names` | `--class-names` | Flow API names, all flow tests of each flow run |
| Flow      | `flow-tests`       | `--tests`       | Individual flow tests as `FlowName.TestName`   |

All selection inputs accept a comma-separated string as well as a multi-line block, and the two notations can be mixed freely. A comma-separated string keeps short lists on a single line, a block stays readable once a list grows:

```yaml
      - name: Run Tests
        uses: svierk/sfdx-run-tests@v1.2.0
        with:
          flow: true
          apex-tests: | # multi-line block
            AccountServiceTest
            OpportunityServiceTest.testClosedWon
            LeadConversionTest
          flow-class-names: 'Create_Case,Update_Account' # comma-separated
```

Surrounding whitespace and blank lines are ignored, so `'Create_Case, Update_Account'` is read exactly like `'Create_Case,Update_Account'`.

How the selection interacts with `test-level`:

- A test type with a selection **always runs with `RunSpecifiedTests`**, so `test-level` does not need to be changed. This also allows mixing, e.g. a selection of Apex tests while the Flow tests keep running with `RunLocalTests`.
- Setting `test-level: RunSpecifiedTests` without a selection for an enabled test type fails the step with a clear error before any test runs.
- The CLI accepts only one kind of selection per test type, so set either `apex-class-names`, `apex-suite-names` or `apex-tests` (and either `flow-class-names` or `flow-tests`). Combining them fails the step up front as well.

> With a selection, coverage only reflects the selected tests. Keep that in mind if a subsequent quality gate (e.g. Sonar) enforces a minimum coverage.

## Inputs

| Name               | Required | Default                     | Description                                                                                   |
| ------------------ | -------- | --------------------------- | --------------------------------------------------------------------------------------------- |
| `apex`             | no       | `true`                      | Run Apex tests.                                                                                |
| `lwc`              | no       | `false`                     | Run LWC (Jest) unit tests.                                                                     |
| `flow`             | no       | `false`                     | Run Flow tests.                                                                                |
| `target-org`       | no       |                             | Username or alias of the target org (Apex and Flow). Not required if the default org is set.   |
| `test-level`       | no       | `RunLocalTests`             | Test level for Apex and Flow: `RunLocalTests`, `RunAllTestsInOrg` or `RunSpecifiedTests`. A test type with a selection always runs with `RunSpecifiedTests` (see [Run specific tests](#run-specific-tests-runspecifiedtests)). |
| `apex-class-names` | no       |                             | Apex test classes to run, comma-separated or multi-line.                                       |
| `apex-suite-names` | no       |                             | Apex test suites to run, comma-separated or multi-line.                                        |
| `apex-tests`       | no       |                             | Apex tests to run as `Class` or `Class.method`, comma-separated or multi-line.                 |
| `flow-class-names` | no       |                             | Flows (API names) whose flow tests should run, comma-separated or multi-line.                  |
| `flow-tests`       | no       |                             | Individual flow tests to run as `FlowName.TestName`, comma-separated or multi-line.            |
| `code-coverage`    | no       | `true`                      | Collect code coverage for Apex and Flow tests.                                                 |
| `apex-output-dir`  | no       | `./tests/apex`              | Directory for Apex result and coverage files (contains `test-result-codecoverage.json`).       |
| `flow-output-dir`  | no       | `./tests/flow`              | Directory for Flow result and coverage files.                                                  |
| `lwc-test-command` | no       | `npx sfdx-lwc-jest --coverage` | Command used to run the LWC Jest tests with coverage (must write an lcov report). Works on any SFDX project without a specific package.json script; override it to use your own, e.g. `npm run test:unit:coverage`. |
| `lwc-lcov-path`    | no       | `./coverage/lcov.info`      | Path to the lcov report produced by the LWC tests, used to report overall coverage.            |
| `wait`             | no       | `33`                        | Number of minutes to wait for the Apex and Flow test runs to complete.                         |
| `api-version`      | no       |                             | Override the api version used for Apex and Flow test api requests, e.g. `59.0`.                |
| `step-summary`     | no       | `true`                      | Write a result section to the GitHub Actions [job summary](https://docs.github.com/en/actions/using-workflows/workflow-commands-for-github-actions#adding-a-job-summary). Set to `false` to avoid collisions with a custom workflow summary. |

## Outputs

| Name            | Description                                             |
| --------------- | ------------------------------------------------------- |
| `apex-outcome`  | Outcome of the Apex test run (`Passed`/`Failed`).       |
| `apex-coverage` | Overall Apex code coverage.                             |
| `lwc-outcome`   | Outcome of the LWC test run (`Passed`/`Failed`).        |
| `lwc-coverage`  | Overall LWC line coverage.                              |
| `flow-outcome`  | Outcome of the Flow test run (`Passed`/`Failed`).       |
| `flow-coverage` | Overall Flow code coverage.                             |

Outputs for a test type that was not run are empty. The step fails if any enabled test type fails, while still writing the summary and outputs. An invalid test selection (see above) fails the step before any test runs.

## Coverage reporting (Sonar / Codecov)

The action writes coverage reports to standard locations so downstream quality gates can pick them up without extra steps:

| Test type | Report | Matching Sonar property |
| --------- | ------ | ----------------------- |
| Apex      | `<apex-output-dir>/test-result-codecoverage.json` | `sonar.apex.coverage.reportPath` |
| LWC       | `<lwc-lcov-path>` (`./coverage/lcov.info`)         | `sonar.javascript.lcov.reportPaths` |

Example `sonar-project.properties` snippet:

```properties
sonar.javascript.lcov.reportPaths=./coverage/lcov.info
sonar.apex.coverage.reportPath=./tests/apex/test-result-codecoverage.json
```

> The scan step above only needs `SONAR_TOKEN` (plus `SONAR_HOST_URL` for a self-hosted SonarQube Server). A `GITHUB_TOKEN` is **not** required: pull request decoration is done by the SonarQube Cloud GitHub App, so handing the repository token to a third-party action would widen its access for no benefit.

## 🔐 Security & versioning

Every `uses:` reference in the snippets above is **pinned to an exact release version**, e.g. `svierk/sfdx-run-tests@v1.2.0`. Do the same in your own pipelines:

- **Never reference a mutable ref** such as `@main` or `@v1`. It runs whatever code sits behind that branch/tag at run time - with access to your org credentials - so a compromised or rewritten ref would run unnoticed.
- **Good - pin to an exact release tag** (`@v1.2.0`). Readable, concrete, and bumped through reviewed pull requests. This is what this repository itself uses.
- **Strictest - pin to a full-length commit SHA** (`@a1b2c3d…`) with the version as a trailing comment. A SHA can never be re-pointed by the publisher; the cost is readability.
- **Enable [Dependabot](https://docs.github.com/en/code-security/dependabot/working-with-dependabot/keeping-your-actions-up-to-date-with-dependabot) for `github-actions`** in the repository that holds your pipelines, so those pins are bumped for you instead of silently ageing.

Beyond pinning, the same rules apply to workflows using this action:

- **Least-privilege `GITHUB_TOKEN`** - declare a `permissions:` block granting only what the workflow needs (`contents: read` for a validation pipeline).
- **`persist-credentials: false` on checkout** - the token is not written to `.git/config`, so later steps (SF CLI, third-party actions) cannot reuse it.
- **Secrets travel as secrets** - pass them into an action input or expose them as **environment variables** in shell steps (`"$SONAR_TOKEN"`), never by interpolating `${{ secrets.* }}` into the script itself - that would allow command injection and can leak values into the log. This action follows the same rule internally: every input is handed to the composite script via `env:` and read as a shell variable.
- **Validate pull requests with `pull_request`, never `pull_request_target`** - the latter runs with the base repository's secrets, which would let a fork execute its own code against your org.

## References

- [apex run test](https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_reference.meta/sfdx_cli_reference/cli_reference_apex_commands_unified.htm#cli_reference_apex_run_test_unified) | Salesforce CLI Command Reference
- [flow run test](https://developer.salesforce.com/docs/platform/salesforce-cli-reference/guide/cli_reference_flow_run_test.html) | Salesforce CLI Command Reference
- [flow get test](https://developer.salesforce.com/docs/platform/salesforce-cli-reference/guide/cli_reference_flow_get_test.html) | Salesforce CLI Command Reference
- [sfdx-lwc-jest](https://github.com/salesforce/sfdx-lwc-jest) | Run Jest against LWC components

## Releases

Latest release notes can be found on the [release page](https://github.com/svierk/sfdx-run-tests/releases).

## License

The scripts and documentation in this project are released under the [MIT License](https://github.com/svierk/sfdx-run-tests/blob/main/LICENSE).
