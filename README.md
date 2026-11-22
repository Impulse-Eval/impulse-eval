<!-- SPDX-License-Identifier: Apache-2.0 -->
# Impulse Evals

<p align="center">
  <img src="docs/images/impulse-demo.svg" alt="Impulse demo: the agent says the appointment is booked after the tool call failed, and the judge fails the run" width="720">
</p>

Impulse Evals tests AI voice and chat agents: a simulated caller talks to your agent, and an LLM judge grades it against your system prompt. It is vendor-neutral and private: it works with any stack, runs on your machine or in your CI, and has compliance checks built in.

Any language, any stack, any industry.

Give us feedback at shrikant.s@testvoiceai.live: any bugs, feature requests, or ideas you have. If it qualifies, you'll be added to contributors.

> [!TIP]
> Using Claude? Install the Impulse Evals skill ([docs/skill.md](docs/skill.md)) and ask it to write your config.yaml.

https://github.com/user-attachments/assets/a7272d6c-41a7-4efa-af84-4b815421a5e6

An Impulse voice report, with sound: the call recording, failed checks with "Hear turn N" links, a timeline with slow replies in red, and a highlighted failing turn. ([Download the MP4](docs/images/report-walkthrough.mp4))

## Quickstart (60 seconds)

From the root of a clone of this repository, with Python 3.11 or newer:

```bash
pip install git+https://github.com/Impluse-Evals/impulse-eval
 .

impulse run examples/config.yaml --dry-run   # no model and no key needed
impulse init                                 # asks for the language and a model per role
impulse doctor examples/config.yaml          # shows what a real run still needs
impulse run examples/config.yaml             # a real run with your models
```

What success looks like: the dry run needs no model and no key, and exits with code 0. It prints this (your run ID differs; paths use `/` outside Windows):

```text
Run 20261025T233715Z-cf132e (dry run): 6 scenarios, 6 passed, 0 failed (pass rate 100.0%)
  PASS  sales_lead_followup_hindi
  PASS  recruitment_interview_booking
  PASS  clinic_appointment_booking
  PASS  loan_emi_reminder_outbound
  PASS  support_order_status_us
  PASS  cita_clinica_mx
Warnings:
  - Dry run: judge metrics and assertions were skipped; results are not verdicts.
Wrote .impulse\runs\20261025T233715Z-cf132e\run.json
Wrote .impulse\runs\20261025T233715Z-cf132e\report.html
Dry run: --fail-on is not applied.
```

- Open `.impulse/runs/<run_id>/report.html` in a browser. [Reading the report](docs/reading-the-report.md) explains every part of it.
- `impulse doctor` shows what a real run still needs. Before you choose models, it prints a `FAIL` row for each model role (primary, fast, simulator) and exits with code 2. The other rows say `PASS` or `INFO`. `impulse init` fixes them ([A real run](#a-real-run)).

Verified on 2026-10-26: `pip install -e .`, the dry run and `impulse doctor` in a fresh virtual environment, on Windows with Python 3.14.7. `impulse init` and the real run depend on the models you choose; see [Status](#status).

**Documentation:** start at the [docs home](docs/index.md).

## Key features

Impulse Evals (formerly PULSE): package `impulse-evals`, command `impulse`, import package `impulse_evals`, version 0.1.0. The project was renamed from PULSE in this release; the old names are no longer used.

**No paid API key?** Start with [Run Impulse for free](docs/run-for-free.md): Ollama for the language models, local speech models for voice.

Impulse plays a simulated caller against your agent, in text or in voice. In voice, real text-to-speech (TTS) audio goes in, and the agent's own audio comes back through speech-to-text (STT). An LLM judge grades every conversation against your agent's own system prompt plus a small universal safety floor. So a sales agent is judged on its sales instructions, and an interview agent on its interview rules. There is no rubric to write and no domain-specific judge code.

- **Any stack.** Adapters:
  - generic: any Python function;
  - LiveKit and Pipecat: Impulse joins your running agent as a client;
  - `http`: an agent in any language behind JSON endpoints;
  - `custom`: your own adapter class, for example for Vapi, Retell or an in-house platform.
- **Tool calls.** The right tool with the right arguments, and no "you're booked" when the booking call failed or was never made.
- **Honest latency.** Measured on the agent only, never Impulse's own TTS or STT time.
- **Any language.** Every language and accent is reported as its own cohort, never blended.
- **CI.** `impulse compare` diffs two runs for regression gates.
- **Compliance packs (optional).** Judge-based checks for regulated industries. India's packs are one example:
  - DPDP, for the Digital Personal Data Protection Act;
  - RBI KYC, for the Reserve Bank of India's Know Your Customer rules;
  - TRAI DLT, for the Telecom Regulatory Authority of India's Distributed Ledger Technology registration of commercial calls.

  An opt-in HIPAA pack covers US healthcare. The domain defaults of `impulse generate` add the India packs only for languages with the region IN.
- **Your models.** You bring the LLMs as LiteLLM model strings; with Ollama and local speech models no paid key is ever needed.
- **A report you can hear.** One self-contained HTML page per run, with the call recording and a player per turn. Each failed check has a "Hear turn N" link ([Reading the report](docs/reading-the-report.md)).
- **Noise simulation (experimental).** Mix white, pink, brown or your own recorded background noise into the caller's audio at a chosen signal-to-noise ratio, and see where your agent stops hearing callers. Not yet verified against a live agent ([Voice testing](docs/voice-testing.md#noise-simulation-experimental)):

  ```yaml
  voice_config:
    tts: {provider: local_light}
    stt: {provider: local_light}
    noise: {profile: pink, snr_db: 10}   # 20 = mild, 10 = busy street, 5 = harsh
  ```

## Install

Requires Python 3.11 or newer (tested on 3.11, 3.12 and 3.14). Python 3.10 is not supported.

Straight from GitHub, with no clone:

```bash
# Base install
pip install "git+https://github.com/Impluse-Evals/impulse-eval"

# With optional extras (Pipecat, LiveKit, local speech, etc.)
pip install "impulse-evals[pipecat] @ git+https://github.com/Impluse-Evals/impulse-eval"
pip install "impulse-evals[livekit] @ git+https://github.com/Impluse-Evals/impulse-eval"
pip install "impulse-evals[local] @ git+https://github.com/Impluse-Evals/impulse-eval"
```

From a clone of this repository (needed for the examples in `examples/`):

```bash
pip install -e .
```

Optional extras (combine them, e.g. `pip install "impulse-evals[local,pipecat] @ git+https://github.com/Impluse-Evals/impulse-eval"`, or `pip install -e ".[local,pipecat]"` from a clone):

| extra | adds | for |
|---|---|---|
| `local` | faster-whisper, transformers, soundfile | free in-process speech models (`local_hf`); install `torch` yourself (CPU or GPU build) |
| `light` | kokoro-onnx (not on Python 3.14; see [docs/troubleshooting.md](docs/troubleshooting.md#voice)), faster-whisper | the light local voice (`local_light`): free speech models with no PyTorch and no GPU |
| `voice` | sounddevice, pydub (plus audioop-lts on Python 3.13+) | converting agent audio that is not WAV (pydub, with ffmpeg) |
| `livekit` | livekit, livekit-api, livekit-protocol | the LiveKit adapter |
| `pipecat` | pipecat-ai 1.12.x (capped below 1.13 until a newer release is tested) | the Pipecat adapter |
| `dev` | pytest-asyncio, ruff, mypy | working on Impulse itself |

On Windows, if `torch` or `livekit` fail to import with a missing `vcruntime140_1.dll`, install the latest Microsoft Visual C++ Redistributable (x64).

## A real run

The dry run talks to the tiny echo agent in [examples/sample_agent.py](examples/sample_agent.py) with a scripted caller. It makes zero LLM calls and skips every judge metric and assertion with the reason "dry run". It still computes the deterministic checks, for example UNIV-025, the expected `book_slot` call.

For a real run, tell Impulse which models to use, check the setup, and run:

```bash
impulse init                          # asks for the language and a model per role; writes .impulse/config.toml
impulse doctor examples/config.yaml   # no row should say FAIL (INFO rows are notes); exit code 2 until every FAIL row is fixed
impulse run examples/config.yaml      # exit code 0, 1 (--fail-on condition met) or 2 (config error)
```

What the maintainer has run so far is under [Status](#status).

## Free path quickstart

Everything runs on your machine: no API key is asked for anywhere. The full guide, with hardware notes and troubleshooting, is [docs/run-for-free.md](docs/run-for-free.md). The path with Ollama models is not verified on real hardware yet.

No GPU, or no PyTorch? The light local voice (`pip install -e ".[light]"`, `provider: local_light`) runs speech on the CPU. Its round trip was verified on 2026-10-26 on a laptop with 8 GB RAM: good in English and Spanish, poor speech-to-text in Hindi ([Voice testing](docs/voice-testing.md#light-local-voice-no-pytorch-weak-laptops)).

1. Install [Ollama](https://ollama.com) and pull two models you choose: `ollama pull YOUR_FAST_MODEL` (simulator and fast judge) and `ollama pull YOUR_PRIMARY_MODEL` (primary judge).
2. Install Impulse with the local speech extra: `pip install -e ".[local]"` from a clone, or `pip install "impulse-evals[local] @ git+https://github.com/Impluse-Evals/impulse-eval"` without one.
3. Run `impulse init --profile free`. It asks for the language, an Ollama tag per role, and the Hugging Face TTS and STT model ids, never for a key. A blank answer keeps the `YOUR_..._MODEL` placeholder, for example `YOUR_TTS_MODEL` or `YOUR_STT_MODEL`. It writes `.impulse/config.toml`. When the current folder has no `config.yaml`, it also writes a starter `config.yaml` for your own agent.
4. [examples/free/config.yaml](examples/free/config.yaml) runs in voice mode with `local_hf` speech, and `impulse init` does not change it. Set its `voice_config.tts.model` and `voice_config.stt.model` to your Hugging Face text-to-speech and speech-to-text (faster-whisper) model ids. Or delete its `voice_config` block for a text-mode run. With `YOUR_TTS_MODEL` or `YOUR_STT_MODEL` left in, `impulse doctor` shows a FAIL row for voice TTS or voice STT. `impulse run` then stops with a config error (exit code 2).
5. Try the example agent from the repository root, in one shell:

   ```bash
   export PYTHONPATH=examples           # PowerShell: $env:PYTHONPATH = "examples"
   impulse doctor examples/free/config.yaml
   impulse run examples/free/config.yaml
   ```

## Examples

### Sales agent (no compliance pack)

A generic-adapter agent is any Python function `(user_text, history) -> str | dict`, sync or async. This config tests an outbound sales call in Hindi; the scenario is the ecommerce one from [examples/config.yaml](examples/config.yaml):

```yaml
# impulse:validate
agent:
  type: voice                                 # voice | chat | multimodal
  adapter: generic
  entrypoint: "sales_agent.handler:respond"   # module:function, importable from this folder
  system_prompt: "system_prompt.txt"          # the judge's rubric; relative to this file

judge:
  primary_model: "YOUR_PRIMARY_MODEL"
  fast_model: "YOUR_FAST_MODEL"

simulator:
  model: "YOUR_SIMULATOR_MODEL"
  persona_style: realistic                    # realistic | adversarial | polite

scenarios:
  - name: "sales_lead_followup_hindi"
    language: hi-IN
    accent: north-indian
    domain: ecommerce
    direction: outbound
    max_turns: 9
    persona: "small business owner, interested but price-sensitive, wants ROI before committing"
    expected_outcome: "the caller understands the price and the free trial and agrees to a demo call"
    assertions: []
    metrics: [UNIV-001, UNIV-003, UNIV-005, UNIV-013, UNIV-019, VOICE-027, VOICE-035]
```

`YOUR_..._MODEL` placeholders are resolved from the environment variables `IMPULSE_MODEL_PRIMARY`, `IMPULSE_MODEL_FAST`, `IMPULSE_MODEL_SIMULATOR`, or from `.impulse/config.toml`; you can also write LiteLLM model strings here. Then `impulse doctor config.yaml` and `impulse run config.yaml`. To draft more scenarios from your system prompt, use `impulse generate --prompt system_prompt.txt --domain ecommerce --language hi-IN` (written to `scenarios/generated.yaml`; review them before use).

[examples/config.yaml](examples/config.yaml) holds six scenarios against the same example agent:

- four Indian ones: ecommerce sales, recruitment, healthcare, and an outbound BFSI EMI reminder with RBI KYC, DPDP and TRAI DLT checks;
- a US order-status call in English;
- a Mexican clinic booking in Spanish.

The last two, as written there:

```yaml
scenarios:
  - name: "support_order_status_us"           # en-US, no compliance pack
    language: en-US
    domain: ecommerce
    direction: inbound
    max_turns: 9
    persona: "online shopper whose order is two days late, a little annoyed, wants a clear delivery date and to know whether a refund is possible"
    opening_user_message: "Hi, my order was supposed to arrive on Monday and it still hasn't shown up. Where is it?"
    expected_outcome: "the agent says honestly that it cannot look up orders, invents no order status or delivery date, and offers to transfer the shopper to a human agent"
    assertions: []
    metrics: [UNIV-001, UNIV-003, UNIV-005, UNIV-007, UNIV-019, VOICE-027, VOICE-035]

  - name: "cita_clinica_mx"                   # es-MX: exactly one booking, checked by code (UNIV-025)
    language: es-MX
    domain: healthcare
    direction: inbound
    max_turns: 12
    persona: "paciente que quiere una cita con el médico general el jueves por la mañana; da su nombre y su teléfono cuando se los piden"
    opening_user_message: "Hola, quisiera una cita con el médico general para el jueves, por favor."
    expected_outcome: "la cita queda agendada con book_slot y se le confirma al paciente el día, la hora y su nombre"
    assertions: []
    metrics: [UNIV-001, UNIV-005, UNIV-011, UNIV-019, UNIV-023]
    expected_tool_calls:
      - name: book_slot
        required_arguments: [date, patient_name, phone]
        max_count: 1
```

These two list no compliance pack. When `impulse generate` drafts scenarios, its domain defaults add the India packs only for languages with the region IN. List any pack's rules in `assertions` to check them for any language.

### Talk2Neo (LiveKit, recruitment, DPDP)

Talk2Neo, a recruitment voice agent on LiveKit, is Impulse's own dogfood agent. [examples/talk2neo/config.yaml](examples/talk2neo/config.yaml) joins one new LiveKit room per scenario as a client. It checks DPDP (India's Digital Personal Data Protection Act) consent, purpose disclosure, and the withdrawal path, in English and Hinglish calls:

```yaml
# impulse:validate
agent:
  type: voice
  adapter: livekit
  system_prompt: "system_prompt.txt"     # fill in examples/talk2neo/system_prompt.txt first
  livekit:
    url: "wss://YOUR_LIVEKIT_HOST"
    api_key_env: "LIVEKIT_API_KEY"       # NAME of the env var holding the LiveKit API key
    api_secret_env: "LIVEKIT_API_SECRET" # NAME of the env var holding the LiveKit API secret
    agent_name: null                     # set it when the agent's worker is registered with an agent_name
```

```bash
pip install -e ".[livekit]"
impulse doctor examples/talk2neo/config.yaml
impulse run examples/talk2neo/config.yaml
```

Impulse's access token carries the `can_manage_agent_session` grant. Tool-call capture uses the agent's `lk.agent.session` stream, and so does the system prompt request (sent only when `agent.system_prompt` is not set). Your LiveKit server or project must allow that grant. Otherwise the request gets no answer and no tool calls are captured. Verified on 2026-10-24 and 2026-10-25 with a local `livekit-server --dev` and a LiveKit Agents 1.8.8 agent. Tool calls were captured in text and voice, and `heard_text` on every user turn of the voice runs. LiveKit Cloud is not verified. On a slow machine, see [Troubleshooting](docs/troubleshooting.md#livekit).

### Pipecat

Impulse connects to your running Pipecat bot through Pipecat's own eval transport, and speaks RTVI, Pipecat's client protocol. Your pipeline does not change. The bot needs the `"eval"` entry in its `transport_params`, which Pipecat's example agents already have. Start the bot with the eval transport and copy the URL it prints:

```bash
# check against your Pipecat version
python bot.py -t eval
```

```yaml
# impulse:validate
agent:
  type: voice
  adapter: pipecat
  pipecat:
    mode: eval                     # eval (default) | websocket
    url: "ws://localhost:7860"     # the WebSocket URL your bot prints when started with -t eval
```

```bash
pip install -e ".[pipecat]"
impulse doctor examples/pipecat/config.yaml   # the Pipecat handshake row should pass
impulse run examples/pipecat/config.yaml
```

Pipecat bots serve one client at a time, so Impulse runs Pipecat scenarios one by one. In eval mode, `impulse run` and `impulse doctor` send an `eval-configure` client message after `bot-ready`. It sets FULL function call reporting on the bot, so tool names, arguments, and results reach Impulse. The setting stays on the bot after the run (and after `impulse doctor`). The bot-side checklist (RTVI function call report level FULL, transcriptions, the `get_system_prompt` handler, the websocket fallback) is in [examples/pipecat/README.md](examples/pipecat/README.md) and [docs/adapters.md](docs/adapters.md#pipecat). Verified on 2026-10-24 and 2026-10-25 with a pipecat-ai 1.12.0 bot started with `-t eval`, in text and voice. Tool calls, the system prompt and per-service timings were captured, and `heard_text` in the voice runs. The websocket mode is not verified against a live bot.

## Configuration: `config.toml`

`impulse init` writes `.impulse/config.toml` (and `.impulse/.gitignore`, so the folder stays out of version control). It holds your keys, your models, and your defaults:

```toml
# impulse:validate
[keys]
ANTHROPIC_API_KEY = ""
GEMINI_API_KEY = ""
OPENAI_API_KEY = ""
IMPULSE_VOICE_API_KEY = ""

[models]
primary = "YOUR_PRIMARY_MODEL"
fast = "YOUR_FAST_MODEL"
simulator = "YOUR_SIMULATOR_MODEL"

[model_options.primary]       # [model_options.fast] and [model_options.simulator] have the same shape
api_base = ""
api_key_env = ""
extra_params = {}

[defaults]
reporter = "all"              # json | html | all
language = "en-IN"            # written by impulse init; there is no built-in default language
fail_on = "high"              # critical | high | any
```

- Keys: the environment variable wins, then `[keys]`. Local models (`ollama/...`, `ollama_chat/...`, or any role with `api_base`) need no key. Keys are redacted from logs, reports, and error messages, with one exception for short keys ([Known limitations](docs/known-limitations.md)).
- Models, per role: `config.yaml` (`judge.primary_model`, `judge.fast_model`, `simulator.model`), then `IMPULSE_MODEL_PRIMARY` / `IMPULSE_MODEL_FAST` / `IMPULSE_MODEL_SIMULATOR`, then `[models]`. A `YOUR_..._MODEL` placeholder counts as not set.
- Critical and high compliance checks use the primary model, medium ones the fast model. The primary model string may name 14B parameters or fewer (for example `:7b`). If the run then includes a critical check, Impulse warns you to run `impulse calibrate` before trusting critical verdicts.

## Adding `heard_text` and `tool_calls` to your agent adapter

Two optional fields make the strongest checks possible:

- `heard_text`: what your agent's own speech recognition heard the user say. In voice mode, the agent-hearing word error rate (WER) metrics VOICE-009 and VOICE-013 compare it with what the simulated caller said. Without it they skip with "agent does not expose its transcription; return heard_text from the adapter".
- `tool_calls`: every tool call your agent made in the turn. UNIV-023 (tool call correctness, judged against your system prompt) and UNIV-025 (exact `expected_tool_calls` and `forbidden_tool_calls` per scenario, checked by code) read them.

With the generic adapter, return a dict instead of a string:

```python
# sales_agent/handler.py  (agent.entrypoint: "sales_agent.handler:respond")
SYSTEM_PROMPT = "You are ..."   # optional: used only when config.yaml sets no agent.system_prompt


async def respond(user_text, history):
    # user_text: what the simulated caller said ("" when the agent opens an outbound call)
    # history: the earlier turns, [{"role": "user" | "assistant", "content": "..."}]
    result = await my_agent.handle(user_text, history)   # your agent
    return {
        "text": result.reply,
        "heard_text": result.transcript,                   # optional
        "tool_calls": [                                    # optional, one dict per call in this turn
            {
                "name": "book_slot",
                "arguments": {"date": "2026-01-01", "phone": "0000000000"},
                "result": {"booking_id": "B-1"},
                "success": True,                           # None = not captured
                "error": None,
            }
        ],
    }
```

A sync function works too. The dict may also carry `audio`, the WAV bytes of the agent's reply, for the audio metrics in voice mode.

[examples/sample_agent.py](examples/sample_agent.py) is a complete agent that returns a dummy `book_slot` call whenever the caller says "book".

The LiveKit and Pipecat adapters capture both fields without code changes:

- Pipecat, from the bot's RTVI user transcriptions and function call messages;
- LiveKit, from the agent's transcription streams and its `lk.agent.session` tool events.

Both were verified with live agents in the 2026-10-24 and 2026-10-25 test campaigns, with `heard_text` on every user turn of the voice runs.

For any other stack, write your own adapter. Subclass `AgentInterface` from `impulse_evals.adapters.base`, and return `AgentReply(text=..., latency_ms=..., heard_text=..., tool_calls=[ToolCall(...)])` from its `send()`. `ToolCall` is in `impulse_evals.plugin_types`. Load your adapter from `config.yaml` with `adapter: custom` and `entrypoint: "module:Class"` ([examples/custom_adapter/](examples/custom_adapter/echo_adapter.py)). Or pass it from Python to the async `impulse_evals.runner.run(config, adapter_factory=...)`. It returns the run result without writing `run.json` or `report.html`. In voice mode it still writes the `audio/` files under `.impulse/runs/<run_id>/`. An agent in another language can answer JSON over HTTP instead, with `adapter: http` ([examples/http/README.md](examples/http/README.md)); its answer carries `heard_text` and `tool_calls` the same way. See [docs/adapters.md](docs/adapters.md).

## CI and regression gates

- `impulse run config.yaml --fail-on high` (the default) exits 1 on any critical or high assertion failure, any scenario error, or a poor UNIV-019 (a credential request, false urgency, a misleading claim) or UNIV-025 (a wrong, missing or duplicate tool call), so these block a merge in any language without a compliance pack. `--fail-on critical` gates on critical failures and errors only; `--fail-on any` on any failed scenario. Exit code 2 means a usage or config error.
- `impulse compare RUN_A RUN_B` diffs two runs, given as run IDs, run folders, or `run.json` paths. It writes `compare_<A>_vs_<B>.json` and `.html` to `.impulse/runs/`. It exits 1 when a critical assertion newly fails, or a scenario newly errors, in the second run. Any new scenario error counts, environment errors included (an agent worker that was down).
- `impulse report RUN_ID` rewrites a run's `report.html`; `--no-embed-audio` links the `audio/` files instead of embedding them.
- `impulse convert-failure .impulse/runs/<run_id>/scenarios/<name>.json` turns a failed scenario into a regression scenario under `scenarios/regression/`.
- From pytest: `impulse_check("config.yaml")` runs a config, writes its run folder, and returns the result. `assert_no_critical(result)` fails the test on critical failures or scenario errors. Both are in `impulse_evals.testing`.

## Privacy

Impulse prints this on the first real `impulse run` in a folder:

> Impulse sends your system prompt and transcripts to the LLM and voice providers you configured.

That is exactly what happens. On the free local path (Ollama, and `local_hf` or `local_light` speech), your data stays on your machine. Any remote LLM or voice provider receives the system prompt and the transcripts. Impulse itself sends no telemetry and never pays for your LLM calls. Keys are redacted from logs, reports, and error messages, with one exception for short keys ([Known limitations](docs/known-limitations.md)). Run folders under `.impulse/` (transcripts, audio, reports) are git-ignored by default. Compliance results are technical signals only, not legal certification. More in [docs/privacy-and-security.md](docs/privacy-and-security.md).

## Licence

- All of Impulse Evals, including the plugin API and the compliance packs (`dpdp`, `rbi_kyc`, `trai_dlt`, `hipaa`), is under the Apache License 2.0 (Apache-2.0, [LICENSE](LICENSE)). Copyright VITAF Technologies ([NOTICE](NOTICE)).
- You are free to use, modify, and embed it, and to offer it as a service, including commercially. Keep the `LICENSE` and `NOTICE` files when you redistribute it.
- Trademark: the licence does not grant rights to the name "Impulse" or the Impulse logo.
- Releases already published on PyPI keep the licence they were published under.

[LICENSING.md](LICENSING.md) explains this in plain language.

"Need help setting up custom adapters, CI pipelines, or DPDP / RBI KYC / HIPAA compliance suites for your voice agents? Reach out at **shrikant.s@testvoiceai.live "

## Status

Version 0.1.0, the first public release (alpha). What the maintainer has run, with dates and versions, is in [docs/verified-setups.md](docs/verified-setups.md):

- Verified: real runs with hosted Gemini models against a LiveKit agent and a Pipecat `-t eval` bot, in text and voice, on 2026-10-24 and 2026-10-25. Voice used Deepgram through a local proxy.
- Verified on 2026-10-26: the light local voice round trip, Deepgram speech-to-text through `provider: litellm`, and the HTTP adapter against `examples/http/server.py` on localhost.
- Verified offline: the test suite, the dry run of `examples/config.yaml`, and noise simulation's mixer, config, report and dry run.
- Not verified: the free path with Ollama models, `local_hf` speech, LiveKit Cloud, noise simulation against a live agent, and any other model or provider.

What is planned next is in the [Roadmap](ROADMAP.md).

## Contributing

Contributions are welcome. Start with [CONTRIBUTING.md](CONTRIBUTING.md) and the [Contributing](docs/contributing.md) page: set up with `pip install -e ".[dev]"`, run `python -m pytest -q`, and follow the rules there. The most useful help right now is [testing noise simulation](docs/contributing.md#help-us-test-noise-simulation) on your own voice agent; testers are added to [CONTRIBUTORS.md](CONTRIBUTORS.md). Report security problems privately ([SECURITY.md](SECURITY.md)), and follow the [Code of Conduct](CODE_OF_CONDUCT.md).

## Documentation

The [docs home](docs/index.md) has the full table of contents.

- [Overview](docs/index.md)
- [Getting started](docs/getting-started.md)
- [Reading the report](docs/reading-the-report.md): every part of the HTML report, its players and timeline
- [Concepts](docs/concepts.md)
- [Run Impulse for free](docs/run-for-free.md)
- [Configuration](docs/configuration.md): `config.yaml` and `config.toml`
- [CLI reference](docs/cli.md): every command, flag, and exit code
- [Metrics](docs/metrics.md): universal, voice, and chat metrics, including UNIV-023 and UNIV-025
- [Compliance packs](docs/compliance-packs.md): DPDP, RBI KYC (Know Your Customer), TRAI DLT, and the opt-in HIPAA pack
- [Voice testing](docs/voice-testing.md): speech providers, and noise simulation (experimental)
- [Adapters](docs/adapters.md): generic, LiveKit, Pipecat, HTTP, and writing your own (custom)
- [Writing scenarios](docs/writing-scenarios.md)
- [CI integration](docs/ci-integration.md)
- [Calibration](docs/calibration.md)
- [Cookbook](docs/cookbook.md): ecommerce, Talk2Neo, healthcare, BFSI
- [Troubleshooting](docs/troubleshooting.md)
- [Known limitations](docs/known-limitations.md): every documented limitation, with a workaround
- [Privacy and security](docs/privacy-and-security.md)
- [Architecture](docs/architecture.md)
- [Contributing](docs/contributing.md): including [help us test noise simulation](docs/contributing.md#help-us-test-noise-simulation); testers are listed in [CONTRIBUTORS.md](CONTRIBUTORS.md)
- [FAQ](docs/faq.md)
- [Verified setups](docs/verified-setups.md)
- [Changelog](docs/changelog.md)
- [Roadmap](ROADMAP.md): what is planned after 0.1.0
- [Claude skill](docs/skill.md): let Claude set up, run and read Impulse for you
- [Licensing](LICENSING.md)
