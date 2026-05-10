# PeerPunch

**PeerPunch is a zero-backend browser app for private, room-based communication.** Open the page, pick a room ID, share the invite link, and peers can chat, talk, share screens, and send files directly over WebRTC.

> Live app: <https://peerpunch.netlify.app>

PeerPunch is intentionally small: the entire application lives in [`index.html`](./index.html). There is no server process, database, account system, build pipeline, or install step required to run it.

## What it does

- **Room-based peer discovery** — users join the same room ID to discover each other.
- **Optional room password** — when provided, the shared secret is passed to Trystero so signaling session descriptions are encrypted with that password.
- **Realtime text chat** — sends messages over Trystero/WebRTC data channels.
- **Typing indicators** — lightweight `typing` action with automatic expiry.
- **Voice chat** — microphone streams are added to the active WebRTC room.
- **Screen sharing** — local and remote screen streams render as in-app video tiles.
- **File transfer** — send one or more files with progress, download buttons, and inline previews for images and videos.
- **Invite links** — the current room ID is stored in the URL hash so links can prefill the join form.
- **Keyboard shortcuts** — `M` toggles microphone, `S` toggles screen sharing, and `F` opens the file picker.

## Quick start

### Use the hosted version

1. Open <https://peerpunch.netlify.app>.
2. Enter a **Room ID** or press the dice button to generate one.
3. Enter a **Display Name**.
4. Optionally enter a **Password**. Everyone in the room must use the exact same password.
5. Click **Join Room**.
6. Share the invite link or room ID with the people you want to reach.

### Run locally

Because the app uses browser ES modules, run it through a local static server instead of opening the file directly:

```bash
python3 -m http.server 8080
```

Then open <http://localhost:8080>.

No dependencies need to be installed locally. The page imports Trystero from `https://esm.run/trystero` and loads Google Fonts at runtime.

## Browser requirements

PeerPunch depends on modern browser APIs:

| Capability | Browser API used | Notes |
| --- | --- | --- |
| Peer connections | WebRTC via Trystero | Requires a network path that WebRTC can traverse. Some restrictive NAT/firewall setups may fail without TURN infrastructure. |
| Microphone | `navigator.mediaDevices.getUserMedia()` | Requires user permission and a secure context, except on localhost. |
| Screen sharing | `navigator.mediaDevices.getDisplayMedia()` | Requires user permission and a browser that supports display capture. |
| File sending | File API + WebRTC data channels | Files are read into memory before sending. Avoid very large files on low-memory devices. |
| Invite copy | Clipboard API | Falls back to telling users to share the room ID manually if clipboard access is denied. |

## How it works

### Single-file app

`index.html` contains the markup, styling, and JavaScript for the whole product:

- The **join overlay** collects the room ID, display name, and optional password.
- The **app shell** contains the top bar, peer list, media controls, screen-share area, chat feed, and composer.
- The **script module** imports `joinRoom` and `selfId` from Trystero, joins a room, registers actions, and wires UI events.

### Connection flow

1. The user enters a room ID and display name.
2. `doJoin()` builds a Trystero config with `appId: 'peerpunch-v4-2026'` and adds `password` only when the password field is non-empty.
3. `joinRoom(config, roomId)` creates or joins the WebRTC room.
4. PeerPunch registers four Trystero actions:
   - `chat` for text messages
   - `meta` for display-name exchange
   - `file` for file payloads and progress callbacks
   - `typing` for typing indicators
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

- **WebRTC media/data channels are encrypted by design.** Chat, files, microphone audio, and screen streams use WebRTC transport once peers connect.
- **Signaling still uses public infrastructure.** Trystero needs a signaling/peer-discovery medium to exchange connection metadata before WebRTC can connect peers.
- **Room IDs are not secrets.** Anyone who knows the room ID can try to join an unprotected room.
- **Use a password for private rooms.** With a password, Trystero encrypts session descriptions using the shared secret; every participant must use the same value.
- **The URL hash contains the room ID.** Invite links make joining easier, but they also expose the room ID to anyone who receives the link.
- **No persistence is implemented.** Downloads and previews are generated locally; PeerPunch does not upload files to storage.

## Limitations

- **Not a guaranteed replacement for hosted conferencing.** Peer-to-peer WebRTC can fail on strict enterprise networks, VPNs, or NAT configurations.
- **No TURN server is configured by this app.** If direct peer connectivity fails, there is no app-owned relay fallback.
- **No message history.** Late joiners only see messages sent after they join.
- **No identity verification.** Display names are self-reported and can be duplicated or impersonated.
- **No moderation or access control beyond the shared room ID/password.** Use high-entropy room names and a password for sensitive sessions.
- **Large files are memory-heavy.** The current implementation reads each file into an `ArrayBuffer` before sending.
- **External runtime dependencies are loaded from CDNs.** The app imports Trystero through `esm.run` and fonts through Google Fonts.

## Project structure

```text
.
├── README.md     # Project documentation
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
