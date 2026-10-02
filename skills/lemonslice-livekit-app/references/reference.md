# Reference — LemonSlice LiveKit plugin, control endpoint, troubleshooting

Package: Python `livekit-agents[lemonslice]` (`from livekit.plugins import lemonslice`); Node `@livekit/agents-plugin-lemonslice`. Requires `LEMONSLICE_API_KEY` in the worker's environment (or `api_key=`).

## `lemonslice.AvatarSession(...)`

Python names; Node uses camelCase (`agentImageUrl`, `agentPrompt`, `idleTimeout`, `avatarParticipantIdentity`). In Node, `model`, `action_engine`, `aspectRatio`, `responseDoneTimeout`, and `simulcast` go inside `extraPayload: { … }`.

| Parameter | Default | Notes |
| --- | --- | --- |
| `agent_image_url` | — | Public `https://` URL. Exactly one of `agent_image_url` / `agent_image` / `agent_id` is required. |
| `agent_image` | — | Local image (PIL image in Python; file path or `Buffer` in Node). The plugin uploads it. |
| `agent_id` | — | Avatar saved in the LemonSlice dashboard; its prompts and idle timeout become the defaults. Optional path; most integrations use a URL. |
| `agent_prompt` | `"a person talking"` | Demeanor while speaking: mood and energy, not gestures. Ignored under CWM-1. |
| `agent_idle_prompt` | `"a serious person"` | Demeanor while silent. Ignored under CWM-1. |
| `idle_timeout` | `60` | Seconds of silence before the avatar leaves. Resets while the avatar talks. `-1` disables; then you must end sessions yourself. |
| `response_done_timeout` | — | Seconds without new audio before a reply is treated as finished. Set `0.8` with Gemini Live S2S; otherwise leave unset. |
| `model` | flagship | `"cwm-1"` / `"cwm-1-lite"` (Character World Model with the emotion engine), `"flash"` (fastest), `"lite"` (cheapest, lower resolution, `2x3`/`1x1` only), `"pro"`. See Model variants. |
| `action_engine` | off | CWM-1 only. `"natural"` lets LemonSlice choose actions and emotions from the conversation. |
| `aspect_ratio` | `"2x3"` | `"2x3"`, `"9x16"`, `"1x1"`. |
| `simulcast` | off | Publish several resolutions so subscribers get what their bandwidth allows. |
| `avatar_participant_identity` | `"lemonslice-avatar-agent"` | Must be unique per avatar when several share a room. |
| `avatar_participant_name` | `"lemonslice-avatar-agent"` | Display name. |
| `api_key`, `api_url`, `conn_options` | env / default | Rarely needed. |

### `await avatar.start(session, room, *, livekit_url=None, livekit_api_key=None, livekit_api_secret=None) -> str`

Rebinds `session.output.audio` to the avatar, mints a LiveKit token for the avatar participant, and calls the LemonSlice session API. Returns the **`session_id`**; store it for control calls. LiveKit creds fall back to `LIVEKIT_URL` / `LIVEKIT_API_KEY` / `LIVEKIT_API_SECRET`. Call it **before** `session.start()`.

### Several avatars in one room

One `AgentSession` per avatar, each `AvatarSession` with its own `avatar_participant_identity`. Start all avatars with `asyncio.gather` / `Promise.all`, then start all sessions. Frontends should pick the avatar by identity, not "the only remote participant".

## Events on the `lemonslice` data topic

Both the worker (`room.on("data_received")`) and the browser (`RoomEvent.DataReceived`) can read these. Payload is JSON with a `type` field.

| `type` | When | Use |
| --- | --- | --- |
| `bot_ready` | LemonSlice is streaming A/V. `session_id` included. | Switch ringing → call UI (or use `LiveKitAvatarReadyWatcher`, which waits for the first rendered frame). |
| `idle_timeout` | Session ended for inactivity. | Reset UI to idle. |
| `error` | Pipeline error; has `error`, `fatal`. | Tear down on `fatal`. |
| `video_generation_error` | A video segment failed. | Log; usually transient. |
| `metric` | After each reply: `time_to_first_push`, `tts_audio_delay`. | `tts_audio_delay: true` means TTS/LiveKit did not deliver audio in real time; look upstream. |
| `image_change_complete` / `image_change_error` | After `update-image`. | Clear transitional UI, or surface the error. |

The Python starter also posts `{ "type": "startup_failure" }` on the `lemonslice/message` topic if the worker cannot start; the frontend disconnects and shows a retry.

## CWM-1

Character World Model 1 (`model="cwm-1"` or `"cwm-1-lite"`, Ultra and Enterprise tiers) has a built-in emotion engine that decides what the character does and feels. It works from any single image, with no per-character tuning. Under CWM-1, `agent_prompt` and `agent_idle_prompt` are ignored; behavior comes from the engine and from the actions you cue.

### Three ways to drive it

| Mode | Set up | Trigger | When |
| --- | --- | --- | --- |
| Automatic | `model="cwm-1", action_engine="natural"` | Nothing; LemonSlice picks from conversation state | Default. Life-like behavior with no custom logic. |
| LiveKit data message | `model="cwm-1"` (± `action_engine`) | Publish `{"action": "<name>"}` on topic `ls.action` to `lemonslice-avatar-agent` | You want specific beats with the lowest latency (~250 ms faster than REST). Works from the worker or the browser. |
| Control endpoint | `model="cwm-1"` (± `action_engine`) | `POST …/control` with `{"event": "action", "action": "<name>"}` | Non-LiveKit transports (Daily, Agora, WebSocket) or backend-triggered events. |

**Listening state (LiveKit, automatic mode).** By default the engine knows two avatar states: idling and responding. Publish the user's speech state on `ls.user-state` (`{"state": "speaking"}` / `{"state": "idle"}`) to the avatar participant to unlock a third, listening, so the character reacts while the user talks. Map `session.on("user_state_changed")`: `speaking` → `"speaking"`, `listening` → `"idle"`.

### Behavior rules

- A cued action pre-empts the one the engine is playing. The engine resumes afterwards if `action_engine` is set.
- One action at a time. `angry` and `arms_crossed` cannot overlap. Wait for the current one, or send `clear_action` first.
- Each action plays once, for a fixed duration, then the avatar returns to its default state. Send it again to hold it longer.
- `clear_action` (same channel as other actions) stops the current action immediately.
- Actions marked "silent" below read best while the avatar is not speaking. Everything else can fire anytime, including mid-sentence.

### Available actions

| Action | What it looks like | Best in silence |
| --- | --- | --- |
| `adjust_collar` | Adjusts collar or necklace | |
| `angry` | Angry expression | |
| `arms_crossed` | Crosses arms over chest | |
| `blow_kiss` | Blows a kiss | yes |
| `clasp_hands` | Clasps hands together | |
| `clear_action` | Ends the current action | |
| `deep_breath` | Slow, deep breath | yes |
| `excited` | Excited expression | |
| `explain` | Talking gesture with hands | |
| `explain_finger_up` | Raises a finger to make a point | |
| `explain_palms_up` | Talking gesture, palms up | |
| `glance_down` | Glances down | |
| `glance_down_then_rub_eye` | Glances down, then rubs eye | |
| `glance_right` | Glances right | |
| `glance_side_down_side` | Glances to the side, then down | |
| `glance_side_up` | Glances to the side, then up | |
| `glance_sideways` | Glances to the side | |
| `glance_up` | Glances up | |
| `hands_on_hip` | Hands on hips | |
| `hold_chin` | Rests hand on chin | |
| `laugh` | Laughs | |
| `lean_long` | Sustained forward lean | |
| `lean_slow` | Slow, gentle lean forward | |
| `lean_strong` | Pronounced lean forward | |
| `listen` | Acknowledging nods | yes |
| `looking_around` | Looks around the room | |
| `move_more` | Pronounced body movement | |
| `move_subtle` | Subtle body movement | |
| `phone_call` | Talks on the phone | |
| `raise_eyebrow` | Raises an eyebrow | |
| `rub_eye` | Rubs one eye | |
| `rub_eye_then_touch_hair` | Rubs eye, then touches hair | |
| `shoulder_shimmy` | Playful shoulder shimmy | yes |
| `smile` | Warm smile | yes |
| `texting` | Texts on the phone | |
| `tilt_head_right` | Tilts head | yes |
| `touch_chin` | Touches chin thoughtfully | |
| `touch_hair` | Touches or strokes hair | |
| `turn_left` | Turns body left | yes |
| `wave` | Waves hello or goodbye | |
| `yawn` | Yawns | yes |

Example videos for each: https://lemonslice.com/docs/reference/cwm-1

### Worker helper and LLM tools (LiveKit)

```python
import json
from livekit.agents import Agent, RunContext, function_tool, get_job_context

LS_ACTION_TOPIC = "ls.action"
LS_AVATAR_IDENTITY = "lemonslice-avatar-agent"

async def perform_action(action: str) -> None:
    room = get_job_context().room
    await room.local_participant.publish_data(
        json.dumps({"action": action}).encode(),
        reliable=True,
        topic=LS_ACTION_TOPIC,
        destination_identities=[LS_AVATAR_IDENTITY],
    )

class Assistant(Agent):
    @function_tool()
    async def wave(self, context: RunContext) -> str:
        """Wave at the user. Use when saying hello or goodbye."""
        await perform_action("wave")
        return "Waved."

    @function_tool()
    async def react(self, context: RunContext, emotion: str) -> str:
        """Show an emotion on the avatar's face and body.

        Args:
            emotion: one of smile, laugh, excited, angry, raise_eyebrow
        """
        if emotion not in {"smile", "laugh", "excited", "angry", "raise_eyebrow"}:
            return "Unknown emotion."
        await perform_action(emotion)
        return "Done."
```

Keep the tool's allowed list short and named for intent. A long enum of 40 actions makes the LLM worse at choosing. With `action_engine="natural"` on, tools are for the beats you want guaranteed, not for everything.

### At call start

```python
avatar = lemonslice.AvatarSession(agent_image_url=AGENT_IMAGE_URL, model="cwm-1", action_engine="natural")
session_id = await avatar.start(session, room=ctx.room)
await session.start(room=ctx.room, agent=Assistant(), room_options=…)

await utils.wait_for_agent(ctx.room)       # avatar is in the room, so the message lands
await perform_action("wave")
await session.generate_reply(instructions="Greet the user.")
```

Sending to `ls.action` before the avatar participant has joined drops the message silently. Always wait for it first.

## Control endpoint

```
POST https://lemonslice.com/api/liveai/sessions/{session_id}/control
Content-Type: application/json
X-API-Key: <LEMONSLICE_API_KEY>
```

Response on success: `{ "success": true }`. Call it from the worker, your backend, or a tool; never from the browser (it would expose the key).

| Body | Effect |
| --- | --- |
| `{ "event": "update-image", "image_url": "https://…" }` | New reference image, ~600 ms. |
| `{ "event": "update-image", "image_base64": "<base64 or data URL>" }` | Same, skipping the server-side fetch (~0.4 s faster). Decoded size < 900 KB. |
| `{ "event": "action", "action": "wave" }` | Trigger a CWM-1 action (see CWM-1). On LiveKit, `ls.action` is faster. |
| `{ "event": "action", "action": "clear_action" }` | Stop the current CWM-1 action. |
| `{ "event": "update-agent-prompt", "agent_prompt": "look excited" }` | Change speaking demeanor. Ignored under CWM-1. |
| `{ "event": "update-idle-prompt", "idle_prompt": "bored, looking around" }` | Change idle demeanor. Ignored under CWM-1. |
| `{ "event": "reset-idle-timeout" }` | Restart the idle clock. |
| `{ "event": "terminate" }` | End the avatar session only. The room and agent stay up. |

Notes:

- `update-image` interrupts any audio that is playing. Send it during silence when you can. The avatar keeps listening and talking throughout; no need to tear down video.
- For prompt-generated images (an image-edit model making the new picture), the slow part is the generation, not LemonSlice. Show a transitional state while generating, send `update-image` when done, clear it on `image_change_complete`.
- Keep a small library of pre-hosted images for predictable tool-driven changes.
- Changing the whole character also means changing TTS voice and LLM instructions on your side; LemonSlice only changes the face.

### Helper (Python)

```python
import httpx, os

LEMONSLICE_CONTROL = "https://lemonslice.com/api/liveai/sessions/{sid}/control"

async def lemonslice_control(session_id: str, body: dict) -> bool:
    async with httpx.AsyncClient(timeout=10.0) as client:
        r = await client.post(
            LEMONSLICE_CONTROL.format(sid=session_id),
            headers={"X-API-Key": os.environ["LEMONSLICE_API_KEY"]},
            json=body,
        )
    return r.is_success
```

### As LLM tools

```python
from livekit.agents import Agent, function_tool

class Assistant(Agent):
    def __init__(self, session_id: str) -> None:
        super().__init__(instructions=INSTRUCTIONS)
        self._sid = session_id

    @function_tool()
    async def change_outfit(self, outfit: str) -> str:
        """Change the avatar's outfit. Use when the user asks to see a different look."""
        url = OUTFIT_IMAGES.get(outfit)
        if not url:
            return f"No image for {outfit}."
        ok = await lemonslice_control(self._sid, {"event": "update-image", "image_url": url})
        return "Done." if ok else "The outfit change failed."
```

## Model variants

| `model` | Use when | Trade-offs |
| --- | --- | --- |
| flagship (unset) | Default. Natural hand gestures while speaking. | — |
| `"cwm-1"` | The character should react and emote: automatic behavior or cued actions. | Ultra and Enterprise. `agent_prompt` / `agent_idle_prompt` ignored. |
| `"cwm-1-lite"` | CWM-1 behavior at lower cost. | Ultra and Enterprise. |
| `"flash"` | Latency matters most. 200–300 ms lower time-to-first-frame. | Ultra and Enterprise. |
| `"lite"` | High-volume consumer apps where per-minute cost dominates. | Lower resolution; `2x3` and `1x1` only. |
| `"pro"` | Highest quality. | Higher cost. |

## Timeouts

- Idle: `idle_timeout` (default 60 s), resets while the avatar speaks. Use `reset-idle-timeout` if the user is engaged but silent (reading, typing).
- Session cap: 30 minutes by default. Longer calls need LemonSlice to raise it.
- Shutdown order: `ctx.room.disconnect()` ends room + avatar; `ctx.shutdown()` ends the job + avatar but leaves the room; `terminate` ends only the avatar. An un-ended session runs until idle timeout or the cap, and bills until then.

## Errors you will see

| Error | Cause | Fix |
| --- | --- | --- |
| `LemonSliceException: LEMONSLICE_API_KEY must be set` | Key missing in the worker env. | Set it in `.env.local` (worker side). |
| `LemonSliceException: Missing agent_id or agent_image_url` | No image source. | Pass `agent_image_url` (or `agent_image` / `agent_id`). |
| `LemonSliceException: Only one of agent_id or agent_image_url can be provided` | Both set. | Pick one. |
| `LemonSliceException: livekit_url, livekit_api_key, and livekit_api_secret must be set` | LiveKit creds missing at `start()`. | Set the env vars or pass them to `start()`. |
| `APIStatusError … status_code=401` | Invalid LemonSlice key. | Regenerate at https://lemonslice.com/developers. |
| `APIStatusError … status_code=402` | No active subscription / insufficient funds. | Add a plan or credits in the LemonSlice dashboard. |
| `APIStatusError … status_code=4xx` with a body about the image | Image URL not fetchable or not an image. | Public `https://` URL; check it opens in an incognito window. |
| `APIConnectionError: Failed to call LemonSlice API after all retries` | Network / 5xx after retries. | Retry; check status; raise `conn_options` timeout if the network is slow. |
| `AgentSession` `error` event with `recoverable == False` | STT/LLM/TTS died. | Pipeline problem, not avatar. End the session cleanly. |

## Symptom → fix

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Avatar never appears, no error | Terminal `console` mode (mock room); or `avatar.start()` called after `session.start()`; or no dispatch for `agent_name`. | Run `dev`, test in the app or LiveKit Cloud Agent Console; attach before start; put `RoomAgentDispatch` in the token. |
| Black frame, then video | UI switched on `ParticipantConnected`. | Switch on first frame (`LiveKitAvatarReadyWatcher`) or `bot_ready`; show a ringing state until then. |
| Every reply is heard twice | Session also publishes audio. | `audio_output=False` (Python) / `outputOptions.audioEnabled: false` (Node). |
| Avatar joins, mouth does not move | Session audio not routed to the avatar: output reassigned after `start()`, or TTS producing nothing. | Do not touch `session.output.audio`; verify TTS without the avatar. |
| Avatar greets an empty room | `generate_reply()` before the avatar joined. | `await utils.wait_for_agent(ctx.room)` / `waitForParticipant` first. |
| Avatar leaves after ~60 s of silence | Idle timeout. | Raise `idle_timeout` or send `reset-idle-timeout` while the user is engaged. |
| Calls end at exactly 30 minutes | Session cap. | Contact LemonSlice to raise it. |
| Stutter / clipped endings with a realtime model | Reply-end detection. | `response_done_timeout=0.8` (Gemini Live); otherwise contact support. |
| Replies feel slow | Usually the LLM (tools, reasoning). | Measure each stage; move slow work off the critical path; `model="flash"` if available. |
| Video looks soft on big screens | Subscriber picked a low layer, or `lite` model. | Enable `simulcast`; render large; avoid `lite` for hero video. |
| Two avatars, one disappears | Same participant identity. | Unique `avatar_participant_identity` per avatar. |
| `update-image` returns success but nothing changes | Listening for the wrong event, or image rejected after accept. | Watch `image_change_complete` / `image_change_error` on the `lemonslice` topic. |
| CWM-1 action does nothing | Session not started with `model="cwm-1"`; message sent before the avatar joined; wrong topic or destination; or an unknown action name. | Set `model`; `wait_for_agent` first; topic `ls.action` to `lemonslice-avatar-agent`; use a name from the action list. |
| Second action ignored | CWM-1 plays one action at a time. | Wait for the first to finish, or send `clear_action` first. |
| `agent_prompt` has no effect | Session uses CWM-1. | Expected. Steer with `action_engine` and cued actions instead. |
| CWM-1 session rejected | Plan does not include CWM-1. | Ultra or Enterprise tier. |
| Avatar only idles or responds, never seems to listen | User state not published. | Publish `ls.user-state` from `user_state_changed`. |
| Works locally, not deployed | Env vars missing on the host. | All four LiveKit/LemonSlice vars on the worker; LiveKit vars on the web host. |
