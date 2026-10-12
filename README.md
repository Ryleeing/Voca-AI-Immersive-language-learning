
#This project was initiated at the GDM Hackathon in Stockholm, where it won second prize, and is now undergoing continuous development and iteration.

# Lost in Paris

**Learn French by living through it.**

Your phone is dead. Somewhere in Le Marais is the best croissant in Paris. With no maps, no translation app, and no battery, your only tool is your French.

A small, voice-first hackathon game: three AI character encounters, one complete mission, original Paris illustrations, and no accounts, database, or movement engine. The interface is English; the conversations are French. Maison Lumière is fictional.

## Setup

Use **Node.js 20.19+** (22 LTS recommended).

```bash
npm install
cp .env.example .env
```

Put your Google Gemini API key in `.env`:

```env
GEMINI_API_KEY=your_google_gemini_api_key_here
```

```bash
npm run dev
```

Open **http://localhost:5173**. Start the adventure and allow microphone access. The microphone stays open throughout each conversation and reconnects automatically for the next NPC. Speak naturally and pause for a response; speak over a character to interrupt them. The mic button toggles mute, not turn submission.

Only localhost or HTTPS supports browser microphone access. Chrome/Edge are recommended; Safari with AudioWorklet support also works. Headphones prevent NPC audio from feeding back into the microphone. Internet access and key access to **`gemini-3.8-live`** are required; the app never silently substitutes another model or fake conversation.

The key stays server-side. `.env` is ignored, and `.env.example` contains only a placeholder. Restart the server after changing the key. The health endpoint, `/api/health`, returns only whether a key is configured, never its value.

```bash
npm test
npm run build
npm start
```

Production runs at **http://localhost:3001** and serves `dist/`. `PORT` can override either local port. This is a local hackathon app, not a public multi-user service.

## Architecture

React + TypeScript, a single small Node HTTP/WebSocket server, Google's official **`@google/genai`** SDK, and esbuild. No external map API, authentication, database, separate text-to-speech service, or deployment framework.

The preferred Vite dependencies were unavailable in the implementation environment's offline package cache, so esbuild handles both development bundling and production builds. Development watches source files; **refresh the browser after edits**. Styling is plain CSS with responsive layouts and reduced-motion support. Google Fonts are optional: Georgia and Arial provide offline fallbacks.

```text
src/
  App.tsx                    Six-state journey and screen composition
  components/                Environment, NPC, mission, voice, hints, transitions
  data/scenes.ts             All three encounters + French configuration
  hooks/useGameState.ts      Idempotent mission state and transitions
  hooks/useGeminiLive.ts     Voice lifecycle, transcripts, playback, recovery
  lib/audio.ts               PCM encoding and sequential audio playback
  lib/microphone.ts          Capture lifecycle + ordered flush acknowledgement
  lib/speechStream.ts        Buffered prefix and silent/short-turn rejection
  types/game.ts              Shared scene and conversation types
server/
  agent.ts                   NPC prompt template, Live config, tool validation
  index.ts                   Secure SDK connection and WebSocket relay
  http.ts                    Public assets and key-free health endpoint
public/
  pcm-capture.js             Browser AudioWorklet microphone capture
  scenes/*.svg               Four original Paris scene illustrations
scripts/build.mjs            Production bundle
```

## How Gemini Live works

The browser connects to `/live` on the same origin. The server chooses the NPC's trusted local configuration and creates a Live session using **`gemini-3.8-live`** with native AUDIO responses. It enables input/output transcription and declares `complete_task`. No API key, system prompt, or arbitrary client-selected model is sent to the browser.

The Start button primes a shared Web Audio context before React mounts the first encounter. Capture and NPC playback reuse that unlocked graph across scenes, so microphone permissions and autoplay policies do not silently suspend a second context. The AudioWorklet continuously captures mono PCM. A lightweight client voice detector uses a 350 ms pre-speech buffer, 100 ms onset confirmation, adaptive noise thresholds and 1.2 seconds of silence to end a turn. It sends ordered activityStart → PCM → activityEnd to Gemini with server VAD disabled, following [Google's manual VAD guidance](https://ai.google.dev/gemini-api/docs/live-api/capabilities#disable-automatic-vad). Speaking starts a turn and immediately clears NPC playback; incoming audio is discarded while the learner speaks. No microphone clicks are needed. The waveform reflects actual captured microphone levels even during NPC playback. Muting or entering an optional encounter finishes the active turn and pauses forwarding; unmuting resumes detection. Scene completion guards count submitted speech turns and still rely on Gemini to judge semantic success. Continuous audio is not retained in memory.

Each encounter supplies its name, role, objective, local knowledge, semantic success criteria, and beginner French behavior. Grammar and wording need not match the hints. The bakery explicitly requires an order **and a separate “sur place / à emporter” answer**.

When Gemini calls `complete_task`, the server validates the scene ID, `success === true`, and duplicate state. A minimum speech-turn guard prevents completion from a greeting alone or a single bakery order. **Gemini, not that turn counter, judges communicative success.** The server sends the function response back to Gemini. The browser waits for the final response and queued audio, displays feedback, then animates to the next scene. A bounded fallback handles a missing final event. Old sockets, audio queues, capture contexts, streams, and timers are disposed when a scene changes or a connection is retried.

The integration follows Google's [Live API capabilities guide](https://ai.google.dev/gemini-api/docs/live-api/capabilities) and [official JavaScript SDK](https://github.com/googleapis/js-genai). Thinking configuration is deliberately omitted for this model.

## Why it is agentic

This is **not a fixed dialogue tree**. Each character has a role and contextual knowledge, understands free-form speech, chooses its own natural response, reasons about the learner's communicative success, and invokes a function that changes the environment.

```text
learner speech
  → Gemini interprets intent
  → NPC chooses a contextual spoken response
  → Gemini evaluates the objective
  → agent invokes complete_task()
  → environment advances automatically
  → a new contextual agent encounter begins
```

Only the three-scene sequence is predetermined. You can ask Camille about a good bakery in your own words, ask Julien for directions, and order naturally with Amélie. Hints scaffold speech and **never complete a task**.

## The journey

| Scene | NPC | Objective | Completion |
|---|---|---|---|
| `street_recommendation` | Camille | Find a good croissant/bakery | Understandable French recommendation request |
| `street_directions` | Julien | Reach Maison Lumière | Understandable French directions request |
| `bakery_order` | Amélie | One croissant and one coffee | Order both, then answer the follow-up |

`intro → street_recommendation → street_directions → bakery_order → completed`

## Go to my first party

The landing page also offers a beginner A1 party story. The learner arrives at a friendly apartment party, introduces themself to the host, meets another guest, then joins a short conversation about music and snacks.

| Scene | NPC | Objective |
|---|---|---|
| `party_arrival` | Emma | Greet the host and introduce yourself |
| `party_meet_someone` | Lucas | Ask someone their name or where they are from |
| `party_join_chat` | Inès | Share something you like and ask a simple question |

`intro → party_arrival → party_meet_someone → party_join_chat → completed`

The party uses the same voice conversation, hint, handbook, and progress systems as Lost in Paris, with its own beginner-level prompts, party illustration, mission map, and ending.

## Nightclub

Pick **Nightclub** on the scenario screen. It's 11:48 PM on Rue Oberkampf, and the bouncer at the fictional club Le Velours turns you away for wearing sneakers.

| Scene | NPC | Objective | Completion |
|---|---|---|---|
| `club_refused` | Karim (bouncer) | Find out why you can't get in | Understandable French "why?" → dress code + friperie tip |
| `club_boutique` | Margaux (friperie owner) | Shoes and a shirt | Ask for both, then answer "Quelle est votre pointure ?" |
| `club_return` | Karim (bouncer) | Get in this time | Ask again, then answer "Vous êtes combien ?" |

`intro → club_refused → club_boutique → club_return → completed`

A scene's `minTurns` (in `src/data/scenes.ts`) sets the server's minimum speech-turn guard.

Each scene has three hint states: no hint, scrambled word chips, and a useful sentence. Hints reset between encounters. Progress and game state are intentionally in memory; replay resets the journey.

## Demo recovery

- **Missing key:** the conversation panel explains how to configure `.env`.
- **Denied microphone:** enable it in browser site settings and retry.
- **Disconnected Gemini / no response:** a clear error and retry reconnect the current encounter. Retrying begins a fresh NPC conversation; repeat any unfinished request or order.
- **Blocked playback:** use **Enable sound**. Text and mission state remain available.
- **Network interruption:** no crash or automatic task completion.
- **Development controls:** open **http://localhost:5173/?dev=true** for a small bottom-left panel: mark complete / skip, restart current scene, or reset. Skipping uses the same completion transition as the normal journey. These controls are compiled out of production builds, even with `?dev=true`.

## Verification and remaining live checks

`npm run build` type-checks browser and server code and creates a production bundle. `npm test` checks the full scene sequence, native audio configuration, PCM sample preservation/endianness/clamping, complete utterance streaming and worklet flush ordering, completion safeguards (wrong scene, duplicate, false success, greeting-only, bakery follow-up, small-talk turns), and home routing / health / private-file protection.

The implementation environment prevents opening local listening sockets and starting Chrome. A key was added locally after initial setup, but the attempted external Live WebSocket check could not establish a session from this environment. Therefore a live microphone session, audible Gemini responses, semantic completion accuracy, and the browser visual acceptance test could not be verified here. The code includes the real Live integration; it is not backed by a simulated agent. Run the checklist below with your key before presenting.

1. Start the app; inspect the intro on desktop and mobile; start the adventure.
2. Start and allow the microphone. Without clicking the mic, speak French; confirm an audible reply. Interrupt Camille mid-sentence and check that old audio stops immediately. Pause briefly inside a sentence, then continue.
3. Click **Need a hint?**, then **Show me the sentence**. Neither click should complete the task.
4. Try “Vous connaissez une bonne boulangerie ?” rather than the hint sentence. Confirm Maison Lumière is recommended and the scene automatically advances.
5. Ask Julien how to reach Maison Lumière. Confirm short directions and arrival at the bakery.
6. Order both items. Confirm Amélie asks **Sur place ou à emporter ?** and stays in the scene until your second answer.
7. Confirm the three-conversation completion screen and replay.
8. Disconnect the network mid-conversation; verify the error and retry after restoring it.
9. Deny microphone access, check the message, allow it in site settings, and retry.
10. Verify `?dev=true` recovery controls; verify they are absent with `npm start`.

## Another scenario or language later

Keep the same Live hook, proxy, tool, and UI. Edit `src/data/scenes.ts` to provide new characters, objectives, hints, greetings, voices, location labels, and success criteria. The language configuration feeds the reusable server prompt. Add matching assets under `public/scenes/` and extend the explicit task IDs/sequence if the number of encounters changes. Translate all scenario content and local knowledge together; simply changing a language label is insufficient. Spanish, Italian, and Swedish are visual “coming soon” options only.

SVG illustrations are self-contained and work without image downloads. To replace them with generated photos, update the asset extension in `Environment.tsx` and place corresponding files under `public/scenes/`. The current backgrounds and independently layered NPCs remain fully usable without external imagery.

## A Paris that moves

The world now uses layered React/SVG/CSS choreography, with no physics engine or manual movement:

- **Ambient life:** four to six people per scene, with independently timed pedestrians, a cyclist, waiter, dog walker, café guests and baguette-carrying customers. Main NPCs breathe, shift weight and blink. Camera drift and a few pixels of pointer parallax provide depth; coffee steam and passing leaves add atmosphere.
- **Automatic travel:** `TravelSequence.tsx` runs approximately 6.9 seconds after validated mission completion. A farewell leads into background, storefront, pavement and foreground layers moving at different speeds. The bakery arrival includes an opening door, warm light and a swinging bell. The next main voice encounter starts after arrival, following the existing scene lifecycle.
- **Optional moments:** a seeded, weighted planner selects at most two unique events per run, from coffee spill, dropped scarf, pigeon mischief, dog greeting and lost visitors. Events appear after 9–14 seconds, expire when ignored, and never gate the mission. Use `?seed=42` for repeatable choreography, or replay without a seed for another selection.
- **Short side conversations:** choosing an available moment opens a separate Gemini Live session through the same proven capture/playback hook. The main session remains connected and idle, retaining its context. A server-validated local side-event ID selects a concise prompt; an invalid event/location is rejected. One meaningful French utterance can earn a star. The UI returns automatically after success or at most two submitted turns, and you can leave at any time. Side completion updates only optional memories, never mission progress.
- **Visible consequences:** the journey strip shows your next location; the story sidebar records recommendations, directions, orders and optional acts of kindness. Travel is triggered by the NPC's accepted completion action, not hint clicks. Earned Paris memories appear on the final screen.

Optional conversations use the same continuous Live architecture. The main session's input is suspended during a side encounter and resumes afterward without discarding its context.

Motion respects `prefers-reduced-motion`: moving traffic and camera effects stop, NPCs remain still, and travel becomes a brief 1.2-second arrival. Desktop gameplay remains viewport-sized with a bounded recent transcript and no scrolling.

New world components live under `src/components/world/`, choreography is styled in `src/world.css`, and event definitions/planning are in `src/data/ambient.ts`. To add a moment, give it scene eligibility, a weight, an objective, hints, and a short semantic success condition. Extend the local allowlist; never accept arbitrary browser-supplied system prompts.

For demo QA, let the first scene run 15 seconds with `?seed=42`, observe the ambient moment, ignore it once, then replay and try its French side interaction. Finish the main request in your own words, verify the full travel and bakery entry, and check that optional stars do not alter the four-task mission. Verify the main NPC resumes the same conversation after leaving an optional moment. Bakery-order scenes deliberately have no side event so the required order follow-up cannot be interrupted. Browser/live voice verification remains necessary on the demo machine.
