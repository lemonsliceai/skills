# New app — LemonSlice avatar on LiveKit from zero

Use this when there is no LiveKit agent yet. Output: a running local app with a ringing UI, a server-side token route, and a worker that drives a LemonSlice avatar.

## Pick a starter

| User wants | Clone | Worker |
| --- | --- | --- |
| A web app (default) | `lemonslice-examples/03-livekit-app-python` | Python |
| A web app, Node everywhere | `lemonslice-examples/04-livekit-app-nodejs` | Node |
| Agent only, test in LiveKit's console | `lemonslice-examples/02-livekit-playground-demo` | Python |
| Avatar in a Zoom / Meet / Teams / Webex call | `lemonslice-examples/07-livekit-zoom` | Python |

Default to `03`. All of them ship with a ringing UI (where there is a UI), a `/api/token` route that dispatches the agent, and the avatar already attached. Do not switch the user to the embeddable widget; they asked for an app.

## Before writing code

1. Tell the user to collect these and wait for confirmation. Do not clone, install, or run yet.

| Variable | Source |
| --- | --- |
| `LIVEKIT_URL`, `LIVEKIT_API_KEY`, `LIVEKIT_API_SECRET` | https://cloud.livekit.io/ → project → Settings → Keys |
| `LEMONSLICE_API_KEY` | https://lemonslice.com/developers |
| `AGENT_NAME` | Any dispatch name; `lemonslice` is fine |

2. STT, LLM, and TTS run on LiveKit Inference with the LiveKit keys. No OpenAI, ElevenLabs, or Deepgram key is needed unless the user asks for a specific provider.
3. Tooling: Node 20+, and `uv` for the Python worker.

## Setup (`03-livekit-app-python`)

```bash
git clone https://github.com/lemonsliceai/lemonslice-examples.git
cd lemonslice-examples/03-livekit-app-python
cp .env.example .env.local        # user pastes the values above
```

Both Next.js and the worker read `.env.local` at the repo root.

### Update the worker

`agent/pyproject.toml`: raise `livekit-agents[lemonslice]` to `~=1.8` and add `livekit-plugins-ai-coustics~=0.3`. Then rewrite the session in `agent/src/agent.py` to LiveKit's current Inference cascade:

```python
from livekit.agents import Agent, AgentServer, AgentSession, TurnHandlingOptions, inference, room_io, utils
from livekit.plugins import ai_coustics, lemonslice

AGENT_IMAGE_URL = "https://…"   # keep the starter's image until the user has one

session = AgentSession(
    stt=inference.STT(model="assemblyai/universal-3-6-pro", language="en"),
    llm=inference.LLM(model="google/gemma-4-31b-it"),
    tts=inference.TTS(
        model="fishaudio/s2.1-pro",
        voice="9a9cf47702da476aa4629e2506d4a857",   # Hannah; pick a voice that fits the image
    ),
    turn_handling=TurnHandlingOptions(
        turn_detection=inference.TurnDetector(),
        interruption={"mode": "adaptive"},
    ),
)

await ctx.connect()

avatar = lemonslice.AvatarSession(
    agent_image_url=AGENT_IMAGE_URL,
    agent_prompt="A person talking.",
)
session_id = await avatar.start(session, room=ctx.room)

await session.start(
    room=ctx.room,
    agent=Assistant(),
    room_options=room_io.RoomOptions(
        audio_input=room_io.AudioInputOptions(
            noise_cancellation=ai_coustics.audio_enhancement(
                model=ai_coustics.EnhancerModel.QUAIL_VF_S
            ),
        ),
        audio_output=False,
    ),
)

await utils.wait_for_agent(ctx.room)
await session.generate_reply(instructions="Greet the user briefly.")
```

Replacement models come from https://docs.livekit.io/agents/models/. Change them only when asked.

### Personality

Ask what the avatar is for and who it talks to, then write `ASSISTANT_INSTRUCTIONS`. Replies are spoken, so: one to three sentences, plain text, no markdown or emojis, spell out numbers.

### Image

Keep the starter's image until the user gives one. When they do, it must be a public `https://` URL (tips: https://lemonslice.com/docs/prompting-guide/avatar-image-tips). `agent_prompt` sets mood ("a warm, upbeat person"); it is not for gestures.

### Install and run

```bash
npm install
cd agent && uv sync && cd ..
npm run dev:all                   # Next.js + worker; or `npm run dev` and `npm run dev:agent` in two terminals
```

Open http://localhost:3000, join, talk.

## Setup (`04-livekit-app-nodejs`)

Same layout; the worker is `agent/src/main.ts` using `@livekit/agents` and `@livekit/agents-plugin-lemonslice`. The avatar block is:

```ts
const avatar = new lemonslice.AvatarSession({
  agentImageUrl: AGENT_IMAGE_URL,
  agentPrompt: 'A person talking.',
});
await avatar.start(session, ctx.room);

await session.start({
  agent: new Assistant(),
  room: ctx.room,
  inputOptions: { noiseCancellation: BackgroundVoiceCancellation() },
  outputOptions: { audioEnabled: false },
});

await waitForParticipant({ room: ctx.room, identity: 'lemonslice-avatar-agent' });
session.generateReply();
```

Models: `new inference.STT({ model: 'assemblyai/universal-3-6-pro', language: 'en' })`, `new inference.LLM({ model: 'google/gemma-4-31b-it' })`, `new inference.TTS({ model: 'fishaudio/s2.1-pro', voice: '…' })`.

## How the pieces connect

**Token route.** `src/app/api/token/route.ts` signs a short-lived JWT with `LIVEKIT_API_SECRET` and returns `{ token, serverUrl, room }`. The browser never sees the secret. The route attaches a `RoomConfiguration` with `RoomAgentDispatch({ agentName: AGENT_NAME })`, which is what makes the worker join that room.

**Dispatch.** The worker registers under `AGENT_NAME` (`@server.rtc_session(agent_name=…)`). That makes dispatch explicit: it only joins rooms whose token asks for it. If you build your own frontend, keep the `RoomAgentDispatch` in the token or the agent never shows up. Removing `agent_name` makes the worker join every room in the LiveKit project, which is wrong outside a solo sandbox.

**Readiness.** `src/components/AgentCallUI.tsx` stays in a ringing state until `LiveKitAvatarReadyWatcher` (from `@lemonsliceai/avatar/livekit-react`) reports the first rendered frame, then switches to the call layout. It listens for the avatar participant leaving to return to the idle state, and for a `startup_failure` message on `lemonslice/message` to bail out cleanly if the worker fails to start.

**Audio.** `audio_output=False` because the LemonSlice participant publishes the agent's voice in sync with video. Leaving it on gives the user two copies of every reply.

## Testing surfaces

| Surface | Real room? | Avatar shows? |
| --- | --- | --- |
| `python agent.py console` (terminal) | No, local mock | Never, silently |
| `python agent.py dev` + the app at localhost:3000 | Yes | Yes |
| `python agent.py dev` + LiveKit Cloud → Agents → Console (pick `AGENT_NAME`) | Yes | Yes |

Use the app or the Cloud console. Say "terminal `console` mode" when warning the user, so they do not avoid the dashboard console that actually works.

## Deploy

- Next.js app: Vercel or similar, with `LIVEKIT_URL`, `LIVEKIT_API_KEY`, `LIVEKIT_API_SECRET`, `AGENT_NAME`.
- Worker: LiveKit Cloud agent hosting (`lk agent create` from `agent/`) or your own host, with the same LiveKit vars plus `LEMONSLICE_API_KEY`.
- Production hardening (ringing UI, disconnect handling, pipeline error events, timeouts, latency budget): https://lemonslice.com/docs/reference/production-checklist
