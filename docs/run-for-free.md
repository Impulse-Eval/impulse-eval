<!-- SPDX-License-Identifier: Apache-2.0 -->
# Run Impulse for free

This guide runs Impulse with no API key: Ollama serves the language models, and open speech models run inside the Impulse process.

## Quick version

- **What it is.** The judge, the simulated caller, text-to-speech (TTS) and speech-to-text (STT) all run on your machine. Your system prompt and transcripts never leave it.
- **Status: not verified end to end.** The full free path has not been run with real Ollama and Hugging Face models. Only the offline checks marked "verified" below were run (Windows, Python 3.14).
- **Prerequisites:** Requires Python 3.11 or newer (tested on 3.11, 3.12 and 3.14). Python 3.10 is not supported. You also need a clone of the repository and [Ollama](https://ollama.com).
- **The commands,** from the root of the clone, in a fresh virtual environment:

```bash
ollama pull YOUR_FAST_MODEL          # simulator and fast judge
ollama pull YOUR_PRIMARY_MODEL       # primary judge
pip install -e ".[local]"            # without a clone: pip install "impulse-evals[local] @ git+https://github.com/Impulse-Eval/impulse-eval"
impulse init --profile free          # asks for the language and the Ollama tags, never for a key
export PYTHONPATH=examples           # PowerShell: $env:PYTHONPATH = "examples"
impulse doctor examples/free/config.yaml
impulse run examples/free/config.yaml --dry-run
```

- **What success looks like.** `impulse doctor` ends with `All checks passed.` (not verified with real models). The dry run needs no model and prints this (verified on Windows):

```text
Run 20261025T233345Z-a0fcb4 (dry run): 2 scenarios, 2 passed, 0 failed (pass rate 100.0%)
  PASS  free_clinic_booking
  PASS  free_opening_hours_question
Warnings:
  - Dry run: judge metrics and assertions were skipped; results are not verdicts.
Wrote .impulse\runs\20261025T233345Z-a0fcb4\run.json
Wrote .impulse\runs\20261025T233345Z-a0fcb4\report.html
Dry run: --fail-on is not applied.
```

- **Then** make a real text run (step 6) and a voice run (step 7).
- **Weak laptop?** Use the [light local voice](#light-local-voice-no-pytorch-weak-laptops) instead of `local_hf`. It needs no PyTorch and no GPU. On the maintainer's 8 GB laptop it used about 1.4 GB of RAM.

## Details

### 1. Prerequisites

Requires Python 3.11 or newer (tested on 3.11, 3.12 and 3.14). Python 3.10 is not supported.

- **A clone and a fresh virtual environment.** The example agent and configs live in `examples/`. Run every command on this page from the repository root.
- **Disk space (not verified).** Ollama and Hugging Face download model weights on first use. The space depends on the models you pick; check each model's page first.
- **RAM (not verified).** In voice mode, the Ollama models, the TTS model and the STT model run at the same time. Pick sizes your RAM (or GPU memory) can hold together.
- **GPU (optional, not verified).** Everything runs on the CPU, slowly. A GPU speeds up Ollama and the speech models (step 8).
- **ffmpeg is not needed.** Impulse reads and writes WAV audio itself. Only pydub (the `voice` extra) uses ffmpeg, to convert agent audio that is not WAV.

Success: `python --version` in your virtual environment prints Python 3.11 or newer.

### 2. Install Ollama and pull two models

Install Ollama from [ollama.com](https://ollama.com). Then pull two models of your choice from the Ollama library:

```bash
ollama pull YOUR_FAST_MODEL      # simulator and fast judge
ollama pull YOUR_PRIMARY_MODEL   # primary judge
```

- Replace both placeholders with the Ollama tags you chose. Impulse does not recommend specific models.
- Pick a small, fast model for the simulated caller and the fast judge, and a stronger one for the primary judge.
- The primary judge decides critical and high compliance verdicts. Impulse may warn you to run [calibration](calibration.md) first. It does so when both hold:
  - the primary model string names 14B parameters or fewer (for example a tag ending in `:7b`);
  - a scenario of the run has a critical assertion.

Success: `ollama list` shows both models.

### 3. Install Impulse with the local extra

Straight from GitHub, with no clone (not verified with the `local` extra):

```bash
pip install "impulse-evals[local] @ git+https://github.com/Impulse-Eval/impulse-eval"
```

To work from your clone, install from the root of the clone (not verified with the `local` extra):

```bash
pip install -e ".[local]"
```

The `local` extra adds faster-whisper, transformers and soundfile. It does not pin `torch` on purpose. Install the CPU or GPU build of PyTorch that fits your machine, following PyTorch's own instructions (not verified).

Success: the install ends without errors. In step 5, the voice rows of `impulse doctor` no longer say the `local` extra is missing.

### 4. Run impulse init --profile free

Run it from the repository root. Impulse reads `.impulse/config.toml` from the current folder.

```bash
impulse init --profile free
```

It asks for:

1. the default language, as a code such as `en-US`, `es-MX`, `en-IN` or `hi-IN` (Impulse never assumes one);
2. an Ollama tag for each role: primary judge, fast judge, and simulator (the simulated caller);
3. a Hugging Face TTS model id and a faster-whisper STT model id.

- Enter tags as Ollama shows them. Impulse turns each into the LiteLLM model string `ollama/<tag>` and checks it offline.
- A blank answer keeps the `YOUR_..._MODEL`, `YOUR_TTS_MODEL` or `YOUR_STT_MODEL` placeholder.
- It never asks for a key.

It then writes `.impulse/config.toml` and `.impulse/.gitignore`, which keeps the folder out of version control. When the current folder has no `config.yaml`, it also writes a starter one for your own agent. When one exists, it prints the `voice_config` block to add instead. Finally it sends a 5-token ping to each model; `--no-ping` skips that.

`impulse init` does not change `examples/free/config.yaml`; steps 6 and 7 edit it.

Success (not verified with real models): it prints `Wrote .impulse/config.toml (profile free, language en-IN) and .impulse/.gitignore` (with your language), a `PASS` row per pinged model, and `Next: impulse doctor`. Verified: `impulse init --profile free --non-interactive --language en-IN` prints these `Wrote` lines (with `\` in Windows paths). That form asks nothing, pings nothing, and keeps every placeholder.

### 5. Run impulse doctor and fix every red row

The example config imports the tiny example agent `examples/sample_agent.py`. Put `examples/` on the Python path first, in the shell you run Impulse from:

```bash
export PYTHONPATH=examples           # PowerShell: $env:PYTHONPATH = "examples"
impulse doctor examples/free/config.yaml
```

`impulse doctor` checks Python, `.impulse/config.toml`, the config, each model, the agent entrypoint, and the voice providers. For each model it checks that it resolves, that LiteLLM knows it, and that it answers a 5-token ping. Red rows say `FAIL`; `INFO` rows are notes.

With every model left as a placeholder and without the `local` extra, the output looks like this. Verified on Windows after `impulse init --profile free --non-interactive --language en-IN`; `<clone>` stands for the clone's path:

```text
impulse doctor
PASS  python            Python 3.14.7
PASS  config.toml       .impulse\config.toml loaded
PASS  config            examples\free\config.yaml is valid
FAIL  primary model     judge.primary_model: no model configured for the primary role. Set judge.primary_model in config.yaml, the environment variable IMPULSE_MODEL_PRIMARY, or [models] primary in .impulse/config.toml.
FAIL  fast model        judge.fast_model: no model configured for the fast role. Set judge.fast_model in config.yaml, the environment variable IMPULSE_MODEL_FAST, or [models] fast in .impulse/config.toml.
FAIL  simulator model   simulator.model: no model configured for the simulator role. Set simulator.model in config.yaml, the environment variable IMPULSE_MODEL_SIMULATOR, or [models] simulator in .impulse/config.toml.
PASS  agent entrypoint  imported sample_agent:respond
INFO  system prompt     from <clone>\examples\system_prompt.txt
FAIL  voice TTS         voice_config.tts.provider: local_hf needs the 'local' extra: pip install impulse-evals[local]
FAIL  voice STT         voice_config.stt.provider: local_hf needs the 'local' extra: pip install impulse-evals[local]
5 check(s) failed. Fix every FAIL row, then run impulse doctor again.
```

Fix each row as it says; step 10 lists the common causes. While the config holds `YOUR_TTS_MODEL` and `YOUR_STT_MODEL`, the voice rows stay red. They say "no TTS model for language en-IN" and "no STT model set". Either do step 6 first (text mode has no voice rows), or set the speech model ids now (step 7).

Success (not verified with real models): the last line is `All checks passed.` and the exit code is 0. It is 2 while any row says `FAIL`.

### 6. Make a text-mode run

Text mode means the config has no `voice_config` block. Delete that block from `examples/free/config.yaml`: from the `voice_config:` line down to `timeout_s: 30`. Then run:

```bash
impulse run examples/free/config.yaml
```

- The two scenarios (a clinic booking and an opening-hours question) run against the example agent.
- On the first real run in a folder, Impulse prints its privacy notice:
  "Impulse sends your system prompt and transcripts to the LLM and voice providers you configured."
  With Ollama, that provider is your own machine.
- `git restore examples/free/config.yaml` brings the `voice_config` block back for step 7.

A quick offline check, before or without Ollama (verified): `impulse run examples/free/config.yaml --dry-run`. It validates the config, always runs in text mode, and makes zero LLM calls. Its output is in the [Quick version](#quick-version). The pass count is not a verdict: no judge check ran.

Success (not verified with real models): the summary line says `(text mode)` and one `PASS` or `FAIL` line follows per scenario. The HTML report is written to `.impulse/runs/<run_id>/report.html`. The exit code is 0, or 1 when a `--fail-on` condition is met ([CLI reference](cli.md)).

### 7. Make a voice-mode run with local_hf

Put the `voice_config` block back (step 6). Replace the two placeholders with your model ids:

- `YOUR_TTS_MODEL`: a Hugging Face text-to-speech model id;
- `YOUR_STT_MODEL`: a faster-whisper model size or Hugging Face model id.

The block as shipped in `examples/free/config.yaml`:

```yaml
# impulse:validate
voice_config:
  tts:
    provider: local_hf
    model: "YOUR_TTS_MODEL"         # a Hugging Face text-to-speech model id
    models_by_language: {}          # optional per-language override, e.g. {hi-IN: "YOUR_TTS_MODEL"}
    voice: null
    device: auto                    # auto | cpu | cuda
    base_url: null
    api_key_env: null
  stt:
    provider: local_hf
    model: "YOUR_STT_MODEL"         # a faster-whisper model id
    device: auto
    compute_type: auto
    base_url: null
    api_key_env: null
  save_audio: true
  save_recording: true
  timeout_s: 30
```

- `models_by_language` picks a TTS model per scenario language. A TTS model usually speaks one language.
- The STT model receives the primary language subtag: `hi-IN` becomes `hi`.

Then check and run:

```bash
impulse doctor examples/free/config.yaml
impulse run examples/free/config.yaml
```

What happens:

- The first voice run downloads the speech models from the Hugging Face Hub.
- Each caller turn is synthesized and sent to the agent.
- Impulse also transcribes its own caller audio. This "harness round trip" checks Impulse's own TTS and STT, not your agent (VOICE-011, informational).
- Per-turn WAV files and a stereo recording of the call (caller left, agent right) are saved under `.impulse/runs/<run_id>/audio/`. The per-turn files are `<scenario>_t<N>_user.wav`, plus `<scenario>_t<N>_agent.wav` when the agent returned audio. The recording is `<scenario>_recording.wav`.

The example agent replies with text only. So VOICE-001, VOICE-003 and VOICE-005 (latency) run as labelled proxies, on the measured latency. Their note says "(proxy: Turn.latency_ms; the agent returned no audio)". The metrics that need the agent's audio skip with "agent returned no audio". An agent that returns `audio` gets the full audio metrics ([Voice testing](voice-testing.md)).

Success (not verified): `impulse doctor` shows `PASS  voice TTS  local_hf: 'local' extra installed; model <your id> loads on the first run`, and the same for voice STT. The run's summary line says `(voice mode)`, and the audio files exist.

#### Light local voice (no PyTorch, weak laptops)

When `local_hf` is too heavy for your machine, use `provider: local_light`. It is free too. Kokoro TTS (an ONNX model) and faster-whisper STT run in the Impulse process. It needs no PyTorch, no GPU, no server and no key.

```bash
pip install -e ".[light]"     # from a clone; without one: pip install "impulse-evals[light] @ git+https://github.com/Impulse-Eval/impulse-eval"
```

```yaml
# impulse:validate
voice_config:
  tts:
    provider: local_light
    model: null      # null = download the Kokoro files to the cache folder on first use
    voice: null      # null = the default voice for the scenario language
  stt:
    provider: local_light
    model: null      # null = faster-whisper "small"
```

Measured on the maintainer's PC on 2026-10-26. These are measurements on one PC, not a guarantee:

| what | measured |
|---|---|
| PC | 8 GB RAM, Windows 11, Python 3.12 |
| RAM | about 1.3 to 1.4 GB for TTS and STT together |
| GPU | not needed and not used (STT runs on the CPU by default) |
| first download | about 354 MB of Kokoro files (checksum-verified) and about 484 MB for Whisper `small` |
| model load | about 30 s, once per run |
| one caller turn, whole run | about 40 to 60 s of wall time |
| Impulse's own TTS per turn | 2 to 8 s, by sentence length |
| Impulse's own STT per pass | about 8 to 12 s |

The 40 to 60 s per turn includes the simulator call and the test agent's own Kokoro speech. Impulse runs one STT pass on its own caller audio, and one more on the agent's audio when there is some.

Languages:

- **en-US and es-MX work well** (measured; VOICE-011 0.000 and 0.048).
- **hi-IN: do not use Whisper `small` for STT** (VOICE-011 0.88, poor). For Hindi, keep `local_light` TTS and use Deepgram STT through `provider: litellm` ([example](voice-testing.md#litellm)).
- **French, Italian, Portuguese and Chinese** have default Kokoro voices but were not measured.
- **Japanese** works only with an explicit `j*` voice and kana-only text.

**Python 3.14.** The `light` extra skips kokoro-onnx there. Impulse then stops with this message, which gives the fix:

```text
voice_config.tts.provider: local_light needs the 'light' extra: pip install impulse-evals[light] (kokoro-onnx 0.6 supports Python 3.11 to 3.13; on 3.14 install it with: pip install kokoro-onnx==0.6.1 --ignore-requires-python)
```

`IMPULSE_CACHE_DIR` sets where the Kokoro files go. All details: [Voice testing](voice-testing.md#light-local-voice-no-pytorch-weak-laptops).

#### When local speech is too slow

You can keep the judge and the caller on Ollama and move one side to a hosted service with `provider: litellm`. That side leaves the free, local path: it needs the provider's key and sends audio to it. For example, STT through Deepgram (verified on 2026-10-26 in English, Spanish and Hindi):

```yaml
# impulse:validate
voice_config:
  tts:
    provider: local_hf
    model: "YOUR_TTS_MODEL"
  stt:
    provider: litellm
    model: "deepgram/nova-3"        # needs DEEPGRAM_API_KEY (environment or [keys] in .impulse/config.toml)
```

The key name follows LiteLLM's convention for the provider; no shim or proxy is needed. More providers and their limits: [Voice testing](voice-testing.md#litellm).

### 8. Hardware notes

None of these notes is verified on real hardware yet, except the light voice measurements in step 7.

- **CPU versus GPU.** Local models on a CPU are slow. One voice turn runs TTS for the caller, then the agent. Then come an STT pass over Impulse's own audio, and one over the agent's audio when it returns some. Each scenario then makes one judge call per judge-based check. Expect a CPU voice run to take much longer than the conversation itself.
- **Latency metrics are not affected.** They never include Impulse's own TTS or STT time.
- **`device`.** `voice_config.tts.device: auto` asks torch for CUDA, then Apple MPS, then uses the CPU. Set `cpu` or `cuda` to choose.
- **`compute_type`.** `voice_config.stt.device` and `compute_type` (both `auto` by default) are passed to faster-whisper as they are.
- **One at a time.** Each speech provider loads its model once per run and does one synthesis or transcription at a time. Keep `--concurrency` at its default of 1 on a small machine.
- **Ollama.** Ollama decides where the language models run. A judge prompt holds the system prompt and the whole transcript. When you set `num_ctx` in a role's options (for example `num_ctx: 8192` under `primary_options`), Impulse warns when a prompt is estimated above 80% of it. The value is also passed to LiteLLM; whether it reaches Ollama is not verified.

### 9. Test tool calls with the mock in sample_agent.py

`examples/sample_agent.py` is a tiny echo agent with a built-in tool-call mock. When the caller's text contains the whole word "book" (any case), the reply carries one dummy `book_slot` call. The call has fixed arguments (`date`, `patient_name`, `phone`) and `success: true`.

That lets you try two checks without a real backend: UNIV-023 (tool call correctness, judged) and UNIV-025 (expected tool calls, checked by code). The `free_clinic_booking` scenario in `examples/free/config.yaml` expects that call:

```yaml
    expected_tool_calls:
      - name: book_slot
        required_arguments: [date, patient_name, phone]
        max_count: 1
```

In a dry run, the scripted caller sends the scenario's `opening_user_message` (or "Hello"), then "I'd like to book". So the mock fires and UNIV-025 is `good` (verified).

To see failures, copy the mock and change the copy; leave `sample_agent.py` as it is:

1. Copy `examples/sample_agent.py` to `examples/free/mock_agent.py`. The config file's folder is on the import path, so it needs no `PYTHONPATH`.
2. Edit the copy's `_BOOK_SLOT_TOOL_CALL` dictionary:
   - failed booking: set `"result": {"success": False, "error": "slot no longer available"}` and `"success": False`, and add `"error": "slot no longer available"`;
   - missing argument: delete `"phone"` from `"arguments"`;
   - claimed action with no call: return the reply text without the `tool_calls` key.
3. Point the config at it: `entrypoint: "mock_agent:respond"`.
4. Run `impulse run examples/free/config.yaml --dry-run` (UNIV-025 needs no LLM), then a real run for UNIV-023.

Success:

- Failed booking (verified): the dry run prints a `FAIL` line for `free_clinic_booking` ending in `UNIV-025 poor`. The UNIV-025 reason is `tool book_slot call failed: slot no longer available`.
- Missing argument (not run): `tool book_slot is missing required arguments: phone`.
- No call at all (not run): `expected tool book_slot was never called`.
- These two texts are UNIV-025's exact reason formats. In a real run, UNIV-023 asks the judge whether the agent claimed a booking that failed or never happened (not verified).

### 10. Troubleshooting

- **`local_hf needs the 'local' extra: pip install impulse-evals[local]`** (doctor voice rows, or a config error from `impulse run`). Install the extra into the same virtual environment: `pip install -e ".[local]"` from the clone.
- **`local_light needs the 'light' extra: pip install impulse-evals[light]`.** Run `pip install -e ".[light]"`. On Python 3.14, also run the kokoro-onnx command the message names.
- **`voice_config.tts.provider: local_hf TTS needs PyTorch, which the 'local' extra does not install: pip install torch (see https://pytorch.org/get-started/locally/)`.** The extra is installed but torch is not. Install the PyTorch build that fits your machine. The run stops before its first model call; `impulse doctor` shows it on the voice TTS row. STT (faster-whisper) works without torch.
- **`voice_config.tts.provider: local_hf TTS needs PyTorch, which is installed but could not be imported (...).`** Torch is installed but its import failed; the part in parentheses is the error. On Windows the message adds the Visual C++ Redistributable hint (see "Windows, missing DLL" below).
- **`no model configured for the primary role`** (or fast, or simulator). `impulse init --profile free` was skipped or answered blank. Run it again, set `IMPULSE_MODEL_PRIMARY` / `IMPULSE_MODEL_FAST` / `IMPULSE_MODEL_SIMULATOR`, or write `ollama/<tag>` strings into the config.
- **Ollama not running.** The doctor model rows say `not reachable:` and a connection error. Start Ollama (its desktop app, or `ollama serve`), then run `impulse doctor` again.
- **Model not pulled.** The doctor model row says `not reachable:` and Ollama's error. Run `ollama pull <tag>` for the tag in `.impulse/config.toml` (without the `ollama/` prefix), then check `ollama list`.
- **A failed row names `openaipublic.blob.core.windows.net`.** This is not an Ollama problem. LiteLLM's tokenizer file needs a one-time download on this machine. Connect to the internet once, or set `CUSTOM_TIKTOKEN_CACHE_DIR` to a folder holding the file. Why it happens: [Troubleshooting](troubleshooting.md#models-and-keys).
- **`no TTS model for language en-IN`** or **`no STT model set`.** `YOUR_TTS_MODEL` or `YOUR_STT_MODEL` is still in `examples/free/config.yaml` (step 7), or `models_by_language` has no model for that language.
- **`agent.entrypoint: cannot import module sample_agent`.** `examples/` is not on `PYTHONPATH` in this shell (step 5).
- **ffmpeg.** Only needed for agent audio that is not WAV, through pydub (`voice` extra). Impulse's own audio, and every WAV, works without it.
- **Windows, missing DLL.** Cause: `torch` or `livekit` fails to import with a missing `vcruntime140_1.dll`. Fix: install the latest Microsoft Visual C++ Redistributable (x64), then open a new shell. For torch, `impulse doctor` and `impulse run` say "installed but could not be imported" with a DLL load error.
- **Slow runs or timeouts.** See step 8. Local models on a CPU are slow, and the first voice run also downloads the speech models.

More messages and fixes: [Troubleshooting](troubleshooting.md). Limits that have no fix yet: [Known limitations](known-limitations.md).

## Next steps

- [Getting started](getting-started.md): the same flow with any model provider.
- [Voice testing](voice-testing.md): the voice pipeline and what each audio metric needs.
- [Reading the report](reading-the-report.md): what the HTML report of your first run shows.
- [Writing scenarios](writing-scenarios.md): scenarios for your own agent.
- [Verified setups](verified-setups.md): add a row once you have run this guide on your machine.

---

<!-- docs-nav -->
[← Previous: Getting started](getting-started.md) | [Docs home](index.md) | [Next: Concepts →](concepts.md)
