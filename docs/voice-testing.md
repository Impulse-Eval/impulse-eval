<!-- SPDX-License-Identifier: Apache-2.0 -->
# Voice testing

Voice mode tests your agent with real audio: the simulated caller's words become speech, and your agent's audio is transcribed back to text.

## Quick version

- **What it is.** Add a `voice_config` block to `config.yaml` and the run goes through audio. Text-to-speech (TTS) voices the caller. Speech-to-text (STT) transcribes the audio.
- **Pick a speech provider** for each side (TTS and STT can differ):

| you want | provider | cost and key |
|---|---|---|
| free, light, any laptop | [`local_light`](#light-local-voice-no-pytorch-weak-laptops) | free, no key, no PyTorch |
| free, any Hugging Face model | [`local_hf`](#local_hf) | free, no key, needs PyTorch for TTS |
| Deepgram, Groq, OpenAI and others | [`litellm`](#litellm) | the provider's key |
| your own speech server | [`openai_compatible`](#openai_compatible) | optional key |

- **The config most people start with** (free, verified on 2026-10-26 in en-US and es-MX):

```bash
pip install -e ".[light]"     # from a clone; without one: pip install "impulse-evals[light] @ git+https://github.com/Impulse-Eval/impulse-eval"
```

```yaml
# impulse:validate
voice_config:
  tts:
    provider: local_light    # Kokoro text-to-speech, no key
  stt:
    provider: local_light    # faster-whisper "small" speech-to-text, no key
```

- **Hindi?** Keep `local_light` for TTS and use Deepgram for STT through [`litellm`](#litellm).
- **Noisy callers?** Add `noise: {profile: pink, snr_db: 10}` to `voice_config` ([noise simulation](#noise-simulation-experimental), experimental).
- **What success looks like.** `impulse doctor CONFIG` shows `PASS` on its `voice TTS` and `voice STT` rows (for `local_light`: not verified here). The run's summary line says `(voice mode)`. Audio files appear under `.impulse/runs/<run_id>/audio/`.
- **Is the harness healthy?** The "harness" is Impulse's own TTS-then-STT loop, used to check itself. VOICE-011 is its word error rate (WER). When it is poor, the run warns and the other audio metrics are unreliable.

Status: the voice pipeline is covered by the test suite with a mock voice server. Real voice runs were made in two ways ([Verified setups](verified-setups.md)):

- with `openai_compatible` against a local Deepgram proxy (the campaigns of 2026-10-24 and 2026-10-25);
- with the light local voice, `local_light` (2026-10-26).

The `litellm` provider was checked with Deepgram STT on one clip per language; no full run used it. Other speech models are not verified.

## Details

### Turning voice mode on

Voice mode is on when `config.yaml` has a `voice_config` block. With no block, the run is in text mode. The switch is per config file, not per scenario. `impulse run --dry-run` always uses text mode.

Impulse uses a "cascade": the simulator writes the caller's words as text, and Impulse turns them into speech. Per user turn in voice mode:

1. The simulator model writes the caller's next sentence.
2. TTS turns it into audio, in the scenario's language. With [noise simulation](#noise-simulation-experimental) (experimental), background noise is mixed in now.
3. The adapter sends the audio to your agent. It returns the agent's reply, its audio when there is some, and the latency it measured. The generic adapter passes the text; a handler that declares a `user_audio` parameter also gets the audio ([Adapters](adapters.md#generic-adapter)).
4. Impulse runs STT on its own TTS audio of the caller's turn (always the clean audio, without noise): the harness round trip, VOICE-011.
5. When the agent returned audio, Impulse runs STT on it for word timings and a confidence.

The agent turn's text is the adapter's reported text when it is not empty. Otherwise it is Impulse's STT text of the agent's audio. The simulated caller sees the same text.

### Speech providers

`voice_config.tts.provider` and `voice_config.stt.provider` each take one of these providers:

| provider | runs where | needs | notes |
|---|---|---|---|
| `local_hf` | in the Impulse process | the `local` extra (`pip install -e ".[local]"`), a model download from the Hugging Face Hub on first use, optional GPU | the free default: no server, no key |
| `local_light` | in the Impulse process | the `light` extra (`pip install -e ".[light]"`), about 354 MB of model files on first use plus the Whisper model | the light local voice: no PyTorch, no GPU, no key ([below](#light-local-voice-no-pytorch-weak-laptops)) |
| `litellm` | any speech provider LiteLLM supports (Deepgram, Groq, OpenAI, Azure, ...) | `model` (a LiteLLM model string, required) and the provider's key | no extra: LiteLLM is a core dependency ([below](#litellm)) |
| `openai_compatible` | any server with the OpenAI audio API shape | `base_url`, optionally a key | works with local servers or paid APIs |
| `impulse_hosted` | an Impulse-hosted voice API | `base_url` (required, no default) and `IMPULSE_VOICE_API_KEY` (or the variable named by `api_key_env`) | an optional convenience, never required |

#### local_hf

The block from [examples/free/config.yaml](../examples/free/config.yaml):

```yaml
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

- **TTS** loads a transformers `text-to-speech` pipeline for `model`, or for `models_by_language[<scenario language>]` when set. It writes a mono 16-bit WAV at the model's sampling rate.
- **`device: auto`** asks torch for CUDA, then Apple MPS, then uses the CPU.
- **STT** loads a faster-whisper model: a model size or a Hugging Face model id. `device` and `compute_type` are passed as they are. It mixes the audio to mono 16 kHz and transcribes with word timestamps. The confidence is the mean word probability.
- Models load once per run and run in a worker thread, one call at a time per provider.
- `YOUR_TTS_MODEL` or `YOUR_STT_MODEL` left in place is a config error. The messages:
  "voice_config.tts.model: no TTS model for language en-IN. ..." or
  "voice_config.stt.model: no STT model set. ...".
- Without the extra: "local_hf needs the 'local' extra: pip install impulse-evals[local]".

TTS also needs PyTorch, which the extra does not install. STT needs no torch.

- **No torch.** The run stops before its first model call with this message:
  "voice_config.tts.provider: local_hf TTS needs PyTorch, which the 'local' extra does not install: pip install torch (see https://pytorch.org/get-started/locally/)".
  `impulse doctor` shows the same text on its voice TTS row.
- **Torch that fails to import.** The run stops the same way, with this message:
  "voice_config.tts.provider: local_hf TTS needs PyTorch, which is installed but could not be imported ({error})."
  On Windows the message adds a hint for DLL load errors, such as a missing `vcruntime140_1.dll`.
  The hint: install or update the Microsoft Visual C++ Redistributable (x64).
- Impulse tries the torch import once, in a separate Python process. So a torch that crashes Python cannot end the run or the doctor. This can take a few seconds.

#### Light local voice (no PyTorch, weak laptops)

`provider: local_light` runs Kokoro text-to-speech (an ONNX model) and faster-whisper speech-to-text in the Impulse process. It needs no PyTorch, no GPU, no server and no key. Use it when `local_hf` is too heavy for your machine.

Install: `pip install "impulse-evals[light] @ git+https://github.com/Impulse-Eval/impulse-eval"`, or `pip install -e ".[light]"` from a clone. On a fresh Python 3.12 environment this installed kokoro-onnx 0.6.1, faster-whisper 1.2.1 and onnxruntime 1.31.0, and no PyTorch (verified).

```yaml
# impulse:validate
voice_config:
  tts:
    provider: local_light
    model: null           # null = download to the cache folder on first use; or a folder holding both model files
    voice: null           # null = the default voice for the scenario language
  stt:
    provider: local_light
    model: null           # null = faster-whisper "small"
    device: auto          # auto = cpu
    compute_type: auto    # auto = int8
  timeout_s: 30
```

**Languages** (measured on 2026-10-26 unless marked):

| language | TTS voice (default) | result |
|---|---|---|
| en-US | `af_heart` | works well: VOICE-011 0.000 (good) |
| es-MX | `ef_dora` | works well: VOICE-011 0.048 (good; only "sí" heard as "si") |
| hi-IN | `hf_alpha` | Whisper `small` STT poor: VOICE-011 0.88. Keep Kokoro TTS, use Deepgram STT ([below](#litellm)) |
| fr, it, pt, zh | `ff_siwis`, `if_sara`, `pf_dora`, `zf_xiaobei` | voices configured, not measured |
| ja | none | only with an explicit `j*` voice and kana-only text |

- **Hindi details.** Whisper `small` writes numbers as misspelled Hindi words and misspells common words.
  The hi-IN run printed "Harness round-trip WER is high; audio metrics in this run are unreliable."
  On a Kokoro Hindi clip, Deepgram's WER was 0.22 (it wrote "order number" in Latin script).
- **Other languages.** A scenario language without a voice stops the run before it starts: "local_light TTS has no voice for {language}. Kokoro speaks en, es, fr, hi, it, pt and zh; set voice_config.tts.voice or use another TTS provider (litellm, openai_compatible) for this language."
- **`tts.voice`** can name any other Kokoro voice id.
- **Mixed languages.** Hindi text with English words is spoken without stray language tags. Impulse removes espeak's `(en)` / `(hi)` markers before Kokoro speaks.
- **Whisper STT** covers many more languages; they were not measured here.
- **TTS and STT are separate choices.** You can use `local_light` for one side only, for example Kokoro speech with Deepgram STT.

**Hardware and time,** measured on the maintainer's PC on 2026-10-26. These are measurements on one PC, not a guarantee:

| what | measured |
|---|---|
| PC | 8 GB RAM, Windows 11, Python 3.12 |
| RAM | about 1.3 to 1.4 GB for TTS and STT together in one process |
| GPU | not needed and not used (`stt.device: auto` means `cpu`) |
| model load | about 30 s, once per run |
| Impulse's own TTS per turn | 2 to 8 s, by sentence length |
| Impulse's own STT per pass | about 8 to 12 s (once for the caller audio, once more for the agent audio) |

A real voice run used `local_light` for TTS and STT, 3 user turns per language. The agent was a scratch agent that also spoke with Kokoro:

| language | VOICE-011 (harness round trip) | wall time, 3 user turns |
|---|---|---|
| en-US | 0.000 (good) | 126 s |
| es-MX | 0.048 (good) | 124 s |
| hi-IN | 0.88 (poor) | 172 s |

That is about 40 to 60 s per user turn. It includes the simulator call, the scratch agent's own Kokoro speech, and the one-time model load.

**Downloads and files:**

- **First use downloads the models.** Kokoro's `kokoro-v1.0.onnx` (325.5 MB) and `voices-v1.0.bin` (28.2 MB) come from the kokoro-onnx GitHub release. They are checksum-verified. Impulse first prints "Downloading the light voice model (353.7 MB) to {folder} (first use only)."
- faster-whisper downloads its `small` model itself from Hugging Face (its `model.bin` is about 484 MB).
- **Where the files go.** `IMPULSE_CACHE_DIR` when you set it. Otherwise `%LOCALAPPDATA%\impulse-evals\kokoro` on Windows, or `~/.cache/impulse-evals/kokoro` (under `$XDG_CACHE_HOME` when set).
- **Your own copy.** Set `voice_config.tts.model` to a folder holding both files. Impulse never downloads into it.
- **`impulse doctor`** checks the extra and whether the model files are present. It loads and downloads nothing.

**Python 3.14.** The `light` extra skips kokoro-onnx there, because kokoro-onnx 0.6.1 declares Python below 3.14. `impulse doctor` then shows this row (verified on Python 3.14.7):

```text
FAIL  voice TTS         voice_config.tts.provider: local_light needs the 'light' extra: pip install impulse-evals[light] (kokoro-onnx 0.6 supports Python 3.11 to 3.13; on 3.14 install it with: pip install kokoro-onnx==0.6.1 --ignore-requires-python)
```

Run the command in the message. Kokoro on Python 3.14 was not measured.

**Licences** of what the `light` extra installs:

- kokoro-onnx (MIT), and the Kokoro-82M model and voices (Apache-2.0);
- phonemizer and the espeak-ng it bundles through espeakng-loader (GPL-3.0-or-later);
- onnxruntime (MIT) and faster-whisper (MIT).

Piper was evaluated and not used. Its voices carry their own licences, with no Hindi voice that is clearly safe for commercial use.

#### litellm

The `litellm` provider reaches a hosted speech service through LiteLLM, the library Impulse already uses for its judge. No extra, shim or proxy is needed.

- `model` is a LiteLLM model string such as `deepgram/...`, `groq/...` or `openai/...`.
- [LiteLLM's provider list](https://docs.litellm.ai/docs/providers) is the authority on what works.
- Verified on 2026-10-26: Deepgram STT (`deepgram/nova-3`) transcribed Kokoro clips in en-US (WER 0.00), es-MX (0.00) and hi-IN (0.22), with word timings and confidence.

A complete config: free Kokoro TTS, Deepgram STT, one scenario each in English, Spanish and Hindi. Save it as `examples/voice-deepgram.yaml` in your clone, next to `sample_agent.py` and `system_prompt.txt`:

```yaml
# impulse:validate
agent:
  adapter: generic
  entrypoint: "sample_agent:respond"
  system_prompt: "system_prompt.txt"

judge:                                 # placeholders: resolved from IMPULSE_MODEL_* or .impulse/config.toml
  primary_model: "YOUR_PRIMARY_MODEL"
  fast_model: "YOUR_FAST_MODEL"

simulator:
  model: "YOUR_SIMULATOR_MODEL"

voice_config:
  tts:
    provider: local_light              # Kokoro text-to-speech, free, no key
  stt:
    provider: litellm                  # speech-to-text through LiteLLM
    model: "deepgram/nova-3"           # the key: DEEPGRAM_API_KEY
  timeout_s: 30

scenarios:
  - name: "order_status_us"
    language: en-US
    max_turns: 3
    persona: "online shopper asking when a late order will arrive"
    assertions: []
    metrics: [UNIV-001, VOICE-001]
  - name: "cita_clinica_mx"
    language: es-MX
    max_turns: 3
    persona: "paciente que quiere una cita con el médico general el jueves"
    assertions: []
    metrics: [UNIV-001, VOICE-001]
  - name: "order_status_hindi"
    language: hi-IN
    max_turns: 3
    persona: "customer who asks in Hindi about order number 4521"
    assertions: []
    metrics: [UNIV-001, VOICE-001]
```

```bash
export DEEPGRAM_API_KEY=...          # PowerShell: $env:DEEPGRAM_API_KEY = "..."; or [keys] in .impulse/config.toml
impulse doctor examples/voice-deepgram.yaml
impulse run examples/voice-deepgram.yaml
```

This config loads with `load_config`, and `impulse doctor` was run on it without a key (verified). A full run with it was not made. Without the key, the doctor's STT row says:

```text
FAIL  voice STT         voice_config.stt.model: deepgram/nova-3 needs the key DEEPGRAM_API_KEY: set the environment variable or add it to [keys] in .impulse/config.toml
```

How it works:

- **Keys follow LiteLLM's conventions.** Without `api_key_env`, the key is the variable LiteLLM reads for the model's provider. For example `DEEPGRAM_API_KEY` for `deepgram/...`, or `GROQ_API_KEY` for `groq/...`. Impulse reads it from the environment, then from `[keys]` in `.impulse/config.toml`.
- **`api_key_env`** names another variable that holds the key. **`base_url`** (optional) is passed to LiteLLM as `api_base`.
- **A missing key stops the run before any paid call**, with the message above. A model that needs no key (a local or credential-file provider) gets none; LiteLLM then reads its own environment.
- **`model` is required.** Without it: "voice_config.stt.model: required when provider is litellm".
- **STT** sends the WAV with the primary language subtag (`es-MX` becomes `es`) and asks for word timings.
- **The Deepgram language workaround.** LiteLLM does not forward `language` to Deepgram, which would then read every turn as English. So for Deepgram, Impulse also passes `detect_language` with the language subtag.
- **TTS** passes `tts.voice` as the voice. It asks for WAV only from `openai`, `azure` and LiteLLM's OpenAI-compatible providers. Other providers (ElevenLabs, for example) answer in their own format. Impulse converts that to WAV with the `voice` extra and ffmpeg.
- **Errors** are one line: "litellm STT failed with model {model}: ..." or "litellm TTS failed with model {model}: ...". Key values are never shown.
- **Retries and timeout.** 429 and 5xx answers are retried twice; other errors are not. Each request times out after `voice_config.timeout_s`.
- **`impulse doctor`** makes a real call per `litellm` side. TTS synthesizes a short phrase. STT transcribes that audio, or a probe tone when TTS is local. Its rows name `litellm`.

Not verified: Groq, OpenAI and ElevenLabs voice through LiteLLM. Known limits (not verified live):

- Groq transcription returns no word timings, so VOICE-015 skips.
- Some OpenAI transcription models return no word timings either. When a model rejects the word-timing request (HTTP 400), Impulse repeats it once without it, and VOICE-015 skips.
- `openai`, `azure` and `elevenlabs` TTS need `voice_config.tts.voice`.

#### openai_compatible

```yaml
# impulse:validate
voice_config:
  tts:
    provider: openai_compatible
    base_url: "YOUR_TTS_SERVER_URL"     # the API root, e.g. ending in /v1
    model: "YOUR_TTS_MODEL"
    api_key_env: null                   # or the NAME of the variable holding the key
  stt:
    provider: openai_compatible
    base_url: "YOUR_STT_SERVER_URL"
    model: "YOUR_STT_MODEL"
    api_key_env: null
```

- **TTS:** `POST {base_url}/audio/speech` with JSON `model` (when set), `input`, `voice` (when set), and `response_format: "wav"`. The response must be WAV bytes.
- **STT:** `POST {base_url}/audio/transcriptions` as multipart: the audio as `file`, plus `model` (when set), `response_format=verbose_json`, word timestamps, and `language`. Word timings and a confidence are used when the server returns them.
- **Key:** `api_key_env` names the variable whose value is sent as `Authorization: Bearer <value>`. Impulse reads the environment first, then `[keys]` in `.impulse/config.toml`.
- **Timeout and retries:** each request times out after `voice_config.timeout_s` (default 30 s). 429 and 5xx answers are retried twice.
- **A 200 answer that is not audio** is a scenario error that shows the server's answer. For example: "openai_compatible TTS: POST http://localhost:8000/v1/audio/speech returned application/json, not audio: {...}". It is not retried. Impulse treats as "not audio" a content type of `application/json`, `application/problem+json` or `text/*`. It also does so for a body starting with `{`, `[` or `<` under `application/octet-stream` or no content type. The same holds for `impulse_hosted`.
- **A `base_url` with credentials** (`http://user:password@host/v1`, for a proxy with Basic auth) is sent as given. Error texts show the URL without the user name and password. The password (or, with no password, the user name) is redacted from the server's error answer. The same holds for `impulse_hosted`.

#### impulse_hosted

- `base_url` must be set in `voice_config.tts` and `voice_config.stt`. There is no default URL; a missing one is a config error.
- Every request sends `Authorization: Bearer <value>` with the value of `IMPULSE_VOICE_API_KEY`, or of the variable named by `api_key_env`. Impulse reads the environment first, then `[keys]` in `.impulse/config.toml`. A missing key is a config error.
- TTS: `POST {base_url}/v1/tts`; STT: `POST {base_url}/v1/stt`.
- Supported languages: `en-IN`, `en-US`, `hi-IN`. Any other scenario language is a config error before anything runs.
- Default voices: `af_sarah` (en-IN), `hi_aryan` (hi-IN), `af_bella` (en-US); `voice` overrides them.
- Requests time out after `voice_config.timeout_s`; 429 and 5xx answers are retried twice.

### Language codes

- Every scenario language is checked against both providers before a voice run starts.
- Engines that expect a bare language, such as Whisper, get the primary subtag only (`hi-IN` becomes `hi`).
- A TTS model usually speaks one language. Use `models_by_language` to pick one per scenario language.

**A TTS that cannot speak the scenario language.** Impulse cannot tell whether an `openai_compatible` or `local_hf` TTS speaks the language. It sends the text and the language code. Many engines then mispronounce or skip the words without any error. (`impulse_hosted` rejects an unsupported language before the run starts.)

The signs:

- a poor VOICE-011 (the harness round-trip WER);
- the run warning "Harness round-trip WER is high; audio metrics in this run are unreliable.";
- a poor VOICE-009.

In a real-world campaign, `en-IN` personas made the simulated caller speak Hinglish into an English-only TTS. VOICE-011 averaged 0.35; an English-only instruction in the persona brought it to 0.06. What to do:

- check the provider's voice list for the scenario's language, and pick a voice with `models_by_language` or `voice`;
- write the persona in a language the TTS speaks. For example, add "speaks English only" to an `en-IN` persona when the TTS has no Hindi voice. Use `code_switching: true` only with a TTS that speaks both languages;
- look at VOICE-011 before trusting the other audio metrics of a run.

### What gets measured, honestly

- **Latency is the agent's own.** Time to first audio (TTFA) and tail latency never include Impulse's own TTS or STT time. Each adapter measures it. The generic adapter times your function call. The LiveKit and Pipecat adapters start the clock when the caller's speech ends, pacing silence excluded. They stop it at the agent's first audio louder than 1% of full scale.
- **Did the agent hear the caller?** VOICE-009 and VOICE-013 are WER metrics. They compare what the caller said with what your agent says it heard, its `heard_text`. An agent that does not report `heard_text` gets the skip "agent does not expose its transcription; return heard_text from the adapter". Impulse never scores the agent's hearing with its own STT.
- **Harness health.** VOICE-011 is the WER of Impulse's TTS audio transcribed by Impulse's own STT. It checks the harness, not the agent. It is informational and never affects pass or fail. When it is poor, the run warns "Harness round-trip WER is high; audio metrics in this run are unreliable."
- **Graceful degradation.** A provider with no word timings makes VOICE-015 skip ("STT provider returned no word timings"). No confidence values means no STT warning for that cohort. Neither fails a scenario.
- **When Impulse's own STT fails on a turn,** the turn is kept with the adapter's text. The run warns "{scenario}: Impulse STT failed on turn {n}: {error}". VOICE-011 (caller audio) or VOICE-015 (agent audio) skips with "Impulse STT failed: {error}". A TTS failure is still a scenario error.
- **Silent agent turns.** An agent turn with no text reaches the simulated caller as "(no speech)". Examples: a tool call only, or audio that Impulse's STT hears as nothing. The transcript keeps the empty text.
- **An agent that returns no audio** (for example a generic handler that returns text only) gets VOICE-001, VOICE-003 and VOICE-005 as labelled proxies. VOICE-009, VOICE-013 and VOICE-015 skip with "agent returned no audio".
- **Text mode** runs the latency metrics as labelled proxies (`is_proxy=True`) on the measured latency. It skips the other audio metrics with "not available in text mode".

The metric definitions and thresholds: [Metrics](metrics.md#voice-metrics).

### Audio files and the report

- With `save_audio: true` (the default), each turn's audio is saved as `.impulse/runs/<run_id>/audio/<scenario>_t<N>_user.wav`, plus `..._t<N>_agent.wav` when the agent returned audio. N is the turn number, both speakers counted. With noise simulation, `..._t<N>_user.wav` is the noisy audio the agent heard, and `..._t<N>_user_clean.wav` the audio before the noise.
- With `save_recording: true` (the default), a stereo recording of the whole call is saved too. It is `<scenario>_recording.wav`, caller on the left channel, agent on the right.
- **Known limitation:** with `save_recording: true`, the runner keeps every audio clip of a scenario in memory until the scenario ends. That holds for each scenario running at once (`--concurrency`). So very long voice calls need a lot of RAM. With `save_recording: false` it keeps none.

The HTML report lets you listen to the call where it went wrong ([Reading the report](reading-the-report.md)):

- **Full call recording.** When the scenario has a stereo recording, its section starts with a player.
  Its title is "Full call recording (left channel: caller, right channel: agent)".
- **Every turn with audio** has a player after its text, for the caller's turns as well as the agent's.
- **"Hear turn N" links.** Some rows of the Assertions and Metrics tables have a "Hear turn N" link.
  These are failed assertions and poor metrics that point at a turn with audio. The link jumps to that turn's player in the transcript. Passed, skipped, warning and informational rows have no link.
- **Call timeline.** Above the transcript of each voice scenario, a small static picture shows the call. It has a "caller" row and an "agent" row, with one bar per turn. A dead-air gap marks an agent turn slower than `metrics.voice.dead_air_ms`. Agent turns slower than `metrics.voice.ttfa_fail_ms` are marked as slow replies. Hover a bar to see the turn number and its times.
- The timeline uses the thresholds the run used (older runs use the defaults, 1500 and 3000 ms). A turn whose times are missing or unusable (for example in a hand-edited `run.json`) is left out; the report is still written. The timeline contains no script and no link.

**Embedded or linked audio:**

- By default (`--embed-audio`), the WAV files are embedded in the report as data URIs. The report is then one self-contained file you can share.
- Each turn's audio and the recording are embedded once; the "Hear turn N" links add nothing.
- The audio is embedded as WAV, so a long call makes a large report. Compressed audio is planned for v1.1 ([Changelog](changelog.md)).
- Above 50 MB of embedded audio (the base64 text in the page, a third more than the files), Impulse prints "Embedded audio is {n} MB; the HTML report will be large. Use --no-embed-audio to link the files instead."
- With `--no-embed-audio`, the report links the files by relative path. The players then work only while the report sits next to its `audio/` folder.
- `impulse report RUN_ID --no-embed-audio` (or `--embed-audio`) rewrites an existing report.

#### WAV files from streaming writers

Some WAV writers stream their output: ffmpeg writing to a pipe, or several TTS servers in stream mode. They put a placeholder (0 or 0xFFFFFFFF) in the WAV size fields. Impulse measures every clip by the audio actually present. It rewrites such a header to the real sizes before the audio is sent, transcribed, saved, or mixed into the recording. So turn times and the recording keep the real length. A 32- or 64-bit floating-point WAV (as torchaudio writes it) is converted to 16-bit PCM with the same rate and channels. A well-formed 16-bit PCM WAV is used unchanged.

### Noise simulation (experimental)

Real callers phone from streets, cafés and cars. Noise simulation mixes background noise into the simulated caller's audio, so you can see how your agent copes. It is **experimental**: covered by offline tests, not yet checked against a live LiveKit or Pipecat agent ([Verified setups](verified-setups.md)).

```yaml
# impulse:validate
simulator:
  model: "YOUR_SIMULATOR_MODEL"
voice_config:
  tts: {provider: local_light}
  stt: {provider: local_light}
  noise: {profile: pink, snr_db: 10}   # every scenario: pink noise at 10 dB
scenarios:
  - name: "booking_noisy"
    language: en-IN
    persona: "patient who wants a slot on Tuesday"
```

All keys and their errors: [Configuration](configuration.md#noise-experimental). A ready example with four levels: [examples/config.noise.yaml](../examples/config.noise.yaml).

**How mixing works:**

- Impulse mixes the noise in after its TTS and before the adapter sends the audio, so the time it takes never counts in your agent's latency.
- `white`, `pink` (softer highs, like a room hum) and `brown` (deep rumble, like traffic) noise is generated by Impulse. With `profile: file`, your own WAV recording is used. It is mixed down to mono, resampled to the caller audio's rate, and looped when it is shorter than the turn.
- The level is set by the signal-to-noise ratio (SNR) in decibels. It is measured on the caller's speech, from its first to its last sound louder than 1% of full scale, so silence around the words does not change the level.
- The noise starts with the clip and stops 20 ms after the caller's last word. The LiveKit and Pipecat adapters time your agent from the end of the caller's speech, so continuous noise would move that point. The cost: noise that goes on after the caller stops talking is not simulated.
- A mix that would clip is turned down as a whole; the SNR stays the same.
- The noise is random, but seeded with the scenario's `seed` (or its name) and the turn number. Re-running a scenario gives the same noisy audio for the same caller audio.

**Choosing the SNR:**

| `snr_db` | sounds like |
|---|---|
| 20 | mild: a quiet office, a fan |
| 10 | a busy street or a café |
| 5 | harsh: a loud street, a car with the window down |
| 0 or below | the noise is as loud as the caller, or louder |

**Your own recordings:** set `profile: file` and `file:` to a WAV file next to your config (any sample rate; 16000 Hz or more is best). Use only recordings you made yourself or that have a clear licence, and no recordings of identifiable people's voices. `impulse doctor` checks each file's sample rate and length.

**Which agents hear the noise:**

- LiveKit and Pipecat: the noisy audio is what Impulse publishes. Not verified live yet.
- A custom adapter: `send()` receives the noisy WAV bytes.
- HTTP: set `agent.http.send_audio: true`, and each turn request carries the noisy audio as `user_audio` ([Adapters](adapters.md#the-exact-json)). Without it, the agent gets text only and the run warns.
- Generic: a handler with a `user_audio` parameter gets the noisy WAV bytes. Without it, the handler never hears the noise and the run warns.

**Which metrics to watch:**

- **VOICE-009** (agent-hearing word error rate): what your agent says it heard, against what the caller said. This is the main signal. It needs `heard_text` from your agent.
- **VOICE-027** (clarifying questions): a good agent asks the caller to repeat what it could not hear, instead of guessing.
- **Task completion** (UNIV-001) and **tool calls** (UNIV-023, UNIV-025): did the booking still happen, with the right details?
- **VOICE-011** stays a harness check: it always transcribes the clean audio, so noise never makes it poor.
- Latency stays your agent's own.

**Recommended workflow: clean against noisy.** Run the same scenarios twice, without and with noise, and let `impulse compare` show what broke:

```bash
impulse run config.yaml
```

Then add `voice_config.noise` (or use a second config file) and run again:

```bash
impulse compare RUN_ID_CLEAN RUN_ID_NOISY
```

The noisy run's report also has a "Pass rate by noise level" table when its scenarios use two or more levels ([Reading the report](reading-the-report.md#noise-simulation)).

### Voice with LiveKit and Pipecat

- **LiveKit:** the caller's audio is published into the room as a microphone track, and the agent's audio track is collected per turn. See [Adapters](adapters.md#livekit).
- **Pipecat:** in eval mode, Pipecat's client transport paces the caller's audio, and the bot's audio arrives resampled to 16000 Hz. In websocket mode, Impulse paces it (`chunk_ms`, `realtime_pacing`, `trailing_silence_ms`). Set `pipecat.sample_rate` to the bot's input rate in both modes. See [Adapters](adapters.md#pipecat).

## Next steps

- [Run Impulse for free](run-for-free.md): a voice run with local speech models.
- [Reading the report](reading-the-report.md): the players, "Hear turn N" links and call timeline.
- [Metrics](metrics.md): every voice metric and its thresholds.
- [Configuration](configuration.md): the full `voice_config` block.
- [Troubleshooting](troubleshooting.md): voice error messages.
- [Known limitations](known-limitations.md): what voice mode cannot do yet.
- [Contributing](contributing.md#help-us-test-noise-simulation): help test noise simulation on your own agent.

---

<!-- docs-nav -->
[← Previous: Adapters](adapters.md) | [Docs home](index.md) | [Next: Metrics →](metrics.md)
