# PeerPunch Technical Documentation

PeerPunch is a serverless, peer-to-peer communication web application. It provides real-time text chat, voice communication, screen sharing, and file transfers without relying on centralized infrastructure for data routing or storage.

> **Try it live:** [https://peerpunch.netlify.app](https://peerpunch.netlify.app)

This document outlines the technical architecture, security model, and implementation details of the application.

## Architecture and Technology Stack

PeerPunch operates entirely client-side. The application leverages modern browser APIs and decentralized signaling networks to establish and maintain connections between users.

* **WebRTC (Web Real-Time Communication):** The foundational protocol used for all peer-to-peer data and media transport. It handles NAT traversal (via STUN/TURN), packet routing, and network congestion control.
* **Trystero:** A library that abstracts the WebRTC connection process. It handles peer discovery and signaling (the exchange of Session Description Protocol (SDP) payloads and ICE candidates) without requiring a custom backend. By default, it utilizes the decentralized Nostr network as a signaling medium.
* **Vanilla Web Stack:** The interface is built using standard HTML5, CSS3, and ES6 JavaScript.

## The Single-File Architecture

PeerPunch is deployed as a single, standalone `index.html` file. This architecture is achieved through several design decisions:

1. **ES Modules and CDNs:** The application does not require Node.js, Webpack, or any build step. It uses native browser ES Modules (`<script type="module">`) to import dependencies directly from an edge CDN (`https://esm.run/trystero`).
2. **Inline Scoping:** All styling is heavily scoped and contained within a single `<style>` block using CSS variables for theme management. 
3. **Zero Asset Dependencies:** The UI avoids external image assets. It relies entirely on CSS rendering (like the animated background blobs), system fonts, and native Unicode characters for icons, drastically reducing network requests and keeping the file self-contained.
4. **Client-Side State Management:** Application state (peer lists, media streams, file buffers) is held entirely in browser memory using standard JavaScript `Map` and `Set` objects.

## Security Model

Because PeerPunch utilizes WebRTC, all communication—whether text, file buffers, or media streams—is **End-to-End Encrypted (E2EE)** by default. WebRTC strictly enforces the use of DTLS (Datagram Transport Layer Security) for data channels and SRTP (Secure Real-time Transport Protocol) for media streams. Once the connection is established, your data never touches a server.

However, the mechanism used to *establish* that connection (Signaling) passes through public infrastructure (e.g., Nostr relays). PeerPunch handles signaling security in two distinct ways, depending on user input:

### Unauthenticated Rooms (No Password)
If a user joins a room using only a Room ID, Trystero still encrypts the SDP signaling data before sending it over the relay network. This encryption uses a key derived from the App ID and the Room ID. 

* **Vulnerability:** While this protects against passive network eavesdropping, a malicious relay operator who knows the Room ID could theoretically reverse-engineer the key, read the SDP data, and map network topologies. (Note: They still cannot decrypt the actual WebRTC media/data traffic).

### Authenticated Rooms (With Password)
If a user specifies a password, the security model is significantly hardened. 

* **Mechanism:** Trystero uses AES-GCM to encrypt the session descriptions using the user-provided shared secret. 
* **Benefit:** The signaling data becomes completely opaque to everyone, including the relay operators. A peer cannot even begin the WebRTC handshake process without possessing the exact shared secret, preventing unauthorized peers from attempting to connect to the room or parsing connection metadata.

## Technical Implementation Details

* **Data Serialization & Chunking:** File transfers and text messages utilize the `room.makeAction()` abstraction from Trystero, which sits on top of WebRTC DataChannels. This layer automatically handles the chunking of large `ArrayBuffer` objects into optimal packet sizes and provides built-in progress callbacks for the UI.
* **Media Streaming:** Voice and Screen sharing rely on the `navigator.mediaDevices` API (`getUserMedia` and `getDisplayMedia`). When a user toggles a stream, the application adds the `MediaStream` object directly to the existing WebRTC `RTCPeerConnection`.
* **File Buffering:** Files are ingested via the HTML5 File API, converted to `ArrayBuffer`, and sent over the DataChannel. On the receiving end, chunks are reassembled into a `Blob`, which is then processed through `URL.createObjectURL()` to generate local download links or inline media previews without server-side processing.

Mostly all based on [Trystero](https://github.com/dmotz/trystero).
