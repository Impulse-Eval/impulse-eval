<!-- SPDX-License-Identifier: Apache-2.0 -->
# Getting started

This guide takes you from install to a dry run, your models, and a first real run, then points Impulse at your own agent.

## Quick version

- **What it is.** Impulse plays a simulated caller against your agent and judges each call against your agent's own system prompt. A dry run checks everything except the judge, with no model and no key.
- **The commands most people need.** Install straight from GitHub with `pip install "git+https://github.com/Impulse-Eval/impulse-eval"`. To try the examples, install from the root of a clone and make a dry run of the example:

  ```bash
  pip install -e .
  impulse run examples/config.yaml --dry-run
  ```

- **What success looks like.** The dry run exits with code 0 and prints `Run <run_id> (dry run): 6 scenarios, 6 passed, 0 failed (pass rate 100.0%)` (verified). It writes a report to `.impulse/runs/<run_id>/report.html`.

No paid API key? Follow [Run Impulse for free](run-for-free.md) instead: the same steps with Ollama and local speech models.

## Details

### 1. Install

Requires Python 3.11 or newer (tested on 3.11, 3.12 and 3.14). Python 3.10 is not supported.

Use a virtual environment:

```bash
python -m venv .venv
source .venv/bin/activate            # PowerShell: .venv\Scripts\Activate.ps1
```

Straight from GitHub, with no clone:

```bash
# Base install
pip install "git+https://github.com/Impulse-Eval/impulse-eval"

# With optional extras (Pipecat, LiveKit, local speech, etc.)
pip install "impulse-evals[pipecat] @ git+https://github.com/Impulse-Eval/impulse-eval"
pip install "impulse-evals[livekit] @ git+https://github.com/Impulse-Eval/impulse-eval"
pip install "impulse-evals[local] @ git+https://github.com/Impulse-Eval/impulse-eval"
```

The developer path, from the root of a clone of the repository (verified on 2026-10-26 in a fresh virtual environment, Windows, Python 3.14.7):

```bash
pip install -e .
```

The examples used below (`examples/config.yaml` and its agent) live in the repository, so the steps below assume a clone.

#### Optional extras

Combine them, e.g. `pip install "impulse-evals[local,pipecat] @ git+https://github.com/Impulse-Eval/impulse-eval"`, or `pip install -e ".[local,pipecat]"` from a clone:

| extra | adds | for |
|---|---|---|
| `local` | faster-whisper, transformers, soundfile | free in-process speech models (`local_hf`); install `torch` yourself (CPU or GPU build) |
| `light` | kokoro-onnx (not on Python 3.14; see [Troubleshooting](troubleshooting.md#voice)), faster-whisper | the light local voice (`local_light`): free speech models with no PyTorch and no GPU |
| `voice` | sounddevice, pydub (plus audioop-lts on Python 3.13+) | converting agent audio that is not WAV (pydub, with ffmpeg) |
| `livekit` | livekit, livekit-api, livekit-protocol | the LiveKit adapter |
| `pipecat` | pipecat-ai 1.12.x (capped below 1.13 until a newer release is tested) | the Pipecat adapter |
| `dev` | pytest-asyncio, ruff, mypy | working on Impulse itself |

On Windows, if `torch` or `livekit` fail to import with a missing `vcruntime140_1.dll`, install the latest Microsoft Visual C++ Redistributable (x64), then open a new shell ([Troubleshooting](troubleshooting.md)).

Success: `impulse --help` lists the commands `run`, `report`, `compare`, `doctor`, `init`, `generate`, `convert-failure`, and `calibrate` (verified).

### 2. Make a dry run

```bash
impulse run examples/config.yaml --dry-run
```

A dry run validates the config and makes zero LLM calls. It talks to the tiny echo agent in [examples/sample_agent.py](../examples/sample_agent.py) with a scripted caller: the scenario's `opening_user_message` or "Hello", then "I'd like to book". Judge-based metrics and assertions are skipped with the reason "dry run"; deterministic checks, such as UNIV-025 (the expected `book_slot` call), are computed.

Success (verified on Windows, Python 3.14; your run ID differs, and the paths use `/` outside Windows):

```text
Run 20261019T124757Z-3a7dc3 (dry run): 6 scenarios, 6 passed, 0 failed (pass rate 100.0%)
  PASS  sales_lead_followup_hindi
  PASS  recruitment_interview_booking
  PASS  clinic_appointment_booking
  PASS  loan_emi_reminder_outbound
  PASS  support_order_status_us
  PASS  cita_clinica_mx
Warnings:
  - Dry run: judge metrics and assertions were skipped; results are not verdicts.
Wrote .impulse\runs\20261019T124757Z-3a7dc3\run.json
Wrote .impulse\runs\20261019T124757Z-3a7dc3\report.html
Dry run: --fail-on is not applied.
```

The exit code is 0. "6 passed" here only means the config and the deterministic checks work: the warning (also in the report header) says the judge checks were skipped. Impulse created `.impulse/` in the current directory, with a `.gitignore` that keeps it out of version control.

The six scenarios cover several markets. Four are Indian: Hindi sales, English recruitment and clinic booking, and a Hinglish loan reminder. `support_order_status_us` is in US English, and `cita_clinica_mx` in Mexican Spanish.

### 3. Open the report

Open `.impulse/runs/<run_id>/report.html` in a browser. It is one self-contained file, in this order:

- a header: agent, languages, judge model, warnings;
- the critical failures, the scenario table, the language and accent cohorts;
- the metric summary and the latency profile;
- each scenario's transcript, with failing turns highlighted;
- the tool call trace per turn.

Success: six scenarios are marked as passed, the judge metrics show as skipped with "dry run", and UNIV-025 is good for the three booking scenarios.

What your first report looks like: a dry run has the same layout. A real voice run, such as the one in the video below, adds a full call recording and a player per turn. It also adds "Hear turn N" links next to failed checks, and a call timeline. [Reading the report](reading-the-report.md) explains every part.

https://github.com/user-attachments/assets/a7272d6c-41a7-4efa-af84-4b815421a5e6

A real voice report, with sound: the call recording, failed checks with "Hear turn N" links, the call timeline, and a highlighted failing turn. ([Download the MP4](images/report-walkthrough.mp4))

### 4. Choose your models

Impulse needs three LiteLLM model strings, one per role:

- the primary judge: critical and high compliance checks, and the most important metrics;
- the fast judge: the remaining checks;
- the simulator: the simulated caller.

Write them once for this folder:

```bash
impulse init
```

The default profile, `custom`, asks in this order:

1. the default language, a code such as `en-IN` (Impulse never assumes one);
2. for each role, a LiteLLM model string and an optional `api_base` (for a local or self-hosted server);
3. the keys the hosted models you chose need, with hidden input.

`--profile hosted` asks for the keys first; `--profile free` asks for Ollama tags ([Run Impulse for free](run-for-free.md)). Instead of `impulse init`, you can set `IMPULSE_MODEL_PRIMARY`, `IMPULSE_MODEL_FAST`, `IMPULSE_MODEL_SIMULATOR`, and your provider's key variable in the environment.

Success: `Wrote .impulse/config.toml (profile custom, language en-IN) and .impulse/.gitignore` (with your language), a `PASS` row for each model that answered its 5-token ping, and `Next: impulse doctor`. The exact `custom` profile flow with real models was not run while writing this page (not verified).

### 5. Check the setup

```bash
impulse doctor examples/config.yaml
```

It checks Python, `.impulse/config.toml`, the config, each model (configured, recognised by LiteLLM, its key present, answering a ping), the agent entrypoint, and the voice providers. A row that says `FAIL` tells you what to fix. Before any model is configured, for example, the output includes (verified):

```text
FAIL  primary model     judge.primary_model: no model configured for the primary role. Set judge.primary_model in config.yaml, the environment variable IMPULSE_MODEL_PRIMARY, or [models] primary in .impulse/config.toml.
PASS  agent entrypoint  imported sample_agent:respond
INFO  voice providers   text mode (no voice_config)
```

Success: the last line is `All checks passed.` and the exit code is 0 (it is 2 while any row says `FAIL`). The maintainer's test campaigns saw "all PASS" with hosted models ([Verified setups](verified-setups.md)).

### 6. Run for real

```bash
impulse run examples/config.yaml
```

On the first real run in a folder, Impulse prints:

> Impulse sends your system prompt and transcripts to the LLM and voice providers you configured.

Then the simulated caller talks to the example agent. The judge grades every conversation against [examples/system_prompt.txt](../examples/system_prompt.txt). The run folder and report are written as in steps 2 and 3.

Success: a summary line `Run <run_id> (text mode): 6 scenarios, ...`, then one `PASS` or `FAIL` line per scenario with the failed checks named. The exit code is 0, or 1 when a `--fail-on` condition is met (by default a failed critical or high assertion, a scenario error, or a poor UNIV-019 or UNIV-025). Exit code 2 means a configuration problem; the message names the field. Real runs were verified against LiveKit and Pipecat test bots with hosted models; this example config with real models was not (not verified).

### 7. Test your own agent

The generic adapter calls any Python function `(user_text, history) -> str | dict`, sync or async. Copy [examples/config.yaml](../examples/config.yaml) next to your agent and change the `agent` block:

```yaml
agent:
  type: voice                          # voice | chat | multimodal
  adapter: generic
  entrypoint: "sample_agent:respond"   # examples/sample_agent.py; this folder is added to sys.path
  system_prompt: "system_prompt.txt"   # relative to this file; wins over sample_agent.SYSTEM_PROMPT
  model: null
  tools:                               # the tools the agent can call (names checked against expected/forbidden calls)
    - name: book_slot
      description: "Book a demo call, interview round, or clinic appointment. Arguments: date, patient_name, phone."
    - name: cancel_slot
      description: "Cancel an existing booking. Arguments: date, phone."
```

1. Set `entrypoint` to your `module:function`, importable from the config's folder.
2. Set `system_prompt` to your agent's system prompt file: it is the judge's rubric.
3. Set `tools` to your agent's tools, or `null` when you do not want to list them.
4. Replace the scenarios with your own ([Writing scenarios](writing-scenarios.md)), or draft them from your prompt: `impulse generate --prompt system_prompt.txt --domain ecommerce --language en-US` writes `scenarios/generated.yaml` for you to review. Use your own language code, for example `en-IN` or `es-MX`.
5. Run `impulse doctor config.yaml`, then `impulse run config.yaml --dry-run`, then `impulse run config.yaml`.

Success: the doctor's `agent entrypoint` row says `PASS  imported <your module:function>` and the dry run exits 0. To check tool calls and what your agent heard, return a dict with `tool_calls` and `heard_text` ([Adapters](adapters.md#generic-adapter)). For a LiveKit agent, a Pipecat bot, an agent behind HTTP, or your own adapter class, see [Adapters](adapters.md).

## Next steps

- [Reading the report](reading-the-report.md): what each part of the report shows.
- [Concepts](concepts.md): how the judge, metrics, and pass logic work.
- [Writing scenarios](writing-scenarios.md): scenarios for your own agent.
- [CLI reference](cli.md): every command and flag.
- [CI integration](ci-integration.md): gate merges on the results.
- [Known limitations](known-limitations.md): what to expect before you rely on a result.

---

<!-- docs-nav -->
[← Previous: Docs home](index.md) | [Docs home](index.md) | [Next: Run Impulse for free →](run-for-free.md)
