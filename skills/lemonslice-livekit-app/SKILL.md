---
name: lemonslice-livekit-app
description: |
  Add a LemonSlice video avatar to a LiveKit Agents app, or build one from scratch. LemonSlice turns any single image into a real-time, lip-synced talking avatar that joins the LiveKit room as a participant. Use when: (1) the user wants a talking avatar, a face on a voice agent, a lip-synced character, or a video agent on LiveKit — even if they do not name LemonSlice; (2) they mention lemonslice.AvatarSession, livekit-agents[lemonslice], @livekit/agents-plugin-lemonslice, or the lemonslice-examples repo; (3) they want the avatar to move and emote on its own or on cue (CWM-1, Character World Model, emotion engine, waving, smiling, laughing, leaning in), or to change its image or demeanor mid-call (update-image); (4) they are moving from another LiveKit avatar plugin (Tavus, HeyGen, Hedra, Beyond Presence, Anam, Synthesia); (5) the avatar never shows up, shows a black frame, drops after a fixed number of minutes, or the audio doubles. Works for Python and Node.js agents.
license: MIT
metadata:
  author: lemonslice
  version: "1.0.0"
---

# LemonSlice avatar on LiveKit

LemonSlice is an avatar plugin for LiveKit Agents, in Python and Node.js. You give it one image (any face, any art style, no training step) and it joins the room as a second participant that publishes lip-synced video. Your `AgentSession` keeps doing STT, LLM, and TTS; the plugin reroutes the session's audio to LemonSlice, which streams synced audio and video back into the room.

This skill works out what the user already has, picks one route, and implements it.

## Facts that shape every decision

- **Keys.** `LEMONSLICE_API_KEY` from https://lemonslice.com/developers, plus `LIVEKIT_URL`, `LIVEKIT_API_KEY`, `LIVEKIT_API_SECRET` from a LiveKit Cloud project. The LemonSlice key needs an active subscription; a key without one gets a `401`/`402` from the session API. Nothing runs without these, so confirm them before writing code.
- **Source image.** Exactly one of `agent_image_url` (public `https://` URL), `agent_image` (local file, uploaded by the plugin), or `agent_id` (an avatar saved in the LemonSlice dashboard). Default to `agent_image_url`; the dashboard is optional. Site paths like `/avatar.png` and `localhost` URLs fail because LemonSlice's servers fetch the image.
- **Both languages.** Python: `livekit-agents[lemonslice]`. Node: `@livekit/agents-plugin-lemonslice`. The frontend is just a LiveKit client.
- **Attach before start.** `await avatar.start(session, room)` must finish before `await session.start(...)`.
- **The avatar owns the audio.** Keep `audio_output=False` in `RoomOptions` (Python) or the equivalent in Node, and do not give the session another audio output afterwards. Otherwise the user hears the reply twice, or lip-sync dies.
- **Sessions time out.** Idle timeout defaults to 60 s and resets while the avatar talks (`idle_timeout`, `-1` disables). Hard cap is 30 minutes per session unless LemonSlice raises it.
- **Mid-call control is a REST call, not a plugin method.** Image and demeanor prompts change via `POST /api/liveai/sessions/{session_id}/control`. `avatar.start()` returns the `session_id` you need.
- **CWM-1 is the behavior model.** `model="cwm-1"` (or `"cwm-1-lite"`) switches the avatar to Character World Model 1, whose emotion engine decides what the character does and feels: waving, smiling, laughing, leaning in, glancing away. It can run fully automatically (`action_engine="natural"`) or take cues from your code. Under CWM-1, `agent_prompt` and `agent_idle_prompt` are ignored.
- **Plan-gated.** CWM-1 and `model="flash"` (fastest response) are on the Ultra and Enterprise tiers. Offer them; do not build on them until the user confirms their plan.

## Step 1: Find the starting point

Scan the codebase before asking anything.

| Look for | Where | Means |
| --- | --- | --- |
| `livekit-agents` / `livekit.agents` | `pyproject.toml`, imports | Python LiveKit agent → **Add** |
| `@livekit/agents` | `package.json` | Node LiveKit agent → **Add** |
| `livekit.plugins.lemonslice` / `agents-plugin-lemonslice` | imports | Already integrated → **Fix**, read `references/reference.md` |
| `livekit.plugins.tavus` / `bey` / `hedra` / `heygen` / `anam` / `synthesia` | imports | Other avatar plugin → **Replace** |
| `AgentSession(llm=<realtime model>)` | entrypoint | Speech-to-speech agent; avatar still attaches the same way |
| `AgentSession(stt=…, llm=…, tts=…)` | entrypoint | Cascade agent |
| `LEMONSLICE_API_KEY`, `LIVEKIT_*` | `.env*` | Which keys exist |
| `@lemonsliceai/avatar`, `LiveKitRoom`, `useVoiceAssistant` | frontend | A LiveKit frontend exists; the avatar will render there |
| `livekit-agents` version | lockfile | `~=1.8` for the current LiveKit Inference models and `TurnDetector` |

Then ask only what is still unknown, as one list:

```
To add the LemonSlice avatar I need to know:
1. Do you have a LemonSlice API key (https://lemonslice.com/developers) and a LiveKit Cloud project?
2. Is there an existing LiveKit agent (Python or Node), or are we starting fresh?
3. Which image should the avatar use? A public https URL is easiest; I can use a placeholder until you have one.
```

## Step 2: Pick one route

```
Plugin already imported and something is wrong?    → FIX      (references/reference.md, troubleshooting)
Working LiveKit agent, Python or Node?              → ADD      (below)
Agent uses another avatar plugin?                   → REPLACE  (below)
No agent at all?                                    → NEW APP  (references/new-app.md)
```

Say which route you picked and why in two sentences, then implement. Do not present a menu.

## Step 3: Implement

### ADD — existing agent

1. Install.
   - Python: `uv add "livekit-agents[lemonslice]~=1.8"` (or pip). Raise an older `livekit-agents` pin to `~=1.8` first.
   - Node: `pnpm add @livekit/agents-plugin-lemonslice`
2. Add `LEMONSLICE_API_KEY` to the agent's environment. It never goes to the frontend.
3. Between building the `AgentSession` and starting it:

Python:

```python
from livekit.plugins import lemonslice

avatar = lemonslice.AvatarSession(
    agent_image_url=AGENT_IMAGE_URL,        # public https URL
    agent_prompt="A friendly person talking.",  # demeanor while speaking, optional
)
session_id = await avatar.start(session, room=ctx.room)   # keep this for control calls

await session.start(
    room=ctx.room,
    agent=Assistant(),
    room_options=room_io.RoomOptions(
        audio_input=room_io.AudioInputOptions(
            noise_cancellation=ai_coustics.audio_enhancement(
                model=ai_coustics.EnhancerModel.QUAIL_VF_S
            ),
        ),
        audio_output=False,   # the avatar publishes the audio
    ),
)
```

Node:

```ts
import * as lemonslice from '@livekit/agents-plugin-lemonslice';

const avatar = new lemonslice.AvatarSession({
  agentImageUrl: AGENT_IMAGE_URL,
  agentPrompt: 'A friendly person talking.',
});
const sessionId = await avatar.start(session, ctx.room);

await session.start({ agent, room: ctx.room, outputOptions: { audioEnabled: false } });
```

4. Wait for the avatar before the first line, so it does not speak into an empty room:

```python
await utils.wait_for_agent(ctx.room)        # Python
await session.generate_reply(instructions="Greet the user briefly.")
```

```ts
await waitForParticipant({ room: ctx.room, identity: 'lemonslice-avatar-agent' });  // Node
await session.generateReply({ instructions: 'Greet the user briefly.' });
```

No frontend change is required to see the avatar: it is a normal LiveKit participant with a video track. For a good experience, though, do the frontend pattern in Step 4.

### REPLACE — from another avatar plugin

LiveKit's avatar plugins share the `AvatarSession` shape, so this is a small diff:

| Theirs | Ours |
| --- | --- |
| `from livekit.plugins import tavus` (`bey`, `hedra`, `heygen`, `anam`, `synthesia`) | `from livekit.plugins import lemonslice` |
| `tavus.AvatarSession(replica_id=…)`, `synthesia.AvatarSession(AvatarConfig(avatar_ids=[…]))`, etc. | `lemonslice.AvatarSession(agent_image_url="https://…")` |
| `TAVUS_API_KEY` / `SYNTHESIA_API_KEY` / … | `LEMONSLICE_API_KEY` |
| Provider-specific avatar ids | Any public image URL |

Keep their `await avatar.start(session, room=…)` call where it is. If they swapped avatars mid-call with a plugin method, port that to a `update-image` control call (Step 5). Remove the old plugin extra from dependencies.

### NEW APP

Read `references/new-app.md`. It covers the two starters (`03-livekit-app-python`, `04-livekit-app-nodejs`), the env file, the current LiveKit Inference cascade, dispatch, and the frontend.

## Step 4: Frontend — ring, then show

Avatar video takes a few seconds to arrive after the participant joins. Showing the video element on `ParticipantConnected` gives the user a black frame. Instead:

1. Play a ringing state while connecting.
2. Switch to the call UI when the first video frame renders. Use `@lemonsliceai/avatar`'s `LiveKitAvatarReadyWatcher` (React), or listen for `bot_ready` on the `lemonslice` data topic.
3. Return to an idle state on `ParticipantDisconnected` for the avatar identity (default `lemonslice-avatar-agent`), so the user can call again.

```tsx
import { LiveKitAvatarReadyWatcher } from "@lemonsliceai/avatar/livekit-react";

<LiveKitRoom serverUrl={serverUrl} token={token} connect>
  <LiveKitAvatarReadyWatcher onReady={() => setAvatarReady(true)} />
  {avatarReady ? <CallUI /> : <RingingUI />}
</LiveKitRoom>
```

Mint tokens server-side. If the worker sets `agent_name`, the token's room config must dispatch it (`RoomAgentDispatch`) or the agent never joins; see `references/new-app.md`.

## Step 5: Bring the character to life with CWM-1 (optional, Ultra/Enterprise)

Without CWM-1 the avatar already talks with natural hand gestures. CWM-1 adds a character that reacts: it smiles at good news, crosses its arms, glances away while thinking, waves goodbye. It works from the same single image, with no per-character tuning.

Pick one of three levels, from hands-off to fully scripted:

**1. Automatic (start here).** LemonSlice picks contextually appropriate behavior from the conversation.

```python
avatar = lemonslice.AvatarSession(
    agent_image_url=AGENT_IMAGE_URL,
    model="cwm-1",              # or "cwm-1-lite"
    action_engine="natural",
)
```

On LiveKit, also publish the user's speaking state, so the character visibly listens instead of only switching between idle and responding:

```python
@session.on("user_state_changed")
def _on_user_state(ev: UserStateChangedEvent) -> None:
    state = {"speaking": "speaking", "listening": "idle"}.get(ev.new_state)
    if state:
        asyncio.create_task(ctx.room.local_participant.publish_data(
            json.dumps({"state": state}).encode(),
            reliable=True,
            topic="ls.user-state",
            destination_identities=["lemonslice-avatar-agent"],
        ))
```

**2. Cued from your code, over the LiveKit data channel.** This is the lowest-latency way to trigger a specific action: ~250 ms faster than REST. Publish `{"action": "<name>"}` on topic `ls.action` to the avatar participant:

```python
async def perform_action(action: str) -> None:
    await ctx.room.local_participant.publish_data(
        json.dumps({"action": action}).encode(),
        reliable=True,
        topic="ls.action",
        destination_identities=["lemonslice-avatar-agent"],
    )
```

The browser can publish the same message (`room.localParticipant.publishData(…, { topic: "ls.action", destinationIdentities: [...] })`), for example from buttons.

**3. Cued over REST**, for non-LiveKit transports or calls from a backend: `POST …/control` with `{"event": "action", "action": "<name>"}`.

Levels 2 and 3 work with or without `action_engine`. A cued action pre-empts whatever the automatic engine is doing; the engine resumes when it finishes. One action plays at a time, once, for a fixed length. Send it again to hold it, or `clear_action` to stop it.

Typical wiring: wrap `perform_action` in LLM `@function_tool`s so the model waves on hello and goodbye, or smiles at good news. Trigger `wave` after `wait_for_agent` at call start. The full action list (`wave`, `smile`, `laugh`, `excited`, `angry`, `arms_crossed`, `lean_slow`, `listen`, `explain_finger_up`, and others), plus which ones read best during silence, is in `references/reference.md`.

## Step 6: Session control (optional)

Everything below is one `POST` with the `session_id` from `avatar.start()`:

```
POST https://lemonslice.com/api/liveai/sessions/{session_id}/control
X-API-Key: $LEMONSLICE_API_KEY
```

| `event` | Body | What it does |
| --- | --- | --- |
| `update-image` | `image_url` or `image_base64` (< 900 KB decoded) | Swap the avatar's look or whole identity mid-call, ~600 ms. Any audio playing is cut, so trigger during silence. Completion arrives as `image_change_complete` / `image_change_error` on the `lemonslice` data topic. |
| `action` | `action` | Trigger a CWM-1 action (see Step 5). On LiveKit, prefer the `ls.action` data message. |
| `update-agent-prompt` | `agent_prompt` | Change demeanor while speaking ("look excited"). Ignored under CWM-1. |
| `update-idle-prompt` | `idle_prompt` | Change demeanor while silent. Ignored under CWM-1. |
| `reset-idle-timeout` | — | Keep the session alive while the user is engaged but nobody is talking. |
| `terminate` | — | End the avatar without ending the room. |

Typical wiring: expose `update-image` as an LLM `@function_tool`, so the model changes outfit or scene when the conversation calls for it. Full payloads are in `references/reference.md`. The `09-realtime-image-change` example in `lemonslice-examples` is the end-to-end reference for image changes.

## Step 7: Verify

1. Start the worker in `dev` mode (`uv run python agent.py dev` / `pnpm dev:agent`). The terminal `console` mode is a local mock room; the avatar will never appear there and no error is raised. Test in a real room: the app's own frontend, or LiveKit Cloud → Agents → Console with the worker's `agent_name`.
2. Confirm the sequence: agent joins → avatar participant joins → `bot_ready` → first frame → the agent greets.
3. Speak. Lip-sync should match the reply audio with no echo or double audio.
4. Leave the call. The avatar participant should leave within the idle timeout; if the room keeps running, call `ctx.room.disconnect()` or send `terminate`.
5. Check that `LEMONSLICE_API_KEY` and `LIVEKIT_API_SECRET` are server-side only.

## Rules that apply on every route

- Keys first. Do not clone, install, or run until the user confirms their `.env`.
- Public image URL. Keep the starter's image until the user supplies one; never block on a dashboard avatar.
- Match the voice to the face. Pick a TTS voice whose gender and age fit the image and confirm with the user. On LiveKit Inference, Fish Audio `9a9cf47702da476aa4629e2506d4a857` (Hannah) is a solid female default; browse LiveKit's TTS catalog for others.
- If the conversation feels slow, measure the LLM first. Avatar video is rarely the bottleneck; `model="flash"` shaves 200–300 ms if the user has access.
- Use `agent_prompt` for mood, not choreography. For specific gestures and emotions, use CWM-1.

## References

- `references/new-app.md` — greenfield scaffold from the LemonSlice starters: env, install, cascade, dispatch, frontend, deploy.
- `references/reference.md` — `AvatarSession` parameters (Python/Node), CWM-1 action list and channels, events on the `lemonslice` topic, control endpoint payloads, model variants, errors, and the symptom → fix table.
- https://lemonslice.com/docs/livekit — plugin docs.
- https://lemonslice.com/docs/reference/cwm-1 — CWM-1 emotion engine, with example videos of every action.
- https://lemonslice.com/docs/reference/production-checklist — what LemonSlice has learned shipping avatars to production.
- https://github.com/lemonsliceai/lemonslice-examples — starters `02` (agent only), `03` (Python full app), `04` (Node full app), `07` (Zoom/Meet/Teams bot), `09` (real-time image change).
- `npx skills add livekit/agent-skills` — LiveKit's own skill for current agent APIs and model catalogs.
