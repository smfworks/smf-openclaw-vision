# SMF OpenClaw companion bundle

A map of the OpenClaw-related repositories SMF Works maintains, and the one project SMF does **not** own.

## Ownership

> **OpenClaw is not an SMF Works product.**
>
> Install and run OpenClaw from the upstream project: [github.com/openclaw/openclaw](https://github.com/openclaw/openclaw). The project site is [openclaw.ai](https://openclaw.ai). Docs start at [docs.openclaw.ai](https://docs.openclaw.ai/start/getting-started).
>
> [github.com/smfworks/openclaw](https://github.com/smfworks/openclaw) is an SMF fork/mirror of that upstream repo. It is not the canonical source, and this bundle does not ask you to treat it as one.
>
> SMF Works authors three companion pieces only: the skills pack, the Mnemosyne memory plugin, and this vision guide.

SMF does not operate a hosted vision service. The camera path in [smf-openclaw-vision](https://github.com/smfworks/smf-openclaw-vision) runs on your phone and your tailnet.

## The three SMF pieces

| Piece | Repository | Purpose | Install pointer |
|-------|------------|---------|-----------------|
| Skills | [smfworks/smfworks-skills](https://github.com/smfworks/smfworks-skills) | Free OpenClaw skills pack for everyday file, document, and system tasks. | Follow **Installation** in that repo’s README. The documented one-liner fetches `install.sh` from the repo. Then install a free skill from the same README, for example `smfw install file-organizer`. |
| Memory | [smfworks/mnemosyne-openclaw](https://github.com/smfworks/mnemosyne-openclaw) | Offline SQLite memory plugin for the OpenClaw gateway. Full-text search stays on the machine that runs the gateway. | Follow **Installation** in that repo’s README: clone, `npm install`, `npm run build`, then `openclaw plugin load` with the plugin path, and enable the `mnemosyne` memory slot in OpenClaw config. |
| Vision | [smfworks/smf-openclaw-vision](https://github.com/smfworks/smf-openclaw-vision) (this repo) | Community guide for iPhone vision over Tailscale, for OpenClaw or any agent that can make HTTP requests. | Start at the [README quick start](../README.md#quick-start). Setup notes: [IP Camera Pro](./ip-camera-pro-setup.md), [verified stream pattern](./ip-camera-pro-mobile-working.md). |

Each repository’s README is the install procedure. This page only points at them.

## Suggested order

1. **OpenClaw, from upstream.** Install with the method in the [OpenClaw install docs](https://docs.openclaw.ai/install) (the upstream installer and `npm install -g openclaw` are both documented there). Finish onboarding until `openclaw gateway status` shows the gateway running. Skip [smfworks/openclaw](https://github.com/smfworks/openclaw) unless you already know you want that fork.
2. **Skills.** Add [smfworks-skills](https://github.com/smfworks/smfworks-skills) and install the free skills you want. The pack includes a host `webcam-capture` skill. That skill is a webcam on the agent machine. It is a different path from the iPhone guide in this repo.
3. **Memory.** Add [mnemosyne-openclaw](https://github.com/smfworks/mnemosyne-openclaw) if the agent should keep a local SQLite memory store inside the gateway. You can skip it and still use vision.
4. **Vision.** Come back to this repo’s [README](../README.md). Put Tailscale on the phone and the agent host, run a camera app on the iPhone, and have the agent request a frame over the phone’s `100.` address. Use placeholders in any notes you publish. Change factory camera passwords before the phone is on the tailnet.

You can stop after any step. Vision does not require the skills pack or Mnemosyne. Those two do require a working OpenClaw gateway.

## Optional: Windows gateway tray

[smfworks/openclaw-windows-companion-app](https://github.com/smfworks/openclaw-windows-companion-app) is an SMF Windows system-tray app for starting, stopping, and watching a local OpenClaw gateway. It is not part of the three-piece set above. Use it only if you want that Windows helper. Install or build steps are in its README.
