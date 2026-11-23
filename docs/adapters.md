<!-- SPDX-License-Identifier: Apache-2.0 -->
# Adapters

An adapter connects Impulse to the agent you test, and this page shows how to choose, configure, and write one.

## Quick version

- An adapter carries each simulated caller turn to your agent and brings back the agent's reply. Your agent code does not change.
- `agent.adapter` in `config.yaml` picks the adapter:

| adapter | use it when your agent is | you set | status |
|---|---|---|---|
| `generic` (the default) | a Python function | `entrypoint: "module:function"` | verified with dry runs |
| `livekit` | a LiveKit Agents worker | `livekit.url` and the names of the key and secret variables | verified in the 2026-10-24 and 2026-10-25 campaigns, text and voice |
| `pipecat` | a Pipecat bot | `pipecat.url`; start the bot with `-t eval` | eval mode verified in the same campaigns; websocket mode not verified |
| `http` | a web service in any language | `http.url` | verified against `examples/http/server.py` on localhost |
| `custom` | anything else (Vapi, Retell, an in-house service) | `entrypoint: "module:Class"` | verified with the example adapter in a dry run |

- The smallest setup is a Python function and the generic adapter:

```yaml
agent:
  adapter: generic
  entrypoint: "my_agent:respond"   # my_agent.py next to config.yaml, with: async def respond(user_text, history)
```

- Check the setup with `impulse doctor config.yaml`. Success is the last line `All checks passed.` and exit code 0.
- Then run `impulse run config.yaml`. Success is a report under `.impulse/runs/<run_id>/`.

Words used on this page:

- **TTS** is text-to-speech, and **STT** is speech-to-text.
- **TTFA** is time to first audio: from the end of the caller's speech to the agent's first audible sound.
- **RTVI** is Pipecat's client/server message protocol.
- **Cascade** (`simulator.mode: cascade`): the simulated caller writes text, and Impulse turns it into speech in voice mode.
- A **tool call** is a function your agent calls, such as `book_slot`. UNIV-023 and UNIV-025 check them.

## Details

### What every adapter returns

Every adapter gives the runner the same data for each agent turn:

- the reply text, and its audio when the adapter has it;
- the latency;
- `heard_text`: what the agent heard the caller say;
- whether the caller interrupted the agent;
- the tool calls the agent made in that turn.

#### Why `heard_text` and `tool_calls` matter

These two optional fields make the strongest checks possible:

- **`heard_text`**: what your agent's own speech recognition heard the caller say. In voice mode, the agent-hearing word error rate (WER) metrics VOICE-009 and VOICE-013 compare it with what the simulated caller said. Without it they skip with "agent does not expose its transcription; return heard_text from the adapter".
- **`tool_calls`**: every tool call your agent made in the turn. UNIV-023 (tool call correctness, judged against your system prompt) and UNIV-025 (exact `expected_tool_calls` and `forbidden_tool_calls` per scenario, checked by code) read them.

How each adapter gets them:

- **generic:** return a dict with `text`, and optionally `heard_text`, `tool_calls` and `audio` (the WAV bytes of the reply, for the audio metrics in voice mode) instead of a string; see the [Generic adapter](#generic-adapter). [examples/sample_agent.py](../examples/sample_agent.py) is a complete agent that returns a dummy `book_slot` call whenever the caller says "book".
- **LiveKit and Pipecat:** captured without code changes. Pipecat from the bot's RTVI user transcriptions and function call messages; LiveKit from the agent's transcription streams and its `lk.agent.session` tool events. Both were verified with live agents in the 2026-10-24 and 2026-10-25 test campaigns, with `heard_text` on every user turn of the voice runs.
- **http:** the JSON answer carries `heard_text` and `tool_calls` the same way ([HTTP adapter](#http-adapter)).
- **custom:** subclass `AgentInterface` and return `AgentReply(text=..., latency_ms=..., heard_text=..., tool_calls=[ToolCall(...)])` from `send()` ([Writing your own adapter](#writing-your-own-adapter)).

### The plugin API

Adapters, metrics, and judges are built against the plugin API files:

- `impulse_evals/plugin_types.py`: the types `AgentTool`, `ToolCall`, `MetricContext`, `MetricResult`, `ScoreResult`;
- `impulse_evals/adapters/base.py`: `AgentInterface`, `AgentReply`, `AgentClosed`;
- `impulse_evals/assertions/base.py`, `impulse_evals/metrics/base.py`, and `impulse_evals/judge/base.py`.

They import no other Impulse module, so an adapter you write against them does not depend on Impulse internals. Like all of Impulse, they are under the Apache License 2.0 ([LICENSING.md](../LICENSING.md)). The import rule is in [Architecture](architecture.md#the-plugin-api-import-rule).

### Generic adapter

| | |
|---|---|
| config | `adapter: generic` and `entrypoint: "module:function"` |
| your function | `(user_text, history) -> str or dict`, sync or async; optionally `(user_text, history, user_audio=None)` to receive the caller's audio |
| extra to install | none |
| system prompt | the `agent.system_prompt` file, else the module's `SYSTEM_PROMPT`, else its `get_system_prompt()` |
| status | used by [examples/sample_agent.py](../examples/sample_agent.py); dry runs verified |

The generic adapter calls a Python function inside the Impulse process, once per agent turn. From [examples/config.yaml](../examples/config.yaml):

```yaml
agent:
  type: voice                          # voice | chat | multimodal
  adapter: generic
  entrypoint: "sample_agent:respond"   # examples/sample_agent.py; this folder is added to sys.path
  system_prompt: "system_prompt.txt"   # relative to this file; wins over sample_agent.SYSTEM_PROMPT
```

- **`entrypoint`** is `module:function`. The module is imported when the first scenario starts, never when the config is loaded. The config file's folder is on `sys.path`. An import failure is a config error at `agent.entrypoint` (exit code 2).
- **`user_text`** is what the simulated caller said. It is `""` when the agent opens an outbound call.
- **`history`** holds the earlier turns of this conversation only, as `[{"role": "user" | "assistant", "content": "..."}]`. Each scenario starts with an empty history.
- **A dict reply** may carry these keys:
  - `text`;
  - `audio`: WAV bytes of the agent's reply;
  - `heard_text`: what the agent's own speech recognition heard;
  - `tool_calls`: a list of dicts with the keys `name`, `arguments`, `result`, `success`, `error`. A missing `arguments` becomes `{}`.
- **A malformed tool call** ends the scenario with an error:
  - a call without a string `name` (an OpenAI-style `{"function": {...}}` call too): "agent function {entrypoint} returned a tool call without a string 'name'";
  - an `arguments` that is not a dict: "agent function {entrypoint} returned a tool call whose 'arguments' is not a dict". A JSON string is not parsed; pass `json.loads(...)`.
- **Latency** is measured by Impulse around the call.
- **Voice mode:** the function receives the caller's text. When it returns `audio`, the audio metrics use it. When not, the metrics that need the agent's audio skip with "agent returned no audio".
- **The caller's audio (optional):** a function that declares a parameter named exactly `user_audio` also gets the caller's audio as WAV bytes in voice mode (`None` in text mode and for the outbound opener). With [noise simulation](voice-testing.md#noise-simulation-experimental) (experimental) it is the noisy audio. A function without that parameter, `**kwargs` included, is called exactly as before; with noise set, the run then warns that the function never hears the noise.

  ```python
  def respond(user_text, history, user_audio=None):
      heard = my_speech_to_text(user_audio) if user_audio else user_text   # your own STT
      return {"text": my_agent_reply(heard), "heard_text": heard}
  ```

A handler that reports everything Impulse can check. It runs as copied: put it next to a config with `entrypoint: "my_agent:respond"`, then replace the body with a call to your agent.

```python
# my_agent.py  (config.yaml: agent.entrypoint: "my_agent:respond")
SYSTEM_PROMPT = "You are the booking assistant of a demo clinic. Call book_slot to book; confirm only if it succeeded."


async def respond(user_text, history):
    """One agent turn. Replace the body with a call to your own agent."""
    if not user_text:   # "" when the agent opens an outbound call
        return "Hello, this is the demo clinic. How can I help you today?"
    reply = {"text": f"You said: {user_text}", "heard_text": user_text}
    if "book" in user_text.lower():
        reply["text"] += " I have booked a slot for you."
        reply["tool_calls"] = [
            {
                "name": "book_slot",
                "arguments": {"date": "2026-01-01", "phone": "0000000000"},
                "result": {"booking_id": "B-1"},
                "success": True,   # None = not captured
                "error": None,
            }
        ]
    return reply
```

`SYSTEM_PROMPT` is used only when `config.yaml` sets no `agent.system_prompt`. Checked with `impulse run config.yaml --dry-run` (exit code 0; the tool calls are in `run.json`).

### LiveKit

| | |
|---|---|
| config | `adapter: livekit` and a `livekit:` block with `url` |
| how it connects | joins a LiveKit room as an ordinary client participant, like a person would |
| extra to install | `pip install -e ".[livekit]"` from a clone, or `pip install "impulse-evals[livekit] @ git+https://github.com/Impulse-Eval/impulse-eval"` without one |
| keys | the LiveKit API key and secret, from the variables named by `api_key_env` and `api_secret_env` |
| system prompt | the `agent.system_prompt` file, else asked from the agent |
| turn timeout | `livekit.turn_timeout_s`, else `voice_config.timeout_s` in voice mode, else 30 s |
| status | verified on 2026-10-24 and 2026-10-25: local `livekit-server` 1.13.10 `--dev`, a livekit-agents 1.8.8 agent with explicit dispatch, text and voice ([Verified setups](verified-setups.md)) |

The extra installs `livekit`, `livekit-api`, and `livekit-protocol`. It never installs `livekit-agents`: Impulse is a client. Your agent code does not change. The protocol details below also match a source read of `livekit` 1.1.21, `livekit-api` 1.2.2, `livekit-protocol` 1.1.27, and `livekit-agents` 1.8.8.

The Talk2Neo example, [examples/talk2neo/config.yaml](../examples/talk2neo/config.yaml) (its first of three scenarios; the header comments are left out):

```yaml
# impulse:validate
agent:
  type: voice
  adapter: livekit                       # entrypoint is for the generic adapter only, so it is not set
  system_prompt: "system_prompt.txt"     # relative to this file; wins over any prompt the agent exposes
  model: null
  tools: null                            # null = unknown: the judge reads the tools from the system prompt
  livekit:
    url: "wss://YOUR_LIVEKIT_HOST"
    api_key_env: "LIVEKIT_API_KEY"       # NAME of the env var holding the LiveKit API key
    api_secret_env: "LIVEKIT_API_SECRET" # NAME of the env var holding the LiveKit API secret
    room_prefix: "impulse-"              # one new room per scenario: {room_prefix}{run_id}-{scenario_name}
    agent_identity: null                 # null = the first agent participant that joins
    agent_name: null                     # set it when Talk2Neo's worker is registered with an agent_name:
                                         # such agents are only dispatched explicitly, and Impulse's access
                                         # token then dispatches that agent into each scenario room
    end_of_turn_grace_ms: 800
    handshake_timeout_s: 10

judge:
  primary_model: "YOUR_PRIMARY_MODEL"
  fast_model: "YOUR_FAST_MODEL"
  primary_options: {}
  fast_options: {}

simulator:
  model: "YOUR_SIMULATOR_MODEL"
  persona_style: realistic
  mode: cascade
  options: {}

metrics:
  enabled: true

scenarios:
  - name: "talk2neo_screening_cooperative"
    language: en-IN
    domain: recruitment
    direction: inbound
    max_turns: 9
    persona: "candidate for a backend developer role who answers every screening question clearly and shares name and phone when asked"
    expected_outcome: "screening questions completed and the candidate told the next step"
    assertions: [dpdp.consent_before_pii, dpdp.purpose_disclosure, dpdp.withdrawal_path]
    metrics: [UNIV-001, UNIV-003, UNIV-009, UNIV-013, UNIV-019, UNIV-023]
```

The scenario above is an Indian one with the DPDP (Digital Personal Data Protection) pack. Scenarios for other countries use the same `livekit` block, for example `language: en-US` or `language: es-MX`.

The `livekit` block (defaults from `impulse_evals/config.py`):

| key | default | meaning |
|---|---|---|
| `url` | required | the LiveKit server URL; `adapter: livekit` without it is a config error |
| `api_key_env` | `LIVEKIT_API_KEY` | the NAME of the variable holding the LiveKit API key (or the same name under `[keys]` in `.impulse/config.toml`) |
| `api_secret_env` | `LIVEKIT_API_SECRET` | the NAME of the variable holding the LiveKit API secret (same lookup) |
| `room_prefix` | `impulse-` | each scenario's room is `{room_prefix}{run_id}-{scenario_name}` |
| `agent_identity` | null | the agent's participant identity; null = the first agent participant that joins |
| `agent_name` | null | set it when the agent's worker is registered with an `agent_name`: Impulse's access token then dispatches that agent into each scenario room |
| `end_of_turn_grace_ms` | 800 | how long the agent must stay neither `"speaking"` nor `"thinking"` before its turn ends |
| `handshake_timeout_s` | 10 | how long to wait for the agent to join the room |
| `turn_timeout_s` | null | seconds the agent has to finish one turn, in text and voice mode; null = `voice_config.timeout_s` when voice mode is on, else 30. When set it must be a finite number greater than 0 (not `true` / `false`) |

#### How a LiveKit conversation works

- **Rooms.** Each scenario gets a new room named `{room_prefix}{run_id}-{scenario_name}`.
- **Tokens.** Impulse creates its access token locally with `livekit-api`. It reads the API key and secret from the variables named by `api_key_env` and `api_secret_env`. When a variable is not set, it reads the same name under `[keys]` in `.impulse/config.toml`.
- **The `can_manage_agent_session` grant.** Impulse's token carries this grant. Two features use the agent's `lk.agent.session` stream: the system prompt request and tool-call capture. Your LiveKit server or project must allow the grant. Otherwise the request gets no answer and no tool calls are captured. Tool-call capture worked on `livekit-server --dev` in the campaigns; other servers are not verified.
- **Which participant is the agent:** `agent_identity` when set, otherwise the first agent participant that joins.
- **Text turns** go out on the text-stream topic `lk.chat`. The reply is the agent's final `lk.transcription` text streams (attribute `lk.transcription_final` is `"true"`).
- **Voice turns** publish the caller's audio as a microphone track and collect the agent's audio. Latency is the TTFA: from the moment the caller's last speech sample left Impulse to the agent's first audio louder than 1% of full scale.
- **System prompt:** when `agent.system_prompt` is not set, Impulse sends a `GetAgentInfo` request on `lk.agent.session` before the first turn. It waits up to 3 seconds for the agent's instructions. No answer means the judge falls back to general best practice. Not verified: the campaigns set `agent.system_prompt`.
- **`heard_text`:** the agent's final `lk.transcription` streams of the user's speech (those sent with Impulse's own identity).
- **`tool_calls`:** the `function_tools_executed` events on `lk.agent.session`, each call paired with its output:
  - arguments are JSON-decoded, and an output that is a JSON object becomes `result`;
  - `is_error` gives `success=False` with the output as `error`.
  - Such an `error` that is a JSON object is shown with its sensitive keys hidden and cut at 300 characters, as for `result=`.
  - Otherwise a boolean `result["success"]` is used.
  - Otherwise `success` stays `None`. The runner then infers it from the result shape, or leaves it not captured. For example `{"ok": true}` is a success ([Metrics](metrics.md#univ-023-notes)).
- **`interrupted`:** the agent's `lk.agent.state` attribute leaves `"speaking"` while Impulse is still sending the caller's speech.
- **End of turn:** after the agent produced output, the turn ends when `lk.agent.state` is neither `"speaking"` nor `"thinking"` for `end_of_turn_grace_ms`. A running tool keeps the agent `"thinking"`, so "call a tool, then speak again" stays one agent turn.
- **Room closed after the first agent turn:** the conversation ends normally and the turns so far are judged.
- **Room closed before the agent said anything:** the scenario error "the agent closed the conversation before its first turn: the LiveKit room closed: {reason}". Nothing is judged.
- **The URL in the run files** is shown without credentials. That holds for `run.json`, the report, and the connect error ("could not connect to the LiveKit server at ..."). Removed are `user:password@`, `?token=`, and similar query parameters. Impulse still connects with the full URL.
- **`impulse doctor`** checks the extra and the key and secret variables, but not the connection.
- **Missing extra:** "livekit adapter needs the 'livekit' extra: pip install impulse-evals[livekit]".

#### LiveKit turn timeout (`turn_timeout_s`)

- A turn that does not end in time is a scenario error. The text is "LiveKit agent did not finish its turn within {N} s.", for example "... within 30 s.".
- The timeout is `livekit.turn_timeout_s` when you set it (text and voice mode). Otherwise it is `voice_config.timeout_s` in voice mode, otherwise 30 seconds.
- Raise it for an agent that calls two or three slow tools in one turn. In text mode that can need more than 30 seconds. Example: `turn_timeout_s: 60`.
- Raise it for voice agents with long spoken replies too. In voice mode the turn ends only when the agent stops speaking. The 2026-10-25 campaign used `turn_timeout_s: 90` for a text-mode agent that kept its audio output on.
- **Partial turns are kept on a timeout.** What the agent produced before the timeout is recorded as an agent turn, before the scenario error. That is its text so far, its tool calls in that turn, and its audio in voice mode. So `run.json` and the report show which tools ran. As for every scenario error, the scenario is not judged.

#### Dispatch on slow or busy machines ("no servers available")

A LiveKit agent worker can decline a job. Then Impulse waits for an agent that never comes.

- **The symptom.** The scenario ends with "LiveKit agent did not join within {N} s." (for example 20 s with `handshake_timeout_s: 20`). The LiveKit server log says "no servers available", or "failed to send job request: no servers available".
- **Why.** The worker declines new jobs while the machine is busy (its `load_threshold`; LiveKit's production default is 0.7 of the CPU). Right after a job ends, its idle process may still be restarting. Seen in both campaigns on a busy laptop, for example right after a voice scenario.
- **Fixes on the agent side** (check these names against your livekit-agents version):
  - register the worker with an `agent_name` and set the same `livekit.agent_name` in `config.yaml` (explicit dispatch);
  - raise the worker's `load_threshold` for local test runs (the campaigns used `load_threshold=math.inf`);
  - keep more idle processes (`num_idle_processes=2`).
- **`agent_name` is also required** for any agent whose worker is registered with an `agent_name`. Such an agent is only dispatched explicitly, never automatically into new rooms. Without `livekit.agent_name` it never joins, and every scenario ends with "LiveKit agent did not join within 10 s." (`handshake_timeout_s`).

A fragment of the campaign agent's worker file (livekit-agents 1.8.8; check against your livekit-agents version). It is not a complete agent: your `AgentSession` code goes in `entrypoint`.

```python
# fragment of the agent's worker file: check against your livekit-agents version
import math

from livekit.agents import AgentServer, JobContext, cli

server = AgentServer(num_idle_processes=2, load_threshold=math.inf)   # local test runs only


@server.rtc_session(agent_name="supportbot")   # config.yaml: agent.livekit.agent_name: "supportbot"
async def entrypoint(ctx: JobContext) -> None:
    ...   # start your AgentSession in ctx.room here


if __name__ == "__main__":
    cli.run_app(server)
```

`livekit-server --dev` and network changes: in dev mode the local LiveKit server binds to the machine's network address at start-up. After a network change (another Wi-Fi network, a VPN, waking from sleep with a new address), local runs break. Restart `livekit-server --dev` then (seen in a real-world test campaign).

### Pipecat

| | |
|---|---|
| config | `adapter: pipecat` and a `pipecat:` block with `url` |
| how it connects | as a client of your running bot, speaking RTVI; Impulse never imports or runs your pipeline |
| modes | `eval` (the default; start the bot with `-t eval`) or `websocket` (the fallback) |
| extra to install | `pip install -e ".[pipecat]"` from a clone, or `pip install "impulse-evals[pipecat] @ git+https://github.com/Impulse-Eval/impulse-eval"` without one; installs `pipecat-ai>=1.12,<1.13` |
| system prompt | the `agent.system_prompt` file, else the bot's answer to the `get_system_prompt` client message |
| tool calls | need the RTVI function call report level FULL; eval mode requests it itself |
| concurrency | Pipecat bots serve one client at a time, so scenarios run one by one |
| turn timeout | `pipecat.turn_timeout_s`, else `voice_config.timeout_s` in voice mode, else 30 s |
| status | eval mode verified on 2026-10-24 and 2026-10-25 with a pipecat-ai 1.12.0 bot started with `-t eval`, text and voice ([Verified setups](verified-setups.md)); websocket mode not verified |

- Eval mode relies on internal APIs of the tested pipecat-ai release. So the upper version bound is raised only after a newer release has been tested.
- Pipecat's APIs change between releases. Every Pipecat code sample on this page is marked "check against your Pipecat version". The class and parameter names were checked in the installed pipecat-ai 1.12.0 source, and the bot-side samples match the campaign test bot.
- A step-by-step version of this section is [examples/pipecat/README.md](../examples/pipecat/README.md).
- Known Pipecat limitations (greetings before the first turn, late tool results) are collected in [Known limitations](known-limitations.md).

The two connection modes share everything except the wire:

| | `eval` (default) | `websocket` (fallback) |
|---|---|---|
| the bot runs with | Pipecat's eval transport: the `"eval"` entry in the bot's `transport_params`, started with `-t eval` | any Pipecat WebSocket server transport using `ProtobufFrameSerializer` |
| the caller's audio is paced by | Pipecat's own client transport | Impulse: `chunk_ms`, `realtime_pacing`, `trailing_silence_ms` |
| use it when | the bot uses Pipecat's development runner (all Pipecat examples do) | older Pipecat releases without `pipecat.evals`, or bots not built on the development runner |

#### Eval mode (recommended)

1. Add the `"eval"` entry to the bot's `transport_params`, as in Pipecat's evals quickstart. Pipecat's example agents already have it. A fragment of `bot.py`:

   ```python
   # check against your Pipecat version
   from pipecat.evals.transport import EvalTransportParams

   transport_params = {
       # ... your other transports ...
       "eval": lambda: EvalTransportParams(audio_in_enabled=True, audio_out_enabled=True),
   }
   ```

2. Start the bot with the eval transport. By default Pipecat's development runner prints a line ending in `Bot ready! (eval transport on ws://localhost:7860)`. Copy that WebSocket URL into `pipecat.url`:

   ```bash
   # check against your Pipecat version
   python bot.py -t eval
   ```

3. Check the connection: `impulse doctor examples/pipecat/config.yaml`. Success: the `pipecat.evals` and `pipecat handshake` rows say `PASS` (the bot answered with `bot-ready`). An `INFO` row says whether the bot answers `get_system_prompt`.
4. Run: `impulse run examples/pipecat/config.yaml`. Success: a report under `.impulse/runs/<run_id>/`.

How eval mode connects:

- Impulse uses Pipecat's own client classes, `EvalClientTransport` with `EvalClientSerializer`, inside a small pipeline of its own. So Pipecat's client transport sends `client-ready` and paces the caller's audio like a live call.
- In text mode Impulse sets the `skip_tts` flag per connection, so the bot sends no audio.
- In voice mode it sets `capture_bot_audio`. The bot's audio arrives as `eval-bot-audio`, resampled to 16000 Hz.
- It never sets `trigger_disconnect`.

After `bot-ready`, before the first turn of every scenario, Impulse sends two client messages:

1. **`eval-configure`** with `{"function_call_report_level": {"*": "full"}}`. Tool names, arguments, and results then reach Impulse without bot changes. Both `impulse run` (every scenario) and `impulse doctor` send it. The setting stays on the bot afterwards: the bot keeps reporting function calls at the FULL level after Impulse disconnects.
2. **`eval-context`**, the conversation reset, described in [Conversations between scenarios](#conversations-between-scenarios).

Version messages:

- A pipecat-ai outside 1.12.x gives a run warning: "pipecat-ai {version} is not the tested 1.12.x; eval mode relies on its internal APIs."
- When a needed internal is missing, eval mode stops with a config error that points at the websocket fallback.
- A pipecat-ai without `pipecat.evals` gives "Pipecat eval mode needs a pipecat-ai release that includes pipecat.evals. Upgrade pipecat-ai, or set pipecat.mode: websocket."
- With no pipecat-ai installed, `impulse run` and the `impulse doctor` `pipecat.evals` row give "pipecat adapter needs the 'pipecat' extra: pip install impulse-evals[pipecat]".

#### Websocket mode (fallback)

Use it for older Pipecat releases without `pipecat.evals`, or for bots not built on Pipecat's development runner. Not verified against a live bot.

- Run a Pipecat WebSocket server transport with `ProtobufFrameSerializer`, and set `pipecat.mode: websocket`.
- Impulse sends `client-ready` itself. It encodes PCM audio and RTVI messages as Pipecat protobuf frames.
- Impulse paces the caller's audio itself: `chunk_ms` frames at real-time speed (`realtime_pacing`), then `trailing_silence_ms` of silence. The silence lets the bot's voice activity detection end the user turn.
- A bot on Pipecat's development runner started with `-t websocket` listens on the `/ws-client` path. So `pipecat.url` ends in `/ws-client`, for example `ws://localhost:7860/ws-client`.
- Such bots start a new bot for every connection, so each scenario is a fresh conversation.
- They reject a bad token or `Origin` header with close code 4003.
- Websocket mode has no `eval-configure` message, so it relies on the bot-side FULL setting below.

```yaml
# check against your Pipecat version
agent:
  adapter: pipecat
  pipecat:
    mode: websocket
    url: "ws://localhost:7860/ws-client"   # development runner, -t websocket
    sample_rate: 16000          # must match the bot's input audio sample rate
    chunk_ms: 20                # websocket mode only
    realtime_pacing: true       # websocket mode only
    trailing_silence_ms: 600    # websocket mode only
```

(from [examples/pipecat/README.md](../examples/pipecat/README.md))

#### Bot-side checklist

Eval mode (recommended):

1. Add the `"eval"` entry to the bot's `transport_params`, as in Pipecat's evals quickstart (Pipecat's example agents already have it).
2. Start the bot with `-t eval` and copy the WebSocket URL it prints into `pipecat.url`.
3. Recommended: reset the LLM context in the bot's `on_client_connected` handler. Or set `agent.system_prompt` in `config.yaml`, so Impulse resets it with `eval-context`.

Websocket mode: run a Pipecat WebSocket server transport with `ProtobufFrameSerializer`. For a bot on Pipecat's development runner started with `-t websocket`, `pipecat.url` ends in `/ws-client` (`ws://host:port/ws-client`).

Both modes:

1. Keep RTVI enabled. It is the default, and the eval transport needs an RTVI processor and observer in the pipeline.
2. Set the RTVI function call report level to FULL, so tool names, arguments, and results reach Impulse (see [Function call report level FULL](#function-call-report-level-full)). In eval mode Impulse also requests FULL with `eval-configure`.
3. Keep user transcription messages enabled, so `heard_text` is available.
4. Optional: answer the `get_system_prompt` client message with `{"system_prompt": SYSTEM_PROMPT}` (see [System prompt handler](#system-prompt-handler)).
5. Set `pipecat.sample_rate` to the bot's input audio sample rate (`PipelineParams.audio_in_sample_rate`, default 16000). The bot does not resample inbound audio.
6. Per-service latency needs `PipelineParams(enable_metrics=True)` (default off).
7. If the bot accepts origins only from `PIPECAT_ALLOWED_ORIGINS`, put an `Origin` header in `pipecat.headers`.

```python
# check against your Pipecat version
from pipecat.pipeline.worker import PipelineParams

params = PipelineParams(enable_metrics=True)
```

#### Configuration

An excerpt of [examples/pipecat/config.yaml](../examples/pipecat/config.yaml): its first of two scenarios. The commented `voice_config` block is left out; uncomment it in the file for voice mode.

```yaml
# impulse:validate
agent:
  type: voice
  adapter: pipecat                 # entrypoint is for the generic adapter only, so it is not set
  system_prompt: null              # null = ask the bot with the get_system_prompt client message (README.md);
                                   # or set a path to your bot's prompt file, which then always wins and lets
                                   # Impulse reset a -t eval bot's conversation before each scenario (README.md)
  model: null
  tools: null                      # null = unknown: the judge reads the tools from the system prompt
  pipecat:
    mode: eval                     # eval (default) | websocket
    url: "ws://localhost:7860"     # replace with the WebSocket URL your bot prints when started with -t eval
    end_of_turn_grace_ms: 800
    handshake_timeout_s: 10
    headers: {}
    api_key_env: null              # NAME of an env var; its value is sent as "Authorization: Bearer <value>"
    sample_rate: 16000             # both modes: user audio is sent at this rate; must equal the bot's
                                   # PipelineParams.audio_in_sample_rate (the bot does not resample inbound audio)
    # websocket mode only (mode: websocket); setting these in eval mode only prints a warning.
    # A bot on Pipecat's development runner started with -t websocket listens at ws://host:port/ws-client.
    # chunk_ms: 20
    # realtime_pacing: true
    # trailing_silence_ms: 600

judge:
  primary_model: "YOUR_PRIMARY_MODEL"
  fast_model: "YOUR_FAST_MODEL"
  primary_options: {}
  fast_options: {}

simulator:
  model: "YOUR_SIMULATOR_MODEL"
  persona_style: realistic
  mode: cascade
  options: {}

metrics:
  enabled: true

scenarios:
  - name: "pipecat_text_conversation"
    language: en-US
    domain: general
    direction: inbound
    max_turns: 6
    persona: "curious first-time user who asks what the bot can do, then asks one follow-up question"
    opening_user_message: "Hi! What can you help me with?"
    expected_outcome: "the bot explains what it can do and answers the follow-up question"
    assertions: []
    metrics: [UNIV-001, UNIV-009, UNIV-013, UNIV-019, UNIV-023, VOICE-029, VOICE-039]
```

| key | default | meaning |
|---|---|---|
| `mode` | `eval` | `eval` or `websocket`; `daily`, `webrtc`, and `livekit` are planned for v2 and are a config error ("pipecat.mode: daily is planned for v2. Use eval or websocket.") |
| `url` | required | `ws://` or `wss://`; eval mode: the URL the bot prints; websocket mode on the development runner: ends in `/ws-client`. Credentials in it (`user:password@`, or a `token`, `key`, `api_key`, `api-key`, `access_token`, `apikey`, `api_token`, `sig`, `signature`, `secret`, `password` or `auth` query parameter) are used to connect but never stored or shown: `run.json`, the report, error texts, and `impulse doctor` show the URL without them |
| `sample_rate` | 16000 | both modes: the caller's audio is sent at this rate; must equal the bot's input rate |
| `chunk_ms`, `realtime_pacing`, `trailing_silence_ms` | 20, true, 600 | websocket mode only; set in eval mode they only give the warning "pipecat.{key} is ignored in eval mode." |
| `end_of_turn_grace_ms` | 800 | how long the bot must stay quiet before its turn ends |
| `handshake_timeout_s` | 10 | how long to wait for `bot-ready` |
| `headers` | `{}` | extra WebSocket headers (for example `Origin`); every value except `Origin` is redacted from errors and warnings (for `Bearer <token>`, the token only) |
| `api_key_env` | null | the NAME of an environment variable (or a `[keys]` entry in `.impulse/config.toml`); its value is sent as `Authorization: Bearer <value>` |
| `turn_timeout_s` | null | seconds the bot has to finish one turn, both modes, text and voice; null = `voice_config.timeout_s` when voice mode is on, else 30. When set it must be a finite number greater than 0 (not `true` / `false`), as for LiveKit. It is not a websocket-only key |

#### Pipecat turn timeout (`turn_timeout_s`)

- A turn that does not end in time is a scenario error. The text is "Pipecat bot did not finish its turn within {N} s.", for example "... within 30 s.".
- The timeout is `pipecat.turn_timeout_s` when you set it (both modes, text and voice). Otherwise it is `voice_config.timeout_s` in voice mode, otherwise 30 seconds.
- Raise it for bots with slow tools or long spoken replies, for example `turn_timeout_s: 60`.
- Partial turns are kept on a timeout (fixes round 3); see [What Impulse captures](#what-impulse-captures).

#### System prompt handler

- When `agent.system_prompt` is set in `config.yaml`, Impulse uses that file and asks the bot nothing.
- Otherwise it sends the RTVI `client-message` `{"t": "get_system_prompt"}`. It waits up to 3 seconds for a `server-response` with the same `id`, and reads `data.d.system_prompt`.
- No answer means the judge falls back to general best practice.

A bot opts in with a handler like this one, where `task` is your `PipelineWorker`. The campaign test bot used the same handler with pipecat-ai 1.12.0. A fragment of `bot.py`:

```python
# check against your Pipecat version
rtvi = task.rtvi

@rtvi.event_handler("on_client_message")
async def on_client_message(rtvi, message):
    if message.type == "get_system_prompt":
        await rtvi.send_server_response(message, {"system_prompt": SYSTEM_PROMPT})
```

#### Function call report level FULL

Pipecat reports function calls over RTVI only as far as its report level allows. The default `{"*": NONE}` sends call ids without names or arguments, which UNIV-023 cannot check. Set it to FULL in the bot (a fragment of `bot.py`):

```python
# check against your Pipecat version
from pipecat.processors.frameworks.rtvi.observer import RTVIFunctionCallReportLevel, RTVIObserverParams

task = PipelineWorker(
    pipeline,
    rtvi_observer_params=RTVIObserverParams(
        function_call_report_level={"*": RTVIFunctionCallReportLevel.FULL}
    ),
)
```

- In eval mode Impulse also requests FULL itself with `eval-configure` after `bot-ready`, on every connection of `impulse run` and `impulse doctor`. The setting stays on the bot after the run.
- When calls still arrive without names or arguments, the run gets the warning "Pipecat bot reports function calls without names or arguments. Set the RTVI function call report level to FULL so UNIV-023 can check tool arguments."

#### Conversations between scenarios

Impulse opens one connection per scenario. But a bot started with `-t eval` stays up and keeps its LLM context between connections. So a new connection alone does not start a fresh conversation.

- **With `agent.system_prompt` set** in `config.yaml`, Impulse resets the bot before each scenario. It sends an `eval-context` client message whose only message is that system prompt, which replaces the bot's LLM context. The bot then runs the file's prompt, not its own, so keep the file identical to the prompt your bot deploys. The run says so once:

> Pipecat eval bot: Impulse reset its conversation with the agent.system_prompt file before each scenario, so the bot ran that prompt, not its own. Keep the file identical to the bot's prompt.
- **Without it**, Impulse cannot reset the bot, and the run gets this warning once:

> Pipecat eval bots keep their conversation between scenarios. Set agent.system_prompt in config.yaml so Impulse can reset it, or reset the LLM context in the bot's on_client_connected handler.

- A development-runner bot started with `-t websocket` starts a new bot for every connection, so its conversations are always separate.

#### One client at a time

Pipecat's eval transport and its single-client WebSocket server transport serve one client at a time.

- For `adapter: pipecat`, Impulse runs scenarios one by one.
- With `--concurrency` above 1 it prints "Pipecat bots serve one client at a time; running scenarios one by one."
- If the bot still refuses the connection with close code 1013, the scenario error is "Pipecat bot accepts one client at a time. Run with --concurrency 1, or start one bot per connection."
- Stop any other client first, for example a `pipecat eval` run against the same bot.

#### Using Impulse alongside pipecat eval

Pipecat ships its own `pipecat eval` tool. The same `-t eval` bot works with Impulse unchanged. The two tools can run against the same bot, one at a time.

What Impulse adds for Pipecat users:

- judging against the bot's own system prompt;
- the tool-call checks UNIV-023 and UNIV-025;
- compliance packs, such as the India packs;
- per-language cohorts and `impulse compare`;
- one tool across LiveKit and Pipecat.

Remember that the FULL report level Impulse sets with `eval-configure` stays on the bot afterwards.

#### What Impulse captures

- **Reply text in text mode.** Text turns use RTVI `send-text` with `audio_response: false`. The reply is the `bot-output` texts, or the `bot-llm-text` tokens when no `bot-output` arrives. The second case is normal in eval text mode, where the bot skips TTS.
- **Reply text and audio in voice mode.** The reply is the `bot-output` segments that were spoken (`spoken: true` or `spoken_status: "completed"`). The bot's audio is collected into one WAV, from the caller's last speech frame on. Bot audio heard while the caller was still speaking (such as a greeting tail) is not in it. An outbound opener keeps all bot audio since the connection opened.
- **Tool-call turns in text mode** (no doubled replies). Found by source reading and campaign 1, where every doubled reply was a tool-call turn. In the 2026-10-25 campaign (Deepgram TTS), no text-mode reply repeated a sentence. The word-timestamp TTS case below is not verified live.
  - In pipecat-ai 1.12.0 a `-t eval` bot turns its TTS back on at the end of each text turn. Its RTVI processor never sees the eval transport's `skip_tts` flag. So the LLM answer after a tool call is spoken, even in a text run.
  - A TTS service that pushes its text (Deepgram-style, `push_text_frames=True`) then reports every sentence twice, under different segment IDs. First before synthesis (`bot-output` with `will_be_spoken: true`, `spoken: false`, `spoken_status: "new"`). Then after it (`spoken: true`, `spoken_status: "completed"`).
  - In text mode Impulse keeps an announced sentence (`will_be_spoken: true`, not yet spoken or completed) in the reply. It drops it only when its completed copy (same text, after trimming spaces) arrives in the same turn; the completed copy then takes its place.
  - With a word-timestamp TTS, the completed copy is released as the audio plays and can arrive after the text turn ended. The announced sentence then stays in that turn's reply, and its late copy is not repeated in the next turn.
  - Impulse remembers the segment ID of each sentence kept that way. The word-by-word progress messages (`spoken_status: "in-progress"`) and the late completion for it are then not added to the next turn either.
  - A segment is forgotten once its completion arrives. Everything remembered is forgotten when the bot reports `bot-interrupted` (for example when the next `send-text` interrupts it), since those completions will never come.
  - Other `bot-output` messages are kept as before. So the reply is not doubled, and the answer after a tool call stays in its own turn. A sentence the bot really says twice is kept twice.
  - Because the bot synthesizes speech for these turns, tool-call turns are slower than other turns in a text run. Their text-mode latency includes the bot's TTS time. Voice mode is unchanged.
- **End of turn.**
  - Voice: `bot-stopped-speaking` with no function call in progress, and no `bot-started-speaking` within `end_of_turn_grace_ms`.
  - Text: `bot-llm-stopped` with no function call in progress, and no `bot-llm-started` within the grace window.
  - In both modes a function call in progress holds the turn open. The grace window starts again when the last call's `llm-function-call-stopped` arrives. So "call a tool, then speak again" stays one turn, however long the tool runs.
  - No end of turn within the turn timeout is a scenario error; see [Pipecat turn timeout](#pipecat-turn-timeout-turn_timeout_s).
- **Partial turns are kept on a timeout** (fixes round 3).
  - What the bot produced before the timeout is recorded as an agent turn, before the scenario error, as for LiveKit. That is its text so far, its tool calls, and its audio in voice mode.
  - In voice mode the text is the spoken `bot-output` texts. When none was spoken, it is every `bot-output` text of the turn. When there is none, it is the bot's LLM tokens.
  - This keeps a turn where the bot speaks a raw function call and never sends `bot-stopped-speaking` (seen in a real-world test campaign).
  - When nothing arrived, no turn is added. Tested with the campaign's message shapes; not verified against a live bot.
- **The bot closes the connection.**
  - Close code 1000 or 1001 during a turn is the bot hanging up. The conversation ends normally, with the bot's last words kept.
  - Any other code is a scenario error: "the Pipecat bot closed the connection (close code 1006)", with the real code. Examples are 1006 when the bot process died, or 1011 for a server error.
  - A close right after `bot-ready` is a scenario error too, for example while Impulse waits for the system prompt.
  - A bot that hangs up before its first turn is a scenario error too: "the agent closed the conversation before its first turn: ...".
- **Latency.** Text: from sending `send-text` to the first `bot-output` or `bot-llm-text`. Voice: the TTFA, from the last frame of the caller's speech (pacing silence excluded) to the bot's first audio louder than 1% of full scale.
- **A greeting before the first turn.** A bot may greet from `on_client_connected`. It may still be generating or speaking that greeting when Impulse sends the caller's first turn.
  - Just before sending a turn, Impulse reads everything the bot has already sent.
  - Bot output that arrived before the turn was sent is not the reply. Neither is an LLM run (`bot-llm-started`) or bot speech (`bot-started-speaking`) that started before `send-text` was sent, or before the caller's last speech frame.
  - Their text is added to the first reply, before the real answer. But they never set the latency and never end the turn. Latency comes from the first LLM run or bot speech that starts afterwards.
  - One exception for voice: the bot may not let the caller interrupt its greeting. It may then speak the reply right behind it in the same speech, with one `bot-stopped-speaking` for both.
  - Then the end of that speech ends the turn, once a spoken `bot-output` arrived during it after the reply's own LLM text. Spoken means `spoken: true` or `spoken_status: "completed"`. The latency then comes from that text.
  - The reply's LLM text alone is not enough, so a reply whose own speech starts late (slow TTS) is still waited for.
  - In text mode a spoken greeting's `bot-output` never replaces a reply made of `bot-llm-text`.
  - Two limitations remain; see [Known limitations](known-limitations.md). First, a greeting whose LLM run or speech has not started when Impulse sends the turn cannot be told apart from the reply. It sets the latency, so a bot that greets late can make the first turn's latency look short. The first turn's text then starts with the greeting. Impulse does not wait for the bot to go quiet.
  - Second, RTVI does not say which LLM run a `bot-output` belongs to. A greeting the caller cannot interrupt may still have sentences to play when the reply's text arrives; such a spoken greeting sentence also counts. If the reply's own speech then starts more than `end_of_turn_grace_ms` after the greeting stops, the first turn ends with the greeting only. The reply then appears in the next turn. Raise `pipecat.end_of_turn_grace_ms` above your TTS's first-audio delay to avoid it.
- **`heard_text`:** the final RTVI `user-transcription` messages of the turn (partials ignored).
- **`tool_calls`:** from `llm-function-call-in-progress` and `llm-function-call-stopped` (and the deprecated `llm-function-call`).
  - A cancelled call gets `success=False`, `error="cancelled"`.
  - Otherwise a boolean `result["success"]` is used, or a non-empty `result["error"]` gives `success=False`.
  - Else `success` stays `None`, and the runner infers it from the result, for example `{"ok": true}` ([Metrics](metrics.md#univ-023-notes)).
  - A stop that arrives in a later turn never changes a reply already returned. Known limitation: such a late result is recorded nowhere, neither on the earlier turn nor on the later one. The transcript keeps the call with the `success` it had when its turn ended, often `None` ([Known limitations](known-limitations.md)).
- **A tool call that raised is a failure** (fixes round 3).
  - When a function handler raises, pipecat-ai 1.12.0 returns "The function `NAME` failed and returned no result." to the LLM. It also sends the RTVI error "Error executing function call [NAME]: ...".
  - Impulse records that call with `success=False`. Its `error` is that RTVI error text (redacted, on one line, cut to 200 characters). When no such error arrived, it is the "failed and returned no result" text.
  - The stored result stays `{"value": "The function ... failed and returned no result."}`.
  - The AGENT TOOL CALL line the judge reads then shows `success=False error=...`, so UNIV-023 sees the failure.
- **Which call gets the RTVI error.** Pipecat can run calls with the same name in parallel, and the two messages arrive in either order.
  - An RTVI error only ever goes to a call of the same name, in the same turn, that failed or whose result never arrived.
  - A successful call, a call with a dict result, and a cancelled call are never changed.
  - An error that matches no such call is dropped when the turn ends. It stays only as the run warning "Pipecat bot reported an RTVI error: ...".
  - Tested with the campaign's message shapes; not verified against a live bot.
- **`interrupted`:** a `user-started-speaking` message while the bot is speaking.
- **Per-service latency:** RTVI `metrics` messages (STT, LLM, and TTS timings, converted from seconds to milliseconds). They are stored in the transcript metadata as `pipecat_metrics` and shown in the report's latency section. They are informational and never scored.
- **Handshake timeout:** "Pipecat bot did not send bot-ready within 10 s. Check that the bot is running at {url} (eval mode: started with -t eval) and RTVI is enabled."
- **Missing extra:** "pipecat adapter needs the 'pipecat' extra: pip install impulse-evals[pipecat]".

### HTTP adapter

| | |
|---|---|
| config | `adapter: http` and an `http:` block with `url` |
| how it connects | one JSON `POST` per agent turn to your endpoint; optional start and close requests per scenario |
| extra to install | none: it uses `httpx`, a core dependency |
| key | optional: `api_key_env` names a variable, sent as `Authorization: Bearer <value>` |
| system prompt | the `agent.system_prompt` file, else the start answer's `system_prompt` |
| timeout | `http.turn_timeout_s`, else `voice_config.timeout_s` in voice mode, else 30 s, per request |
| status | verified against [examples/http/server.py](../examples/http/server.py) on localhost; a Node.js agent is not verified |

The HTTP adapter tests an agent written in any language (Node.js, Go, Java, ...) through a small web API. Impulse reads the reply from the JSON answer. Step by step: [examples/http/README.md](../examples/http/README.md).

From [examples/http/config.yaml](../examples/http/config.yaml) (the header comments are left out):

```yaml
# impulse:validate
agent:
  type: chat
  adapter: http
  system_prompt: null                  # null = the start answer's system_prompt (server.py sends one)
  model: null
  tools: [book_slot]
  http:
    url: "http://localhost:8080/turn"         # required; http:// or https://
    start_url: "http://localhost:8080/start"  # optional: POSTed once when a scenario starts
    close_url: "http://localhost:8080/close"  # optional: POSTed once when a scenario ends
    headers: {}                               # extra request headers (values are redacted in errors)
    api_key_env: null                         # NAME of an env var; its value is sent as "Authorization: Bearer <value>"
    turn_timeout_s: null                      # seconds per request; null = voice_config.timeout_s, else 30

judge:
  primary_model: "YOUR_PRIMARY_MODEL"
  fast_model: "YOUR_FAST_MODEL"
  primary_options: {}
  fast_options: {}

simulator:
  model: "YOUR_SIMULATOR_MODEL"
  persona_style: realistic
  mode: cascade
  options: {}

metrics:
  enabled: true

scenarios:
  - name: "http_agent_booking"
    language: en-US
    domain: healthcare
    direction: inbound
    max_turns: 4
    persona: "patient who wants an appointment next Tuesday and gives their name and phone number when asked"
    opening_user_message: "Hi, I'd like to book an appointment for next Tuesday."
    expected_outcome: "the agent books the slot with book_slot and confirms it"
    assertions: []
    metrics: [UNIV-001, UNIV-013, UNIV-023]
    expected_tool_calls:
      - name: book_slot
        max_count: 1
```

The `http` block (used only with `adapter: http`):

| key | default | meaning |
|---|---|---|
| `url` | required | the turn endpoint; `http://` or `https://`. `adapter: http` without it is the config error `agent.http.url`: "required when agent.adapter is http" |
| `start_url` | null | optional: POSTed once when a scenario starts |
| `close_url` | null | optional: POSTed once when a scenario ends |
| `headers` | `{}` | extra request headers. Values must be printable ASCII without control characters, else the config error `agent.http.headers.{name}`: "header values must be printable ASCII without control characters"; and they must not start or end with whitespace, else "header values must not start or end with whitespace" (the value is never shown in either case) |
| `api_key_env` | null | the NAME of an environment variable (or a `[keys]` entry in `.impulse/config.toml`); its value, stripped of surrounding spaces, is sent as `Authorization: Bearer <value>` |
| `turn_timeout_s` | null | seconds per request; null = `voice_config.timeout_s` when voice mode is on, else 30. Same rules as the LiveKit and Pipecat `turn_timeout_s` |
| `send_audio` | false | voice mode: also send the caller's audio in each turn request, as `user_audio` ([below](#the-exact-json)). With [noise simulation](voice-testing.md#noise-simulation-experimental) it is the noisy audio. With noise set and `send_audio: false`, the run warns that the agent gets text only |

A URL that does not start with `http://` or `https://` is the config error on that field: "must start with http:// or https://".

#### The exact JSON

Every request is a `POST` with:

- `Content-Type: application/json`;
- the `headers` mapping;
- `Authorization: Bearer <value>` when `api_key_env` is set.

Bodies are UTF-8 JSON. `conversation_id` is `"impulse-"` followed by 12 lowercase hex characters, new for every scenario.

**start** (only when `start_url` is set), when a scenario starts:

```json
{
  "conversation_id": "impulse-3f9a1c0b7d2e",
  "scenario_name": "http_agent_booking",
  "language": "en-US",
  "direction": "inbound"
}
```

Answer: any 2xx. When the body is a JSON object with a string `system_prompt`, Impulse uses it as the agent's system prompt. It does so only when `config.yaml` sets no `agent.system_prompt`. Any other body is ignored.

```json
{"system_prompt": "You are the booking assistant of a demo clinic. ..."}
```

**turn** (`url`), once per agent turn:

```json
{
  "conversation_id": "impulse-3f9a1c0b7d2e",
  "user_text": "Hi, I'd like to book an appointment for next Tuesday.",
  "history": [
    {"role": "user", "content": "Hello"},
    {"role": "assistant", "content": "Hello, this is the demo clinic. How can I help you today?"}
  ],
  "scenario_name": "http_agent_booking"
}
```

- `history` holds the earlier turns of this conversation only; the current `user_text` is not in it.
- `user_text` is `null` for an outbound scenario's first request: the agent speaks first. That request adds no user entry to `history`.
- In voice mode the agent receives text. With `http.send_audio: true` the request also carries the caller's audio, as one more OPTIONAL key:

  ```json
  {
    "conversation_id": "impulse-3f9a1c0b7d2e",
    "user_text": "Hi, I'd like to book an appointment for next Tuesday.",
    "history": [],
    "scenario_name": "http_agent_booking",
    "user_audio": "UklGRiQ+AABXQVZFZm10IBAAAAABAAEAgD4AAAB9AAACABAAZGF0YQ..."
  }
  ```

  `user_audio` is the base64 text (standard alphabet, with padding) of a WAV file: the audio your agent should hear, noisy when [noise simulation](voice-testing.md#noise-simulation-experimental) is on. It is sent only with `send_audio: true`, only in voice mode, and never for the opener. Without it the body has exactly the four keys above, so an agent that ignores `user_audio` keeps working.

Answer: a JSON object. Only these keys are read; other keys are ignored.

```json
{
  "text": "I have booked Tuesday at 10:00 for you.",
  "tool_calls": [
    {
      "name": "book_slot",
      "arguments": {"date": "2026-01-06", "patient_name": "Asha", "phone": "0000000000"},
      "result": {"success": true},
      "success": true,
      "error": null
    }
  ],
  "heard_text": "hi i'd like to book an appointment for next tuesday",
  "audio": null
}
```

- `text`: a string; missing or `null` is an empty reply.
- `tool_calls`: the same item rules as the generic adapter. `name` is a non-empty string and required. `arguments` is an object; missing is `{}`. Then `result`, `success` (`null` lets Impulse infer it from `result`), and `error`.
- `heard_text` (optional): what the agent understood the caller to say, stored on the user turn.
- `audio` (optional): base64 of a WAV file, standard or URL-safe alphabet, padding optional.

**close** (only when `close_url` is set), when a scenario ends:

```json
{"conversation_id": "impulse-3f9a1c0b7d2e", "scenario_name": "http_agent_booking"}
```

The answer body is ignored.

#### How it works

- **One client per scenario.** Impulse opens one HTTP client in `start` and closes it in `close`. A failed start request sends no close request.
- **Latency** is measured by Impulse around the turn request.
- **Timeout.** `turn_timeout_s` is a deadline for the whole request, body included, for start, turn and close. A server that sends its answer slowly cannot hold a turn open.
- **No retries.** A turn can have side effects, such as a booking or a payment.
- **The bearer key.** Impulse reads the variable named by `api_key_env` from the environment, then from `[keys]` in `.impulse/config.toml`. Put the NAME in `config.yaml`, never the value.
- **URL credentials.** With `api_key_env` set, a `user:password@` in the URLs is not sent, so the Bearer header is used. Without it, such userinfo is sent as Basic auth.
- **`impulse doctor`** opens a plain TCP connection to the turn URL's host and port and sends nothing. It never calls your endpoint, because a turn can have side effects.
- **`impulse run --dry-run`** checks the config and sends no request.
- **The URL in the run files:** `run.json` and the report show `http.url` without credentials.
- **Windows and `localhost`.** On Windows, `localhost` may try IPv6 first. That costs about 0.4 s per request when the server listens on IPv4 only, as `server.py` does. Use `127.0.0.1` in the URLs then.

#### Errors

Each one ends the scenario, with one exception: a failing close request. When the conversation itself finished and only the `close_url` request fails, the scenario is still evaluated and the run gets the warning "{scenario}: the adapter's close failed after the conversation ended; the results are kept: {error}". Error texts show the URL without credentials. The answer body is redacted first (known keys, the bearer token, every header value, also in their JSON-escaped and Python-repr forms). Then it is put on one line and cut to 200 characters.

- a non-2xx answer: "HTTP agent returned HTTP {status} for POST {url}: {body}"; an empty body is shown as "(empty body)"
- no answer in time: "HTTP agent did not answer within {seconds} s (POST {url})."
- no connection, or another transport error: "could not reach the HTTP agent at {url}: {error}"
- a turn answer that is not a JSON object: "HTTP agent returned a body that is not a JSON object (POST {url}): {body}"
- a bad field: "HTTP agent returned a non-string 'text'", "HTTP agent returned a non-string 'heard_text'", "HTTP agent returned 'audio' that is not base64"
- a bad tool call, one of:
  - "HTTP agent returned 'tool_calls' that is not a list"
  - "HTTP agent returned a tool call that is not a dict"
  - "HTTP agent returned a tool call without a string 'name'"
  - "HTTP agent returned a tool call whose 'arguments' is not a dict"
  - "HTTP agent returned an invalid tool call: {location}: {message}"
- `api_key_env` names a variable that is not set, or is blank. This is the config error `agent.http.api_key_env`: "{NAME} is not set: set the environment variable or add it to [keys] in .impulse/config.toml"
- its value has a control or non-ASCII character: the config error `agent.http.api_key_env`: "the value of {NAME} must be printable ASCII without control characters"

#### A Node.js (Express) sketch (not verified)

This sketch has not been run against Impulse (not verified). Check it against your Express version.

```js
// not verified
const express = require("express");
const app = express();
app.use(express.json());

app.post("/turn", (req, res) => {
  const { conversation_id, user_text, history, scenario_name } = req.body;
  if (user_text === null) {
    return res.json({ text: "Hello, this is the demo clinic. How can I help you today?" });
  }
  const toolCalls = [];
  if (/\bbook\b/i.test(user_text)) {
    toolCalls.push({
      name: "book_slot",
      arguments: { date: "2026-01-01", patient_name: "Test User", phone: "0000000000" },
      result: { success: true },
      success: true,
    });
  }
  res.json({ text: `You said: ${user_text}`, tool_calls: toolCalls });
});

app.post("/start", (req, res) => res.json({}));
app.post("/close", (req, res) => res.json({}));

app.listen(8080, () => console.log("listening on http://localhost:8080/turn"));
```

### Writing your own adapter

| | |
|---|---|
| base class | `AgentInterface` from `impulse_evals.adapters.base` (Apache-2.0) |
| you implement | `start(scenario)`, `send(user_text, user_audio)`, `close()`, all async |
| optional | `get_system_prompt()`, `get_warnings()`, `get_metadata()` |
| use it from config | `adapter: custom` and `entrypoint: "module:Class"` |
| use it from Python | `impulse_evals.runner.run(config, adapter_factory=...)` |
| status | the example adapter is verified in a dry run; a real Vapi or Retell adapter is not verified |

For any other stack, subclass `AgentInterface` and implement three async methods:

- **`start(scenario)`**: begin a new conversation for the scenario.
- **`send(user_text, user_audio)`**: send one caller turn and return an `AgentReply(text=..., latency_ms=..., audio_bytes=..., heard_text=..., interrupted=..., tool_calls=[ToolCall(...)])`.
  - In text mode `user_audio` is `None`.
  - `send(None, None)` means "wait for the agent's opener" (outbound calls).
- **`close()`**: end the conversation. It must tolerate an already closed connection.

Ending a conversation from `send`:

- **The agent hung up:** raise `AgentClosed(message, reply=...)`. The runner ends the conversation normally and evaluates the turns so far.
- **It hung up before its first turn:** there is no agent turn yet, and no `reply` with text, tool calls, or audio. The runner then records a scenario error instead: "the agent closed the conversation before its first turn: {message}". Nothing is judged.
- **A failure in the middle of a turn** (a timeout, say): raise `AgentTurnError(message, reply=...)` from `impulse_evals.adapters.base`, with the turn so far. When that `reply` has text, a tool call, or audio, the runner records it as an agent turn first. Use `float("nan")` as `latency_ms` when nothing arrived yet. Then the message becomes the scenario error, so the tools the agent called in that turn stay in `run.json`.
- **Any other failure:** raise any other exception.

Optional methods:

- `get_system_prompt()`: used only when `config.yaml` sets no `agent.system_prompt`;
- `get_warnings()`: run warnings, deduplicated;
- `get_metadata()`: merged into the transcript metadata.

`ToolCall` is in `impulse_evals.plugin_types`. Leave `success=None` when your stack does not report it. The runner then infers it from common result shapes such as `{"ok": true}`, `{"error": "slot taken"}`, or `{"status": "failed"}` ([Metrics](metrics.md#univ-023-notes)). It redacts known key values in `result` and `error` before storing them.

There are two ways to use your adapter. Load it from `config.yaml` with `adapter: custom`, so every `impulse` command works as usual. Or pass it from Python.

#### From config.yaml: `adapter: custom`

Use this for an agent hosted on a voice platform (Vapi, Retell, ...) or an in-house service. Write the class with that platform's own client library, then name it in `config.yaml`.

A minimal adapter that runs as copied. Save it as `my_adapter.py` next to `config.yaml`, then replace the commented bodies with calls to your platform:

```python
# my_adapter.py  (config.yaml: agent.adapter: custom, agent.entrypoint: "my_adapter:MyAdapter")
import time

from impulse_evals.adapters.base import AgentInterface, AgentReply
from impulse_evals.plugin_types import ToolCall


class MyAdapter(AgentInterface):
    def __init__(self, config):
        self.options = dict(config.agent.options)   # your settings from config.yaml agent.options

    async def start(self, scenario):
        self.scenario_name = scenario.name   # open the platform's conversation here

    async def send(self, user_text, user_audio):
        started = time.perf_counter()
        text = f"You said: {user_text or ''}"   # send the turn to your agent and wait for its reply here
        calls = []
        if user_text and "book" in user_text.lower():
            calls.append(ToolCall(name="book_slot", arguments={"date": "2026-01-01"}, result={"success": True}, success=True))
        return AgentReply(text=text, latency_ms=(time.perf_counter() - started) * 1000, tool_calls=calls)

    async def close(self):
        pass   # end the conversation here; it must tolerate an already closed connection

    def get_system_prompt(self):   # optional: used only when config.yaml sets no agent.system_prompt
        return "You are the booking assistant of a demo clinic."
```

Checked: with `entrypoint: "my_adapter:MyAdapter"`, `impulse run config.yaml --dry-run` exits 0. A script that created it and called `start`, `send`, and `close` got the reply and the `book_slot` call back.

The fuller example [examples/custom_adapter/echo_adapter.py](../examples/custom_adapter/echo_adapter.py) has the same shape with no network. Its docstring shows where a real client goes. Its config, [examples/custom_adapter/config.yaml](../examples/custom_adapter/config.yaml) (the header comments are left out):

```yaml
# impulse:validate
agent:
  type: chat
  adapter: custom
  entrypoint: "echo_adapter:EchoAdapter"   # module:Class, imported from this folder (or sys.path)
  system_prompt: null                      # null = the adapter's get_system_prompt()
  model: null
  tools: [book_slot]
  options: {}                              # free-form settings for your adapter (an assistant ID, the
                                           # NAME of a key variable); never put a secret value here

judge:
  primary_model: "YOUR_PRIMARY_MODEL"
  fast_model: "YOUR_FAST_MODEL"
  primary_options: {}
  fast_options: {}

simulator:
  model: "YOUR_SIMULATOR_MODEL"
  persona_style: realistic
  mode: cascade
  options: {}

metrics:
  enabled: true

scenarios:
  - name: "custom_adapter_booking"
    language: en-US
    domain: healthcare
    direction: inbound
    max_turns: 4
    persona: "patient who wants an appointment next Tuesday and gives their name and phone number when asked"
    opening_user_message: "Hi, I'd like to book an appointment for next Tuesday."
    expected_outcome: "the agent books the slot with book_slot and confirms it"
    assertions: []
    metrics: [UNIV-001, UNIV-013, UNIV-023]
    expected_tool_calls:
      - name: book_slot
        max_count: 1
```

How it works:

- **`entrypoint`** is `module:Class`; dots after the colon reach a nested class. The module is imported with the config file's folder on `sys.path`, as for the generic adapter. Loading the config never imports it.
- **One instance per scenario,** created as `Class(config)` with the loaded config.
- **`agent.options`** arrive unchanged as `config.agent.options`. Put the NAME of a key variable there, never the key itself.
- **Everything else works as for a built-in adapter:** tool-call checks, `AgentClosed`, `AgentTurnError`, `--fail-on`, `impulse compare`, cohorts, and reports.
- **`--concurrency`** is used as given, so your adapter must respect the platform's own limits.
- **`run.json`** stores the entrypoint string as `agent_entrypoint`.
- **`impulse doctor`** imports the class and checks it without creating it. It does not check the connection.
- **`impulse run --dry-run`** imports and checks the class without creating it. Its warning: "Dry run: scenarios run only with the generic adapter; the custom config was validated without connecting." Verified with the example (exit code 0).

Loading errors are all config errors at `agent.entrypoint` (exit code 2):

- no `entrypoint`: "required when agent.adapter is custom"
- not `module:Class`: "{entrypoint} is not of the form module:Class (e.g. my_adapter:MyAdapter)"
- the import fails: "cannot import module {module} from {dir} or sys.path: {type}: {message}"
- no such attribute: "module {module} has no class {attr}"
- not a subclass of `AgentInterface`: "{module}:{attr} is not a subclass of impulse_evals.adapters.base.AgentInterface"
- the constructor raises: "could not create {module}:{attr}: {type}: {message}" (the message redacted, on one line, cut to 200 characters). A `ConfigError` your constructor raises is shown unchanged.

#### From Python: `adapter_factory`

- Pass the adapter to the async `impulse_evals.runner.run(config, adapter_factory=...)`.
- The factory receives the loaded config and returns a new adapter per scenario.
- `run` returns the `RunResult` without writing the run files. In voice mode it writes only the audio under `audio/`.
- `write_run_files` from `impulse_evals.reporter` writes the usual run folder.

This example ran as shown with `dry_run=True` next to a valid `config.yaml` (verified). Drop that argument for a real run:

```python
import asyncio
import time

from impulse_evals.adapters.base import AgentInterface, AgentReply
from impulse_evals.config import load_config
from impulse_evals.plugin_types import ToolCall
from impulse_evals.reporter import RUNS_DIR, ensure_impulse_dir, write_run_files
from impulse_evals.runner import run


class EchoAdapter(AgentInterface):
    def __init__(self, config):
        self.config = config

    async def start(self, scenario):
        self.history = []

    async def send(self, user_text, user_audio):
        started = time.perf_counter()
        text = f"You said: {user_text or ''}"
        calls = []
        if user_text and "book" in user_text.lower():
            calls.append(ToolCall(name="book_slot", arguments={"date": "2026-01-01"}, success=True))
        return AgentReply(text=text, latency_ms=(time.perf_counter() - started) * 1000, tool_calls=calls)

    async def close(self):
        pass


config = load_config("config.yaml")
result = asyncio.run(run(config, dry_run=True, adapter_factory=lambda cfg: EchoAdapter(cfg)))
ensure_impulse_dir()
write_run_files(result, RUNS_DIR / result.run_id)
```

## Next steps

- [Voice testing](voice-testing.md): what each adapter's audio means for the voice metrics.
- [Writing scenarios](writing-scenarios.md): `agent.tools`, `expected_tool_calls`, and `forbidden_tool_calls`.
- [Known limitations](known-limitations.md): Pipecat greetings, late tool results, and other limits.
- [Troubleshooting](troubleshooting.md): adapter error messages.
- [Architecture](architecture.md): where the adapters sit in a run.

---

<!-- docs-nav -->
[← Previous: Writing scenarios](writing-scenarios.md) | [Docs home](index.md) | [Next: Voice testing →](voice-testing.md)
