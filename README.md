# PeerPunch

**PeerPunch is a zero-backend browser app for private, room-based communication.** Open the page, pick a room ID, share the invite link, and peers can chat, talk, share screens, and send files directly over WebRTC.

> Live app: <https://peerpunch.netlify.app>

PeerPunch is intentionally small: the entire application lives in [`index.html`](./index.html). There is no server process, database, account system, build pipeline, or install step required to run it.

## What it does

- **Room-based peer discovery** — users join the same room ID to discover each other.
- **Required room password** — the shared secret is passed to Trystero to encrypt signaling session descriptions; share it out-of-band.
- **Realtime text chat** — sends messages over Trystero/WebRTC data channels.
- **Typing indicators** — lightweight `typing` action with automatic expiry.
- **Voice chat** — microphone streams are added to the active WebRTC room.
- **Screen sharing** — local and remote screen streams render as in-app video tiles.
- **File transfer** — send one or more files with progress, download buttons, and inline previews for images and videos.
- **Invite links** — the current room ID is stored in the URL hash so links can prefill the join form.
- **Keyboard shortcuts** — `M` toggles microphone, `S` toggles screen sharing, and `F` opens the file picker.
- **Alternative signaling** — choose BitTorrent trackers or Nostr relays; invite links remember the selected network.
- **TURN support** — optionally configure your own TURN URL and credentials for networks that block direct WebRTC paths.
- **Local image previews** — image/video previews are shown to the sender as well as the receiver; images can be enlarged.
- **Chat tools** — search messages, copy text, use the emoji picker, and react to individual messages.
- **Live polls** — create room polls with 2–4 options and change your vote; poll state is ephemeral and room-local.
- **Personal themes and party mode** — cycle through five accent palettes and trigger a local confetti celebration.
- **Safer setup** — required strong shared room password, one-click cryptographically random password generation, and higher-entropy room IDs.
- **One-click signaling retry** — when no peers are present, switch between the two discovery networks and copy a fresh invite.
- **Connection diagnostics** — the status dot distinguishes an active peer connection, available signaling with no peers yet, and relays that are reconnecting or unavailable.

## Quick start

### Use the hosted version

1. Open <https://peerpunch.netlify.app>.
2. Enter a **Room ID** or press the dice button to generate one.
3. Enter a **Display Name**.
4. Click **Generate strong password** (recommended) or enter a unique password of at least 16 characters. Avoid leading/trailing spaces and obvious patterns.
5. Share the password separately with intended participants. It is deliberately **not** included in invite links.
6. Click **Create secure connection**.
7. Share the invite link or room ID with the people you want to reach.

### Run locally

Because the app uses browser ES modules, run it through a local static server instead of opening the file directly:

```bash
python3 -m http.server 8080
```

Then open <http://localhost:8080>.

No dependencies need to be installed locally. The page imports pinned Trystero `0.26.0` from `https://esm.run/trystero@0.26.0` and the BitTorrent strategy from `https://esm.run/@trystero-p2p/torrent@0.26.0`, plus Google Fonts at runtime. These pinned CDN imports remain a supply-chain dependency; self-host audited copies for a stronger boundary.

## Browser requirements

PeerPunch depends on modern browser APIs:

| Capability | Browser API used | Notes |
| --- | --- | --- |
| Peer connections | WebRTC via Trystero | Requires a network path that WebRTC can traverse. Some restrictive NAT/firewall setups may fail without TURN infrastructure. |
| Microphone | `navigator.mediaDevices.getUserMedia()` | Requires user permission and a secure context, except on localhost. |
| Screen sharing | `navigator.mediaDevices.getDisplayMedia()` | Requires user permission and a browser that supports display capture. |
| File sending | File API + WebRTC data channels | Files are read into memory before sending. Transfers are limited to 100 MiB per file to reduce memory pressure. |
| Invite copy | Clipboard API | Falls back to telling users to share the room ID manually if clipboard access is denied. |

## How it works

### Single-file app

`index.html` contains the markup, styling, and JavaScript for the whole product:

- The **join overlay** collects the room ID, display name, and required shared password. The password is never added to an invite URL or saved by PeerPunch.
- The **app shell** contains the top bar, peer list, media controls, screen-share area, chat feed, and composer.
- The **script module** imports `joinRoom` and `selfId` from Trystero, joins a room, registers actions, and wires UI events.

### Connection flow

1. The user enters a room ID and display name.
2. `doJoin()` builds a Trystero config with `appId: 'peerpunch-v5-2026'`, uses the selected signaling network, and always sets the required shared password so signaling session descriptions are encrypted with the password-derived key.
3. `joinRoom(config, roomId, callbacks)` creates or joins the WebRTC room and reports peer-connection errors.
4. PeerPunch registers five Trystero message actions:
   - `chat` for text messages and message IDs
   - `meta` for display-name exchange
   - `file` for file payloads and progress callbacks
   - `typing` for typing indicators
   - `fun` for reactions and live polls
5. Peer events update the member list, announce joins/leaves, and attach incoming media streams.
6. Once connected, application payloads move over WebRTC between peers rather than through a PeerPunch backend.

### Data and media paths

- **Chat:** messages are simple objects like `{ text, n }`, then rendered with `textContent` to avoid HTML injection.
- **Display names:** peers broadcast a compact metadata payload `{ n: myName }` after joining and when a new peer arrives.
- **Files:** files are read as `ArrayBuffer`s, sent through the `file` action, reconstructed as `Blob`s on receipt, and offered through local object URLs.
- **Images/videos:** received image and video blobs are previewed inline with generated object URLs.
- **Voice:** microphone audio comes from `getUserMedia()` and is added with `room.addStream()`.
- **Screen share:** display capture comes from `getDisplayMedia()` and is rendered in local/remote video tiles.

## Privacy and security model

PeerPunch has no application backend and does not store messages, files, names, room IDs, or media. State exists in browser memory and is cleared when the page reloads or the room is left.

Important details:

- **Chat, reactions, polls, files, voice, and screen streams travel over WebRTC peer connections.** WebRTC encrypts data and media end-to-end between connected browsers; files and chat are not sent through PeerPunch servers.
- **The required shared password protects signaling setup.** Trystero uses it to encrypt session descriptions with AES-GCM while those descriptions traverse public signaling infrastructure. All participants must use the same secret, and it must be shared out-of-band.
- **A password is not identity verification.** A participant who knows the secret can join; display names remain self-reported. Protect the secret and only share it with the intended people.
- **Room IDs are locators, not authentication.** New IDs include a cryptographically generated random suffix to reduce accidental collisions and guessing, but the password remains essential.
- **The URL hash contains the room ID only.** Invite links do not contain the password or TURN credentials. Anyone who gets an invite can learn the room ID, so share links thoughtfully.
- **No persistence is implemented.** Messages, polls, reactions, files, and media are held in browser memory and cleared when leaving or reloading. Received files are not uploaded to storage.
- **Trust the endpoints and shipped JavaScript.** E2E transport cannot protect content from a compromised browser/device, malicious participant, malicious modified app build, or someone who copies or records received content. Runtime dependencies are loaded from pinned CDN URLs; self-host them for a stronger supply-chain boundary.
- **Network metadata remains visible.** Signaling providers can observe room topics/traffic timing and connection-related metadata; peers may learn network information required by WebRTC. TURN servers relay encrypted transport packets when needed.

## Limitations

- **Not a guaranteed replacement for hosted conferencing.** Peer-to-peer WebRTC can fail on strict enterprise networks, VPNs, or NAT configurations.
- **TURN is optional and user-configured.** PeerPunch lets you enter your provider's TURN URL and credentials; these are kept out of invite links. Without TURN, restrictive NAT/firewall setups can still block direct connections.
- **No message history.** Late joiners only see messages sent after they join.
- **No identity verification.** Display names are self-reported and can be duplicated or impersonated.
- **No moderation or role-based access control.** Anyone with both the room ID and shared password can join; peers can save files or capture audio/video they receive.
- **Password strength matters.** The generator uses browser cryptographic randomness. Weak hand-picked passwords can reduce protection for signaling descriptions.
- **No independent application-layer cryptographic review or browser-to-browser integration test is provided by this single-file project.** Review the Trystero version and test with the browser/network combinations you plan to use.
- **Large files are memory-heavy.** The current implementation reads each file into an `ArrayBuffer` before sending, and currently limits each transfer to 100 MiB.
- **External runtime dependencies are loaded from CDNs.** The app imports Trystero through `esm.run` and fonts through Google Fonts.

## Project structure

```text
.
├── README.md     # Project documentation
├── SECURITY.md   # Security model and maintainer checklist
└── index.html    # Entire PeerPunch application
```

## Development notes

- There is no package manager configuration because there is no build step.
- Keep user-supplied content rendered with `textContent` or text nodes, not `innerHTML`.
- If you add dependencies, document whether they are bundled, CDN-loaded, or installed through a build process.
- If you add a backend, update this README's privacy/security claims immediately.

## Deploying

Any static host can serve PeerPunch:

- Netlify
- Vercel static output
- GitHub Pages
- Cloudflare Pages
- S3/R2-style object hosting
- Any basic HTTP server

Deploy `index.html` as the site root. HTTPS is strongly recommended and is required by most browsers for microphone, screen capture, and clipboard APIs outside `localhost`.

## Credits

PeerPunch is built around [Trystero](https://github.com/dmotz/trystero), which abstracts WebRTC room discovery, actions, streams, and peer lifecycle events.
