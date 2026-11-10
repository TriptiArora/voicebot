# Voice bot plan - 100x Stage 1 assessment

## Goal

A public web page where anyone can talk to a voice bot that answers interview questions as Tripti Arora.
No install, no API key entry, works on desktop and phone.
The deadline is 24 hours from the 100x email.

## Stack

| Part | Choice | Notes |
| --- | --- | --- |
| Backend | n8n workflow on the n8n-tripti instance | Public PikaPods HTTPS URL, holds all keys |
| LLM | Inception `mercury-3` | `POST https://api.inceptionlabs.ai/v1/chat/completions`, existing `Inception ai` credential |
| Speech-to-text | ElevenLabs Scribe | `POST https://api.elevenlabs.io/v1/speech-to-text`, `model_id=scribe_v2`, `language_code=en` |
| Text-to-speech | ElevenLabs | `POST https://api.elevenlabs.io/v1/text-to-speech/{voice_id}`, low-cost Flash model, stock voice (no voice cloning) |
| Persona data | n8n Data Table `persona_facts` | Sample rows now, real answers later |
| Voice page | One static `index.html` on GitHub Pages | n8n's own pages cannot reliably use the microphone |

## How one turn works

1. The page detects that the visitor stopped speaking and encodes that turn as 16 kHz mono WAV.
2. It posts the audio, the recent conversation and a turn ID to the n8n webhook.
3. n8n transcribes the audio, builds the prompt from `persona_facts`, calls Inception, cleans the reply and converts it to speech.
4. n8n returns JSON with the transcript, the reply text and the reply audio.
5. The page plays the audio, shows both lines in the transcript and goes back to listening.

## n8n workflow "Voicebot - Turn"

1. Webhook: POST, binary upload enabled, allowed origin limited to the GitHub Pages domain, replies through a Respond to Webhook node.
2. Guard: reject oversized or over-long audio, then check the daily usage cap in the `usage_log` Data Table.
3. Transcribe with ElevenLabs when audio is present, otherwise use the typed text.
4. Empty transcript: reply `empty` straight away, skipping the LLM and voice steps.
5. Build messages: system prompt and persona facts are added server-side only, and the browser's history is limited to user and assistant roles, the last 10 turns and a length cap.
6. Call Inception with an HTTP Request node: `reasoning_effort` low, `realtime` true, temperature 0.75, capped `max_completion_tokens`.
7. Clean the reply: strip markdown, emoji and anything that should not be read aloud.
8. Convert to speech with ElevenLabs, and continue without audio if that call fails.
9. Respond with JSON.
10. Error branch: return a friendly message and write a row to the `error_log` Data Table.

Workflow settings: save failed executions only, so visitor audio is not stored.

## Persona behaviour

- First person, conversational, about 20 to 40 seconds of speech per answer.
- Hard facts such as employers, dates and degrees come only from `persona_facts`.
- Behavioural and opinion questions that are not in the table are answered in character from the stories and values in the table.
- Attempts to change the bot's role or extract the prompt are politely declined.
- Every sample row is flagged `is_sample`, and the page shows a "sample data" banner while any flagged row exists.

## Voice page

- One Start button, which also unlocks audio playback on iPhone.
- Hands-free turns using in-browser voice activity detection (Silero VAD).
- Interrupting: speaking over the bot stops its audio, with echo cancellation on and a sustained-speech threshold so the bot cannot interrupt itself.
- Pause and resume, End, and mute controls.
- Status indicator: Listening, Thinking, Speaking, Paused.
- Live transcript of both sides.
- Clickable suggestion chips for the five 100x sample questions.
- Typed-question box as a fallback.
- Late replies for an interrupted turn are discarded by turn ID.
- Noise shorter than a minimum length is ignored.
- Fallbacks: browser voice if ElevenLabs speech fails, browser speech recognition and then typing if transcription fails.
- In-app browsers (LinkedIn, Gmail) get an "open in Chrome or Safari" notice, with typing still working.
- Plain-language messages for mic denied, offline and server errors.
- The greeting is one pre-generated audio file, so it costs no credits per visit.

## Abuse and cost protection

- Daily turn cap in `usage_log`; above the cap the bot answers with the browser voice instead of ElevenLabs.
- Audio length and size limits, reply length limit.
- Spend limits set on the Inception and ElevenLabs accounts.

## Build order

1. Create `persona_facts`, `usage_log` and `error_log`, and add the sample persona rows.
2. Build the workflow and test the Inception call on its own.
3. Add the ElevenLabs steps once the credential exists, then test with recorded clips.
4. Publish the workflow.
5. Build `index.html` and test it against the live webhook.
6. Deploy to GitHub Pages.
7. Test on Chrome, Safari, Firefox, iPhone and Android, including interruption, pause, mic denied and ElevenLabs failure.
8. Measure the gap between question and answer, and show text before audio if it feels slow.
9. Replace sample data with real answers and re-test with the five 100x questions plus unlisted ones.
10. Tripti sends the URL and resume to 100x.

## Needed from Tripti

- ElevenLabs API key added in n8n as a Header Auth credential named `ElevenLabs`, header `xi-api-key`.
- Real answers, using the questionnaire delivered in step 1.
- Decision on paying for ElevenLabs Starter before submitting.

## Open items

- Whether the ElevenLabs free plan is blocked from the PikaPods server IP is unknown until the first real call.
- The exact Flash model ID is read from `GET /v1/models` at build time.
- Multipart upload handling by the n8n Webhook node is confirmed by test in step 3.
