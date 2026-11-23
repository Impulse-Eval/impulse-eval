<!-- SPDX-License-Identifier: Apache-2.0 -->
# Testing a Pipecat bot with Impulse

Impulse connects to your running Pipecat bot as a client and speaks RTVI, Pipecat's standard client/server message protocol. Your pipeline does not change, and Impulse never imports or runs it in its own process.

Every code sample on this page is marked "check against your Pipecat version": Pipecat's APIs change between releases. The names below were read from the pipecat-ai 1.12.0 source; nothing on this page has been verified against a live LLM bot yet (not verified).

## Run it

1. Install the extra from a clone of the repository: `pip install -e ".[pipecat]"` (without a clone: `pip install "impulse-evals[pipecat] @ git+https://github.com/Impulse-Eval/impulse-eval"`). It installs `pipecat-ai>=1.12,<1.13`, the tested release, which includes `pipecat.evals`. Success: the install finishes without errors.
2. Prepare your bot with the checklist below.
3. Start your bot with the eval transport and copy the WebSocket URL it prints into `pipecat.url` in `config.yaml`.

   ```bash
   # check against your Pipecat version
   python bot.py -t eval
   ```

   Pipecat's development runner prints a line ending in `Bot ready! (eval transport on ws://localhost:7860)` by default.

4. Check the connection: `impulse doctor examples/pipecat/config.yaml`. Success: the Pipecat handshake row passes (the bot answered with `bot-ready`); an info row says whether the bot answers `get_system_prompt`.
5. Run the scenarios: `impulse run examples/pipecat/config.yaml`. Success: a report under `.impulse/runs/<run_id>/`.

As shipped, `config.yaml` runs in text mode: Impulse sends text with RTVI `send-text` and the bot skips text-to-speech (TTS). Voice mode is switched on per config file by the `voice_config` block; uncomment it to run both scenarios in voice mode, so the bot's audio is captured and scored.

## Bot-side checklist

### Eval mode (recommended)

1. Add the `"eval"` entry to the bot's `transport_params`, as in Pipecat's evals quickstart (Pipecat's example agents already have it).

   ```python
   # check against your Pipecat version
   from pipecat.evals.transport import EvalTransportParams

   transport_params = {
       # ... your other transports ...
       "eval": lambda: EvalTransportParams(audio_in_enabled=True, audio_out_enabled=True),
   }
   ```

2. Start the bot with `-t eval` and copy the WebSocket URL it prints into `pipecat.url`.
3. Recommended: reset the LLM context in your `on_client_connected` handler, or set `agent.system_prompt` in `config.yaml` so Impulse resets it for you (see "Conversations between scenarios" below).

### Websocket mode (fallback)

For older Pipecat releases without `pipecat.evals`, or bots not built on Pipecat's development runner: run a Pipecat WebSocket server transport with `ProtobufFrameSerializer` and set `pipecat.mode: websocket`. A bot on Pipecat's development runner started with `-t websocket` listens on the `/ws-client` path, so `pipecat.url` ends in `/ws-client` (for example `ws://localhost:7860/ws-client`); such bots start a new bot for every connection and reject a bad token or `Origin` with close code 4003.

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

### Both modes

1. Keep RTVI enabled (the default; the eval transport needs an RTVI processor and observer in the pipeline).
2. Set the RTVI function call report level to FULL so tool names, arguments, and results reach Impulse (UNIV-023 checks them). The default is `{"*": NONE}`, which reports only call ids. In eval mode `impulse run` (every scenario) and `impulse doctor` also request FULL themselves with an `eval-configure` message after `bot-ready`, and the setting stays on the bot after the run (also after `impulse doctor`); websocket mode relies on this setting.

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

3. Keep user transcription messages enabled so `heard_text` is available; without it the agent-hearing word error rate (WER) metrics skip.
4. Optional: answer the `get_system_prompt` client message with `{"system_prompt": SYSTEM_PROMPT}`, so the judge can grade the bot against its own prompt when `agent.system_prompt` is not set in `config.yaml`.
5. Set `pipecat.sample_rate` to the bot's input audio sample rate (`PipelineParams.audio_in_sample_rate`, default 16000): Impulse sends the user's audio at that rate in both modes, and the bot does not resample inbound audio.
6. Per-service latency, the report's timings of the bot's speech-to-text (STT), LLM, and TTS services, needs metrics turned on in the bot: `PipelineParams(enable_metrics=True)` (default off).

   ```python
   # check against your Pipecat version
   from pipecat.pipeline.worker import PipelineParams

   params = PipelineParams(enable_metrics=True)
   ```

7. If the bot accepts origins only from `PIPECAT_ALLOWED_ORIGINS`, put an `Origin` header in `pipecat.headers`.

On the wire, Impulse sends an RTVI `client-message` with `data = {"t": "get_system_prompt"}`, waits up to 3 seconds for a `server-response` with the same `id`, and reads `data.d.system_prompt`. A handler sketch, where `task` is your `PipelineWorker`:

```python
# check against your Pipecat version (not verified)
rtvi = task.rtvi

@rtvi.event_handler("on_client_message")
async def on_client_message(rtvi, message):
    if message.type == "get_system_prompt":
        await rtvi.send_server_response(message, {"system_prompt": SYSTEM_PROMPT})
```

Note for users of Pipecat's own `pipecat eval`: the same `-t eval` bot works with Impulse unchanged; the two tools can run against the same bot (one at a time).

## Good to know

- Pipecat bots serve one client at a time, so Impulse runs Pipecat scenarios one by one, whatever `--concurrency` says. A bot that still refuses a second client closes it with code 1013, and Impulse reports that as the scenario error.
- Conversations between scenarios: Impulse opens one connection per scenario, but a bot started with `-t eval` stays up and keeps its LLM context between connections, so a new connection alone does not start a fresh conversation. When `agent.system_prompt` is set in `config.yaml`, Impulse resets the bot before each scenario with an `eval-context` message whose only message is that system prompt. Without it, Impulse cannot reset the bot and the run shows the warning "Pipecat eval bots keep their conversation between scenarios. Set agent.system_prompt in config.yaml so Impulse can reset it, or reset the LLM context in the bot's on_client_connected handler." A development-runner bot started with `-t websocket` starts a new bot for every connection, so its conversations are always separate.
- Impulse records the bot's per-service timings (STT, LLM, TTS) from RTVI `metrics` messages, converted to milliseconds, and shows them in the report's latency section. They are informational and never scored.
- Turn timeout: a bot turn that does not end in time is the scenario error "Pipecat bot did not finish its turn within {N} s.". Set `pipecat.turn_timeout_s` (seconds, both modes) for slow tools or long spoken replies; without it the timeout is `voice_config.timeout_s` in voice mode, else 30 s. What the bot said and the tools it called before the timeout are kept in `run.json`.
- A tool call whose handler raised is recorded as `success=False`, with the bot's "Error executing function call [NAME]: ..." text as its error.
