# ✅ SFDX Run Tests

This repository implements a unified GitHub composite action to run Salesforce **Apex**, **LWC (Jest)** and **Flow** tests from a single step. Each test type can be toggled on or off, coverage is highlighted in the CLI logs and the GitHub step summary, and the coverage reports are written to the paths expected by quality tools like **SonarQube/SonarCloud** or **Codecov**.

Apex tests are enabled by default (they are mandatory in Salesforce); LWC and Flow tests are opt-in.

## Usage

### Run all test types and report to Sonar

This streamlines the otherwise separate, manual test steps into one action and produces the coverage reports a subsequent SonarCloud scan consumes:

```yaml
jobs:
  validation:
    name: Validation
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Select Node Version
        uses: svierk/get-node-version@main

      - name: Install Dependencies
        run: npm install --ignore-scripts

      - name: Install SF CLI
        uses: svierk/sfdx-cli-setup@main

      - name: Salesforce Org Login
        uses: svierk/sfdx-login@main
        with:
          client-id: ${{ secrets.SFDX_CONSUMER_KEY }}
          jwt-secret-key: ${{ secrets.SFDX_JWT_SECRET_KEY }}
          username: ${{ vars.SFDX_USERNAME }}

      - name: Deploy Metadata
        run: sf project deploy start

      - name: Run Tests
        uses: svierk/sfdx-run-tests@main
        with:
          apex: true
          lwc: true
          flow: false

      - name: SonarCloud Scan
        uses: SonarSource/sonarqube-scan-action@master
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

> LWC tests run locally with Jest, so make sure the Node.js dependencies are installed first (e.g. `npm install`). Apex and Flow tests run against the authenticated org, so log in (and deploy the metadata) beforehand. When Flow tests are enabled, the required `@salesforce/plugin-flow` CLI plugin is installed automatically if it is not already present.

In the workflow logs, each test run reports a human-readable result - one line per test plus a run summary - folded into a collapsible section, with a concise pass/fail line that stays visible. Test failures are always printed unfolded (including message and stack trace), so they can be inspected without expanding anything.

### Apex tests only (default)

```yaml
      - name: Run Tests
        uses: svierk/sfdx-run-tests@main
        with:
          target-org: ci
          test-level: RunLocalTests
```

## Inputs

| Name               | Required | Default                     | Description                                                                                   |
| ------------------ | -------- | --------------------------- | --------------------------------------------------------------------------------------------- |
| `apex`             | no       | `true`                      | Run Apex tests.                                                                                |
| `lwc`              | no       | `false`                     | Run LWC (Jest) unit tests.                                                                     |
| `flow`             | no       | `false`                     | Run Flow tests.                                                                                |
| `target-org`       | no       |                             | Username or alias of the target org (Apex and Flow). Not required if the default org is set.   |
| `test-level`       | no       | `RunLocalTests`             | Test level for Apex and Flow: `RunLocalTests`, `RunAllTestsInOrg` or `RunSpecifiedTests`.      |
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

Outputs for a test type that was not run are empty. The step fails if any enabled test type fails, while still writing the summary and outputs.

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

## References

- [apex run test](https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_reference.meta/sfdx_cli_reference/cli_reference_apex_commands_unified.htm#cli_reference_apex_run_test_unified) | Salesforce CLI Command Reference
- [flow run test](https://developer.salesforce.com/docs/platform/salesforce-cli-reference/guide/cli_reference_flow_run_test.html) | Salesforce CLI Command Reference
- [flow get test](https://developer.salesforce.com/docs/platform/salesforce-cli-reference/guide/cli_reference_flow_get_test.html) | Salesforce CLI Command Reference
- [sfdx-lwc-jest](https://github.com/salesforce/sfdx-lwc-jest) | Run Jest against LWC components

## Releases

Latest release notes can be found on the [release page](https://github.com/svierk/sfdx-run-tests/releases).

## License

The scripts and documentation in this project are released under the [MIT License](https://github.com/svierk/sfdx-run-tests/blob/main/LICENSE).
