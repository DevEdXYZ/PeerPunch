# Security model and review notes

## Intended guarantees

- PeerPunch has no application backend or message database.
- After a WebRTC peer connection is established, chat payloads, reactions, poll events, files, voice and screen media move through the peer connection. Trystero documents that peer communications are end-to-end encrypted by WebRTC.
- The app requires a shared room password and passes it to Trystero. Trystero documents the password as the key material used to encrypt SDP/session descriptions with AES-GCM while signaling crosses relays. Share the password out-of-band.
- Generated room IDs use cryptographic randomness. The password is never placed in the invite URL and TURN credentials are not included in invite links.
- Remote text, names, poll labels and filenames are rendered as text nodes rather than inserted as HTML. Incoming text and file sizes are bounded, and rendered chat history is capped.

## Important limits

E2E transport protects data in transit between the participating browser endpoints. It does not protect content from the endpoints themselves, a malicious room participant, a browser/device compromise, a compromised copy of the app JavaScript, or content recipients who record or save what they receive.

The display name is not an authenticated identity. Public signaling providers can observe discovery traffic, room topics, timing and some connection metadata. WebRTC may expose network information to peers as part of connection establishment. A configured TURN service can relay encrypted packets and still see connection metadata.

The password is not an account password and PeerPunch does not persist it. Choose a long, unique value; prefer the built-in cryptographically random generator (28 characters), and deliver it through a separate trusted channel. A weak shared secret can undermine the benefit of password-protected signaling.

Pinned CDN imports are still a supply-chain dependency. For a stronger deployment boundary, self-host audited copies of the exact Trystero packages and review updates before replacing them. The static app does not currently provide identity fingerprints, key transparency, abuse moderation, or a formal third-party cryptographic audit.

## Security checklist for maintainers

- Keep Trystero versions pinned and review release notes/source before upgrades.
- Never put room passwords, TURN credentials, or message contents in URLs, logs, analytics, or issue reports.
- Keep untrusted values on text-safe DOM APIs. Do not introduce remote-content `innerHTML`.
- Test mismatched passwords, peer leave/rejoin, malformed chat/poll events, oversized files, revoked media permissions, and browsers that reject autoplay.
- Serve over HTTPS (or localhost in development); microphone, screen capture and secure random APIs require a secure context in typical browsers.
- Treat claims about confidentiality as transport/security-model claims, not as proof that the full deployment has been independently audited.
