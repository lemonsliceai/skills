# LemonSlice Skills

[Agent skills](https://agentskills.io) for building with [LemonSlice](https://lemonslice.com). Each skill lives in `skills/<name>/` with a `SKILL.md` entry point, so they are discoverable by [`skills`](https://skills.sh) and other skill finders.

## Skills

| Skill | Description |
| --- | --- |
| [lemonslice-livekit-app](skills/lemonslice-livekit-app/) | Add a LemonSlice video avatar to a LiveKit agent (Python or Node), or build one from scratch. Covers keys, the voice cascade, the ringing UI, CWM-1 behavior, mid-call image changes, and troubleshooting |

## Install

With the [skills CLI](https://skills.sh) (works with Claude Code, Cursor, Codex, Copilot, and others):

```sh
npx skills add lemonsliceai/skills
```

Or install a single skill:

```sh
npx skills add lemonsliceai/skills --skill lemonslice-livekit-app
```

Then ask your coding agent something like:

> Add a LemonSlice video avatar to a real-time voice app.

The skill checks what you already have, then either attaches the avatar to your existing LiveKit agent or scaffolds the starter app: keys, voice cascade, avatar image, ringing UI, and optionally CWM-1 for a character that reacts and emotes.

## License

[MIT](LICENSE)
