<!-- SPDX-License-Identifier: Apache-2.0 -->
# CI integration

This page shows how to block a merge with Impulse: `impulse run` and `impulse compare` exit 1 when the problems you choose appear.

## Quick version

- The loop: change a prompt or a tool, and run the scenarios. Compare with the last good run, and block the merge when something important broke.
- The two commands a pipeline needs:

```bash
impulse run evals/config.yaml --fail-on high                                    # exit 1: the run found a problem
impulse compare evals/baseline/run.json .impulse/runs/RUN_ID/ --fail-on high    # exit 1: something newly broke
```

- Exit code 0 means success.
- Exit code 1 means Impulse judged the run and found a problem.
- Exit code 2 means the setup is broken and nothing was judged.
- `impulse compare` exits 1 when a critical check newly fails, or a scenario newly errors. With `--fail-on high` it also exits 1 when a high check newly fails.
- A new scenario error counts whatever its cause, including a bot that was not running. Read the `scenario error:` note before you blame the agent.

## Details

### Exit codes

| command | exit code 0 | exit code 1 | exit code 2 |
|---|---|---|---|
| `impulse run CONFIG` | no `--fail-on` condition met (and always for `--dry-run` with a valid config) | the `--fail-on` condition is met | usage or config error, unresolved model, missing key |
| `impulse compare RUN_A RUN_B [--fail-on critical\|high]` | no critical regression (with `--fail-on high`: and no high regression) | a critical regression: a critical assertion fails in `RUN_B` and did not fail in `RUN_A`, or a scenario has an error in `RUN_B` and none in `RUN_A`; with `--fail-on high`, also a high regression: a high assertion newly fails, or UNIV-019 or UNIV-025 newly becomes poor | a run that cannot be found or read, or a usage error |
| `impulse doctor CONFIG` | every check passed | (never) | any `FAIL` row |

Treat exit code 2 differently from 1 in a pipeline:

- 2 means the run never judged anything: fix the setup.
- 1 means it did, and found a problem.

### --fail-on

`impulse run --fail-on critical|high|any` chooses what makes the run exit 1. Each level includes everything above it, and a scenario error exits 1 at every level:

| level | exit code 1 when |
|---|---|
| `critical` | any `critical` assertion fails, or any scenario has an `error` (including a critical check the judge could not evaluate) |
| `high` (default) | also: any `high` assertion fails, a `high` assertion got no judge verdict, or any scenario's UNIV-019 (harmful or misleading content) or UNIV-025 (expected tool calls) is `poor` |
| `any` | any scenario has `passed == False`: any failed assertion, any poor metric that is neither skipped nor informational, or any error |

- **`high` is the default** so that the failures that matter in every language block a merge without a compliance pack: a bot that asks for a password, invents urgency or makes a misleading claim (a poor UNIV-019), a wrong, missing or duplicate tool call (a poor UNIV-025), or a high-severity rule such as `rbi_kyc.no_unauthorized_commitment`. Pick `critical` to gate on critical rules and errors only.
- **A folder default:** `fail_on` under `[defaults]` in `.impulse/config.toml`. The flag overrides it. A `config.toml` without `fail_on` means `high`.
- **The reasons** are printed when the condition is met: `--fail-on high: exit 1`, then one line per reason. Example: `  - clinic_appointment_booking: UNIV-025 is poor (counts as severity high)`.
- **Provider errors** appear in these reasons as one line: type, HTTP status, provider, model, the provider's message, and the retry delay when known. So a quota error does not fill the job log; see [CLI reference](cli.md#--fail-on).

`impulse compare` takes `--fail-on` too, with two levels:

- `critical` (the default): exit 1 on a critical regression.
- `high`: also exit 1 on a high regression. That is a scenario where a `high` assertion fails in `RUN_B` and did not in `RUN_A`. Or it is one where UNIV-019 or UNIV-025 is poor in `RUN_B` and was not in `RUN_A`. The scenario's pass/fail change does not matter.
- `high` catches a scenario that already failed in the baseline for another reason, and now also makes a wrong or duplicate tool call.
- `impulse compare` never reads `fail_on` from `.impulse/config.toml`; pass the flag. `any` is not accepted (exit 2).

### Models and keys in CI

- A CI job needs no `.impulse/config.toml`.
- Set the models with `IMPULSE_MODEL_PRIMARY`, `IMPULSE_MODEL_FAST`, and `IMPULSE_MODEL_SIMULATOR`.
- Set each provider's key under its usual variable name (for example `OPENAI_API_KEY`), from your CI system's secrets.
- The environment wins over `config.toml` for keys. Key values never appear in Impulse's output or reports.
- For models, `config.yaml` values that are not `YOUR_..._MODEL` placeholders win over the environment.
- If you want a `config.toml` anyway, `impulse init --non-interactive --language en-IN` writes one with placeholders. It asks nothing and pings nothing.
- Every fresh checkout is a "first run" in its folder. So the privacy notice ("Impulse sends your system prompt and transcripts to the LLM and voice providers you configured.") appears in every job log.

### A GitHub Actions workflow

Not verified: this workflow has not been run on GitHub Actions. Its `impulse` commands match the CLI of this release. The compare exit codes were checked locally:

- 0 for two dry runs of `examples/config.yaml`;
- 1 when the agent raised in the second run;
- 2 for a run that does not exist.

What it does:

1. Keeps a baseline `run.json` from a known-good run in the repository, outside `.impulse/` (which is git-ignored).
2. Validates the config with a dry run (no LLM calls).
3. Runs the scenarios and compares with the baseline.
4. Uploads the run folders, then fails the job when the run or the compare found a problem.

```yaml
name: agent-evals

on:
  pull_request:

jobs:
  evals:
    runs-on: ubuntu-latest
    env:
      IMPULSE_MODEL_PRIMARY: ${{ vars.IMPULSE_MODEL_PRIMARY }}
      IMPULSE_MODEL_FAST: ${{ vars.IMPULSE_MODEL_FAST }}
      IMPULSE_MODEL_SIMULATOR: ${{ vars.IMPULSE_MODEL_SIMULATOR }}
      OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}   # the key variable of your provider
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install
        run: python -m pip install "git+https://github.com/Impulse-Eval/impulse-eval"   # or install from your clone

      - name: Validate the config (no LLM calls)
        run: impulse run evals/config.yaml --dry-run --reporter json

      - name: Run the scenarios
        id: run
        run: |
          set +e
          impulse run evals/config.yaml --fail-on high --reporter all
          status=$?
          set -e
          echo "status=$status" >> "$GITHUB_OUTPUT"
          echo "run_dir=$(ls -td .impulse/runs/*/ | head -n 1)" >> "$GITHUB_OUTPUT"
          if [ "$status" -eq 2 ]; then exit 2; fi

      - name: Compare with the baseline
        id: compare
        run: |
          set +e
          impulse compare evals/baseline/run.json "${{ steps.run.outputs.run_dir }}" --fail-on high | tee compare.txt
          status=${PIPESTATUS[0]}
          set -e
          echo "status=$status" >> "$GITHUB_OUTPUT"
          if grep -q "scenario error:" compare.txt; then
            echo "::warning::A scenario newly errored. Check the 'scenario error:' lines: an environment error (bot not running, LiveKit worker busy) also fails the compare."
          fi
          if [ "$status" -eq 2 ]; then exit 2; fi

      - name: Upload the reports
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: impulse-runs
          path: .impulse/runs/

      - name: Gate
        run: |
          if [ "${{ steps.run.outputs.status }}" != "0" ]; then echo "--fail-on condition met"; exit 1; fi
          if [ "${{ steps.compare.outputs.status }}" != "0" ]; then echo "critical or high regression"; exit 1; fi
```

How it handles the exit codes:

- GitHub Actions runs each step with `bash -e`. So the run and compare steps switch that off around the `impulse` command and keep its exit code. They stop at once only on exit code 2 (a broken setup).
- The compare step pipes its output through `tee`, so it reads the exit code from `PIPESTATUS[0]`, not from `tee`.
- The new run's folder is the newest folder under `.impulse/runs/`. The compare files written next to the folders are not folders. `impulse compare` takes a run folder or a `run.json` path as well as a run ID.
- The reports are uploaded whatever happened. Then the gate fails the job when the run met `--fail-on`, or the compare found a critical or high regression.
- To gate the compare on critical regressions only, drop `--fail-on high` from the compare step.
- To refresh the baseline after an intended change, copy the new `run.json` from the uploaded artifact to `evals/baseline/run.json` and commit it.

The `--dry-run` step is cheap and needs no models or keys. It catches config errors (exit code 2) before any paid call. It also computes the code-checked metrics, such as UNIV-025, with the generic adapter.

### Reading the compare result

`impulse compare RUN_A RUN_B` matches scenarios by name and prints:

- regressions (pass to fail) and improvements (fail to pass);
- newly failing checks (fail to fail): scenarios that failed in both runs, with a new failure in `RUN_B`. That is a metric or assertion that newly fails, or a new scenario error;
- the unchanged counts, and added and removed scenarios;
- the critical regressions and, when there are any, a `High regressions: {names}` line.

Each changed scenario lists its assertion changes, such as `dpdp.consent_before_pii: pass -> fail`. It also lists metric notes, such as `UNIV-023 newly failing` or `VOICE-001 improved 0.30`. A metric note is shown when a metric moves by 0.1 or more.

Two dry runs of the same config compare like this (verified):

```text
Compare 20261019T124757Z-3a7dc3 (2026-10-19T12:47:57Z) -> 20261019T124759Z-8315f8 (2026-10-19T12:47:59Z): pass rate +0.0 points
Regressions (pass -> fail): 0
Improvements (fail -> pass): 0
Newly failing checks (fail -> fail): 0
Unchanged: 6 pass, 0 fail
No critical regressions.
Note: run 20261019T124757Z-3a7dc3 is a dry run: judge metrics and assertions were skipped, so its results are not verdicts.
Note: run 20261019T124759Z-8315f8 is a dry run: judge metrics and assertions were skipped, so its results are not verdicts.
Wrote .impulse\runs\compare_20261019T124757Z-3a7dc3_vs_20261019T124759Z-8315f8.json
Wrote .impulse\runs\compare_20261019T124757Z-3a7dc3_vs_20261019T124759Z-8315f8.html
```

A `Note: run ... is a dry run` line is printed for each run whose judge metrics and assertions were skipped with the reason "dry run". Never use a dry run as the baseline: every scenario it "passed" was never judged. The note does not change the exit code.

The two files are written to `.impulse/runs/` (`--reporter json|html|all`, default `all`):

- `compare_<A>_vs_<B>.json`: the full `CompareResult`. It includes per-scenario metric deltas and the mean delta per metric (`metric_summary_deltas`). Skipped and informational results never enter a delta. The newly failing checks are in `check_regressions`, and the high regressions in `high_regressions`.
- `compare_<A>_vs_<B>.html`: a self-contained page. It has a critical regression banner, a "High regressions" banner when there are any, and the dry-run notes. Then the regressions, improvements, and "Newly failing checks (fail -> fail)" tables, and one inline bar per metric showing its mean delta. Attach it to the job as above.

What makes `impulse compare` exit 1:

- **By default:** only critical assertions that newly fail, and new scenario errors.
- **With `--fail-on high`:** also high regressions.
- **Newly failing checks alone** (fail -> fail) do not change the exit code, unless they are also critical or high regressions.
- **Not counted:** a high or medium regression under the default level, or a metric that got worse. They are reported but exit 0. Gate on high ones with `impulse compare --fail-on high` (or `impulse run --fail-on high`). Gate on everything with `impulse run --fail-on any`.

New scenario errors:

- A scenario with no error in `RUN_A` and an error in `RUN_B` is a critical regression. Examples: an agent crash, a timeout, or "critical check {rule_id} could not be evaluated: judge returned no verdict". The same condition makes `impulse run --fail-on critical` exit 1.
- It gets the note `scenario error: {error}`. An error in both runs, or only in `RUN_A`, is not a critical regression.
- **Environment errors count too.** In a real-world campaign (2026-10-25), a LiveKit worker that declined one job made the compare exit 1. On flaky infrastructure, read the `scenario error:` note before you blame the agent, and re-run the scenario.

A baseline scenario that ended with an error has no check results. So every check that fails in `RUN_B` for that scenario counts as newly failing. It is listed under "Newly failing checks", and a failed `high` assertion or a poor UNIV-019 or UNIV-025 there is a high regression. Use a baseline run without scenario errors, or read such rows with that in mind.

To look at a run's own report (players, timeline, failed checks), see [Reading the report](reading-the-report.md).

### Other commands for a CI loop

- **`impulse compare RUN_A RUN_B`** takes run IDs, run folders, or `run.json` paths, and writes `compare_<A>_vs_<B>.json` and `.html` to `.impulse/runs/`.
- **`impulse report RUN_ID`** rewrites a run's `report.html`. `--no-embed-audio` links the files in `audio/` instead of embedding them, which keeps CI artifacts small; the players then work only while the report sits next to its `audio/` folder ([CLI reference](cli.md#impulse-report)).
- **`impulse convert-failure .impulse/runs/<run_id>/scenarios/<name>.json`** turns a failed scenario into a regression scenario under `scenarios/regression/`. Review it and add it to your `config.yaml`, so the failure is gated on every later run ([Cookbook](cookbook.md#a-regression-test-from-a-failed-call)).

### From pytest

To run Impulse inside your own test suite, use the pytest helpers in `impulse_evals.testing`:

```python
from impulse_evals.testing import assert_no_critical, impulse_check


def test_booking_agent():
    run = impulse_check("config.yaml")          # or impulse_check("config.yaml", scenario_name="clinic_appointment_booking")
    assert_no_critical(run)
```

- `impulse_check(config_path, scenario_name=None)` runs the config the way `impulse run` does. It writes the run folder `.impulse/runs/<run_id>/` (with embedded audio in the report) and returns the `RunResult`.
- `impulse_check` never applies `--fail-on` itself.
- `assert_no_critical(run)` raises `AssertionError` under the `--fail-on critical` rule: a failed critical assertion or any scenario error, each named in the message. It keeps that rule although `impulse run` defaults to `high`; gate on high failures in pytest by checking `fail_on_reasons(run, "high")` from `impulse_evals.cli`.
- An unknown `scenario_name` raises before anything runs. A config problem raises `ConfigError`.

## Next steps

- [CLI reference](cli.md): every flag of `impulse run` and `impulse compare`.
- [Reading the report](reading-the-report.md): the report you attach to the job.
- [Writing scenarios](writing-scenarios.md): `expected_tool_calls` for UNIV-025 gates.
- [Compliance packs](compliance-packs.md): which assertions are critical.
- [Troubleshooting](troubleshooting.md): exit code 2 messages.

---

<!-- docs-nav -->
[← Previous: CLI reference](cli.md) | [Docs home](index.md) | [Next: Calibration →](calibration.md)
