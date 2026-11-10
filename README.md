# Voice bot - 100x Stage 1 assessment

A public web page where anyone can talk to a voice bot that answers interview questions as Tripti Arora.
The page is one static `index.html`; all keys and logic live in an n8n workflow.

## How it works

1. The page detects the end of each spoken question in the browser (Silero VAD) and posts it as 16 kHz WAV to the n8n webhook.
2. n8n transcribes it with ElevenLabs Scribe, answers with Inception `mercury-3` using the facts in the `persona_facts` Data Table, and converts the reply to speech with ElevenLabs.
3. The page plays the reply, shows the transcript and listens again.

See `PLAN.md` for the full design and the request/response contract.

## Files

| Path | What it is |
| --- | --- |
| `index.html` | The voice page, no build step |
| `n8n/voicebot-turn.workflow.json` | Export of the "Voicebot - Turn" workflow, importable into n8n |
| `n8n/persona_facts.json` | Tripti's knowledge base rows for the `persona_facts` Data Table |
| `PLAN.md` | Design, contract and build plan |

## Run the page locally

```bash
python3 -m http.server 8765 --bind 127.0.0.1
```

Then open http://localhost:8765 in Chrome or Safari.
Use `localhost`, not `127.0.0.1`, because the webhook only accepts allowed origins.

## Restore the backend from this repo

1. In n8n, create two credentials:
   - `Inception ai`: type OpenAI, with the Inception API key.
   - `Elevenlabs`: type Header Auth, header name `xi-api-key`, value the ElevenLabs API key.
2. Create three Data Tables:
   - `persona_facts`: `topic` (string), `content` (string), `is_sample` (boolean). Import the rows from `n8n/persona_facts.json`.
   - `usage_log`: `day` (string), `turns` (number), `cappedTurns` (number).
   - `error_log`: `timestamp` (date), `turnId` (string), `stage` (string), `message` (string).
3. Import `n8n/voicebot-turn.workflow.json` (Workflows, Import from file).
4. Re-select the two credentials on the "Transcribe Audio", "Ask Inception" and "Synthesize Speech" nodes.
5. Re-select the Data Tables on the "Read Usage Today", "Record Usage", "Load Persona Facts", "Log Failure" and "Log Speech Failure" nodes, because new tables get new ids.
6. Publish the workflow and update `CONFIG.endpoint` in `index.html` if the webhook URL changed.

## Limits worth knowing

- All limits, models and the voice id are in the workflow's "Voicebot Config" node.
- The ElevenLabs free plan covers roughly 40 spoken answers a month.
- The voice is Matilda (`XrExE9yKIg1WjnnlVkGX`), an ElevenLabs default voice. Default voices expire on 31 December 2026, and library voices need a paid plan for API use.
