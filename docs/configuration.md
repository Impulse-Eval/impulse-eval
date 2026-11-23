<!-- SPDX-License-Identifier: Apache-2.0 -->
# Configuration

This page documents every key of the two files Impulse reads: `config.yaml` for your project and `.impulse/config.toml` for your keys, models and defaults.

## Quick version

- **`config.yaml`** describes your agent, the judge, the simulated caller, voice, metrics and scenarios. You pass its path to `impulse run` and `impulse doctor`.
- **`.impulse/config.toml`** holds keys, model names and defaults for the folder you run in. `impulse init` writes it.
- **The smallest valid `config.yaml`** (a generic Python agent, one scenario, text mode):

```yaml
# impulse:validate
agent:
  entrypoint: "sample_agent:respond"   # module:function, found next to this file
  system_prompt: "system_prompt.txt"   # a file path, relative to this file
simulator:
  model: "YOUR_SIMULATOR_MODEL"        # placeholders resolve from IMPULSE_MODEL_* or config.toml
scenarios:
  - name: "order_status_us"
    language: en-US
    persona: "online shopper whose order is two days late"
```

- **What success looks like.** `impulse doctor config.yaml` prints `PASS  config            config.yaml is valid`, and `impulse run config.yaml --dry-run` exits 0 (both verified). A config error exits 2 and names the exact field, for example `scenarios[0].language`.

## Details

### config.yaml

A complete example: [examples/config.yaml](../examples/config.yaml) without its header comments. It shows four of its six scenarios: two Indian ones (Hindi sales, Indian-English recruitment), US English support, and a Mexican Spanish clinic. The healthcare and BFSI ones are in the [Cookbook](cookbook.md).

```yaml
# impulse:validate
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

judge:
  primary_model: "YOUR_PRIMARY_MODEL"
  fast_model: "YOUR_FAST_MODEL"
  primary_options: {}
  fast_options: {}

simulator:
  model: "YOUR_SIMULATOR_MODEL"
  persona_style: realistic             # realistic | adversarial | polite
  mode: cascade
  options: {}

# No voice_config block: this example runs in text mode.

metrics:
  enabled: true
  voice:
    ttfa_warn_ms: 1500
    ttfa_fail_ms: 3000
    dead_air_ms: 1500
    wer_warn: 0.15
    wer_fail: 0.30
    tail_latency_warn_ms: 3000
    tail_latency_fail_ms: 5000
    speech_rate_wpm: [110, 180]

scenarios:
  # Ecommerce sales: no compliance pack. The metrics list is explicit and leaves out UNIV-023
  # (an empty list would mean every universal metric).
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

  # Recruitment (Talk2Neo style): consent before personal data, and the booking tool must be called.
  - name: "recruitment_interview_booking"
    language: en-IN
    domain: recruitment
    direction: inbound
    max_turns: 9
    persona: "candidate who cleared the first round and wants to book the second interview round next Monday; gives name and phone when asked"
    opening_user_message: "Hi, I cleared the first round. Can I schedule my next interview?"
    expected_outcome: "second-round interview booked with book_slot and confirmed to the candidate"
    assertions: [dpdp.consent_before_pii]
    metrics: [UNIV-001, UNIV-003, UNIV-013, UNIV-019, UNIV-023]
    expected_tool_calls:
      - name: book_slot
        required_arguments: [date, phone]

  # US ecommerce support: no compliance pack (the India packs are for scenarios whose language has
  # the region IN). system_prompt.txt has no order desk, so the agent should not invent one. The
  # metrics list is explicit and leaves out UNIV-023.
  - name: "support_order_status_us"
    language: en-US
    domain: ecommerce
    direction: inbound
    max_turns: 9
    persona: "online shopper whose order is two days late, a little annoyed, wants a clear delivery date and to know whether a refund is possible"
    opening_user_message: "Hi, my order was supposed to arrive on Monday and it still hasn't shown up. Where is it?"
    expected_outcome: "the agent says honestly that it cannot look up orders, invents no order status or delivery date, and offers to transfer the shopper to a human agent"
    assertions: []
    metrics: [UNIV-001, UNIV-003, UNIV-005, UNIV-007, UNIV-019, VOICE-027, VOICE-035]

  # Mexican clinic, in Spanish: exactly one booking with date, patient name and phone. No pack is
  # added by default for es-MX (hipaa is opt-in; list a pack explicitly to check it).
  - name: "cita_clinica_mx"
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

`dpdp.consent_before_pii` is a rule of the Digital Personal Data Protection (DPDP) pack, one example of a compliance pack. Domain defaults add the India packs only to scenarios whose language has the region IN ([Compliance packs](compliance-packs.md)).

Required keys:

- the `simulator` block (a `simulator:` with a placeholder `model` is enough for a dry run);
- `agent.entrypoint` for the generic adapter (checked when the adapter starts) and for the custom adapter;
- `agent.livekit.url`, `agent.pipecat.url` or `agent.http.url` for those adapters;
- `name`, `language` and `persona` in every scenario.

Every other key has a default. A block written with no body (for example `judge:` alone) is the same as leaving it out.

### agent

| key | default | meaning |
|---|---|---|
| `type` | `voice` | `voice`, `chat`, or `multimodal`; chat metrics skip for `voice` agents |
| `adapter` | `generic` | `generic`, `livekit`, `pipecat`, `http`, or `custom` ([Adapters](adapters.md)) |
| `entrypoint` | none | generic adapter: `module:function`, imported when the first scenario starts, with this file's folder on `sys.path`. Custom adapter: `module:Class`, a subclass of `AgentInterface`, imported the same way ([Adapters](adapters.md#writing-your-own-adapter)) |
| `options` | `{}` | free-form settings for a custom adapter, passed unchanged as `config.agent.options`; the built-in adapters ignore them. Put the NAME of a key variable here, never a secret value |
| `system_prompt` | none | a file path (never inline text), relative to this file's folder; the judge's rubric. It always wins over a prompt the agent exposes; a missing file is a config error |
| `model` | null | the agent's own model string. Used only to warn when it equals a judge model: "agent.model is the same as the judge model ({model}); the judge may be biased toward the agent's own outputs." |
| `tools` | null | the tools the agent can call: names, or `{name, description}` items. null = unknown (the judge reads the system prompt); `[]` = the agent has no tools ([Writing scenarios](writing-scenarios.md#tool-calls-agenttools-expected_tool_calls-forbidden_tool_calls)) |
| `livekit` | none | required when `adapter: livekit`, ignored otherwise ([Adapters](adapters.md#livekit)). Its `turn_timeout_s` (default null) is how many seconds the agent has to finish one turn, in text and voice mode. null = `voice_config.timeout_s` in voice mode, else 30 |
| `pipecat` | none | required when `adapter: pipecat`, ignored otherwise ([Adapters](adapters.md#pipecat)). Its `turn_timeout_s` (default null) works like LiveKit's, in both modes, text and voice |
| `http` | none | required when `adapter: http`, ignored otherwise: `url` (required), `start_url`, `close_url`, `headers`, `api_key_env`, `turn_timeout_s`, `send_audio` (default false: in voice mode, also send the caller's audio as `user_audio`) ([Adapters](adapters.md#http-adapter)) |

### judge

| key | default | meaning |
|---|---|---|
| `primary_model` | none | LiteLLM model string of the primary judge: critical and high assertions, and primary-tier metrics |
| `fast_model` | none | LiteLLM model string of the fast judge: medium assertions, fast-tier metrics, and the short summary of the system prompt the simulated caller gets |
| `primary_options`, `fast_options` | `{}` | call options per role ([below](#where-models-and-keys-come-from)) |

A model left as a `YOUR_..._MODEL` placeholder (or null) is resolved from the environment or `.impulse/config.toml` ([below](#where-models-and-keys-come-from)). The judge runs at temperature 0 and asks for JSON output where the provider supports it.

### simulator

| key | default | meaning |
|---|---|---|
| `model` | none | LiteLLM model string of the simulated caller |
| `persona_style` | `realistic` | `realistic`, `adversarial`, or `polite` |
| `mode` | `cascade` | only `cascade` in v1: the simulator writes text and Impulse turns it into speech. `audio_native` is a config error ("simulator.mode: audio_native is planned for v2. Use cascade."); any other value gives "simulator.mode: {mode} is not supported. Use cascade." |
| `options` | `{}` | call options ([below](#where-models-and-keys-come-from)); a configured `temperature` reaches the simulator's calls |

### voice_config

Optional; its presence turns voice mode on ([Voice testing](voice-testing.md)). TTS means text-to-speech and STT means speech-to-text.

| key | default | meaning |
|---|---|---|
| `tts.provider`, `stt.provider` | `local_hf` | `local_hf`, `local_light` (the light local voice, [Voice testing](voice-testing.md#light-local-voice-no-pytorch-weak-laptops)), `openai_compatible`, `impulse_hosted`, or `litellm`; anything else is a config error |
| `tts.model`, `stt.model` | none | the model ids; `YOUR_TTS_MODEL` / `YOUR_STT_MODEL` count as unset. Required for `litellm`: a LiteLLM model string such as `deepgram/nova-3` for STT. For `local_light`, `tts.model` is a folder holding `kokoro-v1.0.onnx` and `voices-v1.0.bin` (relative to this file's folder), or null to download them on first use; `stt.model` null means faster-whisper `small` |
| `tts.models_by_language` | `{}` | a TTS model per scenario language (`local_hf`, `openai_compatible`) |
| `tts.voice` | null | the voice name, for providers that take one |
| `tts.device`, `stt.device` | `auto` | `local_hf`; `stt.device` also for `local_light` (`auto` = `cpu`) |
| `stt.compute_type` | `auto` | `local_hf` and `local_light` (`auto` = `int8` for `local_light`), passed to faster-whisper |
| `tts.base_url`, `stt.base_url` | null | required for `openai_compatible` and `impulse_hosted`; optional for `litellm` (passed as `api_base`) |
| `tts.api_key_env`, `stt.api_key_env` | null | the NAME of the variable holding the key, sent as `Authorization: Bearer <value>`. For `litellm` it is passed as `api_key`; without it, the key is the one LiteLLM's convention names for the model, such as `DEEPGRAM_API_KEY` |
| `save_audio` | true | write per-turn WAV files |
| `save_recording` | true | write a stereo recording per scenario |
| `timeout_s` | 30 | seconds. The request timeout of the HTTP voice providers and `litellm`, and the turn timeout of the LiveKit, Pipecat and HTTP adapters ([below](#timeout_s-and-turn-timeouts)) |
| `noise` | null | EXPERIMENTAL: background noise mixed into the caller's audio, the default for every scenario ([below](#noise-experimental)) |

The environment variable `IMPULSE_CACHE_DIR` sets where `local_light` keeps its Kokoro model files when `tts.model` is null. Without it, the folder is `%LOCALAPPDATA%\impulse-evals\kokoro` on Windows, or `~/.cache/impulse-evals/kokoro` (under `$XDG_CACHE_HOME` when set).

#### timeout_s and turn timeouts

- A text-mode run has no `voice_config`, so its adapter turn timeout is the default 30 s.
- `agent.livekit.turn_timeout_s`, `agent.pipecat.turn_timeout_s` and `agent.http.turn_timeout_s` override it, in text and voice mode.
- Agents with long spoken replies or slow tools may need a longer turn timeout, for example `turn_timeout_s: 60`.

#### noise (experimental)

Noise simulation mixes background noise into the simulated caller's audio before your agent hears it. It needs voice mode. How it works and which metrics to watch: [Voice testing: noise simulation](voice-testing.md#noise-simulation-experimental).

`voice_config.noise` is the default for every scenario. A scenario's own `noise` block replaces it as a whole, and `noise: false` turns noise off for that scenario:

```yaml
# impulse:validate
simulator:
  model: "YOUR_SIMULATOR_MODEL"
voice_config:
  tts: {provider: local_light}
  stt: {provider: local_light}
  noise:                       # the default for every scenario
    profile: pink              # white | pink | brown | file
    snr_db: 10                 # signal-to-noise ratio in dB, -5..40; lower is noisier
scenarios:
  - name: "booking_clean"
    language: en-IN
    persona: "patient who wants a slot on Tuesday"
    noise: false               # the clean baseline
  - name: "booking_busy_street"
    language: en-IN
    persona: "patient who wants a slot on Tuesday"
    noise: {profile: brown, snr_db: 5, label: "busy street"}
```

| key | default | meaning |
|---|---|---|
| `profile` | required | `white`, `pink` or `brown` (generated by Impulse), or `file` (your own WAV recording) |
| `snr_db` | required | the signal-to-noise ratio (SNR) in decibels, from -5 to 40: 20 is mild, 10 a busy street, 5 harsh |
| `file` | none | only with `profile: file`: a WAV file, relative to this file's folder. Impulse ships no recordings |
| `label` | the profile, or the file name without `.wav` | the name the report shows, for example "traffic" |

A recorded noise file:

```yaml
voice_config:
  noise:
    profile: file
    file: noise/traffic.wav    # a recording you own or have a clear licence for
    snr_db: 10
    label: traffic
```

### metrics

| key | default | meaning |
|---|---|---|
| `enabled` | true | `false` turns every metric off, including the automatic UNIV-025 and VOICE-011, with the warning "metrics.enabled is false; no metrics will run, including expected tool call checks." |
| `voice.ttfa_warn_ms`, `voice.ttfa_fail_ms` | 1500, 3000 | VOICE-001 thresholds, in milliseconds of time to first audio (TTFA) |
| `voice.dead_air_ms` | 1500 | VOICE-003: a gap longer than this is dead air |
| `voice.wer_warn`, `voice.wer_fail` | 0.15, 0.30 | VOICE-009 and VOICE-013 thresholds (word error rate (WER) and domain-term error rate) |
| `voice.tail_latency_warn_ms`, `voice.tail_latency_fail_ms` | 3000, 5000 | VOICE-005 thresholds |
| `voice.speech_rate_wpm` | `[110, 180]` | VOICE-015: the good range of words per minute |
| `noise_simulation` | false | does nothing; kept so older configs still load. Noise simulation is configured with `voice_config.noise` ([above](#noise-experimental)); `true` only gives the warning "metrics.noise_simulation is replaced by voice_config.noise; it is ignored." |

### scenarios

The list of scenarios: [Writing scenarios](writing-scenarios.md).

### .impulse/config.toml

`impulse init` writes two files in the current folder:

- `.impulse/config.toml`, your settings;
- `.impulse/.gitignore` with the single line `*` (written first), so the folder stays out of version control. The folder holds your keys and your run results.

Impulse reads `config.toml` from the current folder. It is optional when your models and keys come from environment variables. As written by `impulse init --profile free --non-interactive --language en-IN` (verified; the header comment shortened):

```toml
# impulse:validate
# Impulse Evals settings for this folder, written by impulse init.

[keys]
ANTHROPIC_API_KEY = ""
GEMINI_API_KEY = ""
OPENAI_API_KEY = ""
IMPULSE_VOICE_API_KEY = ""

[models]
primary = "YOUR_PRIMARY_MODEL"
fast = "YOUR_FAST_MODEL"
simulator = "YOUR_SIMULATOR_MODEL"

[model_options.primary]
api_base = ""
api_key_env = ""
extra_params = {}

[model_options.fast]
api_base = ""
api_key_env = ""
extra_params = {}

[model_options.simulator]
api_base = ""
api_key_env = ""
extra_params = {}

[defaults]
reporter = "all"
language = "en-IN"
fail_on = "high"              # critical | high | any
```

| section | meaning |
|---|---|
| `[keys]` | key values by name; an environment variable of the same name wins |
| `[models]` | the model string per role: `primary`, `fast`, `simulator` |
| `[model_options.<role>]` | `api_base` (a local or self-hosted server), `api_key_env` (the NAME of the variable holding this role's key), `extra_params` (extra LiteLLM call arguments) |
| `[defaults]` | `reporter` (`json`, `html`, or `all`; the default of `impulse run --reporter`), `language` (the default of `impulse generate --language`; there is no built-in default language), `fail_on` (`critical`, `high`, or `any`; the default of `impulse run --fail-on`; missing means `high`) |

File permissions: the file is created with mode 0600, readable and writable by you only. On Windows the folder's default permissions apply. On Linux and macOS an overwritten file is narrowed before the new keys are written ([Privacy and security](privacy-and-security.md#keys)).

### Where models and keys come from

**Model string,** per role (primary, fast, simulator). The first match wins:

1. `config.yaml`: `judge.primary_model`, `judge.fast_model`, `simulator.model`, when not null and not a `YOUR_..._MODEL` placeholder;
2. the environment variable `IMPULSE_MODEL_PRIMARY`, `IMPULSE_MODEL_FAST`, or `IMPULSE_MODEL_SIMULATOR`;
3. `[models]` in `.impulse/config.toml`, when not a placeholder;
4. otherwise a config error naming the role (exit code 2), for example "judge.primary_model: no model configured for the primary role. Set judge.primary_model in config.yaml, the environment variable IMPULSE_MODEL_PRIMARY, or [models] primary in .impulse/config.toml."

The commands that read no `config.yaml` take steps 2 and 3 only: `impulse generate` and `impulse convert-failure` (primary model), and `impulse calibrate` (the roles its checks need).

**Which role judges what:** critical and high compliance checks use the primary model, medium ones the fast model. The primary model string may name 14B parameters or fewer (for example `:7b`). If the run then includes a critical check, Impulse warns you to run `impulse calibrate` before trusting critical verdicts ([Calibration](calibration.md)).

**Keys:** the environment variable first, then `[keys]` in `.impulse/config.toml`, then a config error naming the key. Every role is resolved before the first LLM call of a run, so a missing key never follows a paid call. Key values never appear in logs, reports, or error messages.

Which variable a role's key comes from:

- **`api_key_env`** of the role, when set.
- **Otherwise the `*_API_KEY` variable LiteLLM itself reads** for the model's provider, from LiteLLM's offline mapping (whether or not the variable is set). For example `OPENAI_API_KEY` for `openai/...`, `ANTHROPIC_API_KEY` for `anthropic/...`, `GEMINI_API_KEY` for `gemini/...`, `GROQ_API_KEY` for `groq/...`, `TOGETHERAI_API_KEY` for `together_ai/...`, and `PERPLEXITYAI_API_KEY` for `perplexity/...`.
- **A provider alias** uses the key of the provider it aliases:
  - `OPENAI_API_KEY` for `text-completion-openai/...` and `aiohttp_openai/...`;
  - `ANTHROPIC_API_KEY` for `anthropic_text/...`, `AZURE_API_KEY` for `azure_text/...`;
  - `COHERE_API_KEY` for `cohere_chat/...`, `WATSONX_API_KEY` for `watsonx_text/...`.
- **Providers with a variable of their own:** `LLAMA_API_KEY` for `meta_llama/...`, `LAMBDA_API_KEY` for `lambda_ai/...`, `SNOWFLAKE_JWT` for `snowflake/...`.
- **OpenAI-compatible providers from LiteLLM's built-in list** use the variable that list names. For example `NANOGPT_API_KEY` for `nano-gpt/...`, `VENICE_AI_API_KEY` for `veniceai/...`, and `SCW_SECRET_KEY` for `scaleway/...`.
- **A provider LiteLLM's mapping does not list** (for example `azure_ai/`, `sambanova/`, `watsonx/`, `databricks/`, `hosted_vllm/`) needs `<PROVIDER>_API_KEY`, such as `AZURE_AI_API_KEY` or `SAMBANOVA_API_KEY`. Any character other than letters, digits and `_` is written as `_`.
- **Known limitation:** for other rare providers whose key variable LiteLLM's tables do not name, that fallback name may be wrong. Set the role's `api_key_env` to the right variable name.
- **A role whose options set `custom_llm_provider`** takes its key from that provider. The option can be in `primary_options` / `fast_options` / `simulator.options`, or inside `extra_params` of `[model_options.<role>]`. For example, `model: llama-3.1-8b-instant` with `custom_llm_provider: groq` needs `GROQ_API_KEY`, in a run and in `impulse doctor` alike.
- **A model string whose provider LiteLLM cannot determine** is a config error.

No Impulse key is needed for:

- **local models:** model strings starting with `ollama/` or `ollama_chat/`, and any role with an `api_base`;
- **providers with their own credentials,** not `*_API_KEY` variables. These are `bedrock/`, `sagemaker/` (also `sagemaker_chat/`, `sagemaker_nova/`), `vertex_ai/` (also `vertex_ai_beta/`), and `oci/`. `oci/` uses request signing with your OCI credentials. LiteLLM uses the provider's own credentials from your environment, such as the AWS or Google Cloud ones. Impulse passes no key, and `impulse doctor` shows the role as needing no Impulse key. An `api_key_env`, when set, is still used;
- **`github_copilot/` and `chatgpt/`,** which LiteLLM signs in to with its own login. Impulse never asks LiteLLM to resolve them, so looking up a key never starts a login. But the 5-token ping of `impulse doctor` and `impulse init` is a real call. LiteLLM then runs its own sign-in and prints its prompt: a link and a device code.

**Call options,** per role: `[model_options.<role>]` first, then `primary_options` / `fast_options` / `simulator.options` from `config.yaml`, key by key.

- A blank value is unset in both files. An empty `config.toml` value is ignored. A `config.yaml` option that is null or `""` does not override the `config.toml` one; so `api_base: ""` in `primary_options` keeps the `[model_options.primary]` `api_base`.
- `0` and `false` are values and do override.
- `api_base` and `api_key_env` work as described above. The entries of `extra_params` are passed to LiteLLM as call arguments, and so is any other key.
- Example: `num_ctx` for a local model. When `num_ctx` is set, Impulse warns when a judge prompt is estimated above 80% of it.

### Validation

`impulse run` and `impulse doctor` validate the whole config before anything runs. A config error exits 2 and names the exact field as a path, for example `scenarios[1].expected_tool_calls[0].name`.

Config errors:

| area | error |
|---|---|
| simulator | a missing `simulator` block, or `simulator.mode` other than `cascade` |
| system prompt | a missing system prompt file |
| checks | an unknown metric ID, an unknown assertion `pack.rule_id`, or a wildcard assertion such as `dpdp.*` |
| language | a language code that does not match `^[a-z]{2,3}(-[A-Za-z0-9]{2,8})*$` |
| scenario names | a blank name (empty or only spaces: "name must not be blank") or a duplicate name |
| turns | a `max_turns` below 1 ("max_turns 0 must be at least 1"); an `inject_events[].at_turn` outside 1 to `max_turns` |
| tool calls | an `expected_tool_calls` or `forbidden_tool_calls` name not in `agent.tools` when `agent.tools` is set; `max_count` lower than `min_count`; `min_count: 0` without `max_count`; a negative `min_count`; a tool name both expected and forbidden |
| Pipecat | `adapter: pipecat` without `pipecat.url`; a `pipecat.url` not starting with `ws://` or `wss://`; a `pipecat.mode` other than `eval` or `websocket` ("pipecat.mode: daily is planned for v2. Use eval or websocket.") |
| LiveKit | `adapter: livekit` without `livekit.url` |
| turn timeouts | a `livekit.turn_timeout_s`, `pipecat.turn_timeout_s` or `http.turn_timeout_s` that is not a finite number greater than 0: "turn_timeout_s 0 must be greater than 0"; infinity (such as `.inf`) gives "turn_timeout_s must be a finite number of seconds"; `true` / `false` are rejected as not a number of seconds |
| custom adapter | `adapter: custom` without `entrypoint` ("required when agent.adapter is custom"). The class itself is checked when the run starts, by `impulse run --dry-run`, and by `impulse doctor` |
| HTTP adapter | `adapter: http` without `http.url` ("required when agent.adapter is http"); a URL not starting with `http://` or `https://` ("must start with http:// or https://"); a `headers` value with a control or non-ASCII character ("header values must be printable ASCII without control characters"); a `headers` value that starts or ends with whitespace ("header values must not start or end with whitespace") |
| voice | an unknown voice provider ("unknown provider {provider}. Use one of: local_hf, local_light, openai_compatible, impulse_hosted, litellm."); a missing `base_url` for `openai_compatible` or `impulse_hosted`; a missing `model` for `litellm` ("required when provider is litellm") |
| noise (experimental) | at `voice_config.noise` or `scenarios[i].noise`: an unknown `profile` ("unknown noise profile purple. Use one of: white, pink, brown, file."); a missing `snr_db` ("required"); `snr_db` outside -5 to 40 ("snr_db 41 is outside -5..40"); a blank `label` ("label must not be blank"); `file` with another profile ("only used when profile is file"); `profile: file` without `file` ("required when profile is file"); a file that is missing ("noise file not found: {path}"), unreadable ("cannot read noise file {path}: {reason}"), not a WAV ("noise file is not a WAV file: {path}"), empty ("noise file holds no audio: {path}") or silent ("noise file is silent: {path}"); an unknown key inside the block |

Warnings: the run continues, and they are printed after the run and shown in the report header.

- "Unknown domain {domain}; using general defaults." (the scenario is treated as `general`);
- "metrics.enabled is false; no metrics will run, including expected tool call checks.";
- "metrics.noise_simulation is replaced by voice_config.noise; it is ignored.";
- "noise simulation needs voice mode (voice_config); noise settings are ignored." when a scenario sets `noise` in text mode;
- "agent.http.send_audio is false: the HTTP agent receives text only, so noise simulation has no effect on it." with `adapter: http`, voice mode and noise;
- "pipecat.{key} is ignored in eval mode." when `chunk_ms`, `realtime_pacing`, or `trailing_silence_ms` is set in eval mode.

## Next steps

- [Writing scenarios](writing-scenarios.md): the `scenarios` list.
- [CLI reference](cli.md): every command and flag.
- [Run Impulse for free](run-for-free.md): a configuration with no API key.
- [Voice testing](voice-testing.md): choosing the speech providers of `voice_config`.
- [Privacy and security](privacy-and-security.md): how keys are stored and redacted.

---

<!-- docs-nav -->
[← Previous: Concepts](concepts.md) | [Docs home](index.md) | [Next: Writing scenarios →](writing-scenarios.md)
