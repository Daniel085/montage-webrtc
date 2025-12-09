# Montage WebRTC

> A modern WebRTC library built for the Montage framework, providing peer-to-peer communication, data channels, and media streaming capabilities.

[![GitHub](https://img.shields.io/badge/github-Daniel085%2Fmontage--webrtc-blue)](https://github.com/Daniel085/montage-webrtc)
[![Montage](https://img.shields.io/badge/montage-%5E0.15.2-orange)](https://github.com/montagejs/montage)
[![WebRTC](https://img.shields.io/badge/WebRTC-Modern%20APIs-green)](https://webrtc.org/)

---

## Table of Contents

- [Features](#features)
- [Recent Updates](#recent-updates)
- [Browser Support](#browser-support)
- [Installation](#installation)
- [Quick Start](#quick-start)
  - [Basic WebRTC Connection](#basic-webrtc-connection)
  - [Using Presence Client](#using-presence-client)
  - [Attaching Media Streams](#attaching-media-streams)
- [Architecture](#architecture)
- [API Reference](#api-reference)
- [Connection Roles](#connection-roles)
- [Common Use Cases](#common-use-cases)
- [Development](#development)
- [Contributing](#contributing)
- [License](#license)

---

## Features

### Core Capabilities

- **Modern WebRTC APIs** - Fully updated to use Promise-based WebRTC APIs and track-based media handling
- **Multiple Connection Roles** - Supports signaling, data, and media peer connections with separate `RTCPeerConnection` instances
- **Flexible Topology Management** - Client and server-side topology services for managing peer networks and mesh patterns
- **Dual Presence System** - WebSocket-based and RTC-based presence clients for different connectivity scenarios
- **Reliable Data Channels** - Fault-tolerant data channel communication with health monitoring
- **Modern Media Streaming** - Audio/video track management with modern `addTrack`/`removeTrack` APIs
- **Cross-Browser Support** - Works across all modern browsers (Chrome, Firefox, Safari, Edge)

### Advanced Features

- **Health Monitoring** - Automatic ping/pong system with 5-second timeout and 5-miss threshold
- **Dynamic Mesh Networks** - Configurable mesh patterns based on CPU strength and network topology
- **P2P Mode Switching** - Ability to switch from WebSocket signaling to direct RTC signaling
- **Smart Path Computation** - Multi-hop network path optimization for efficient peer communication
- **ICE Candidate Handling** - Robust ICE candidate management with proper error handling
- **Graceful Degradation** - Fault-tolerant design with automatic reconnection and cleanup

---

## Recent Updates

This library has been **fully modernized** as of December 2025:

- Removed vendor prefixes (`webkitRTCPeerConnection`)
- Converted callback-based APIs to async/await
- Migrated from deprecated stream-based APIs (`addStream`/`removeStream`) to track-based APIs (`addTrack`/`removeTrack`)
- Replaced `onaddstream` with modern `ontrack` event handling
- Enhanced error handling with try/catch blocks
- Improved connection stability and fault tolerance
- Added CPU-based mesh pattern optimization

See [MODERNIZATION_AUDIT.md](./MODERNIZATION_AUDIT.md) for complete details.

**Git Branch**: `modern-webrtc` (main development branch)

---

## Browser Support

| Browser | Minimum Version | Status |
|---------|----------------|--------|
| Chrome/Edge | 72+ | Fully Supported |
| Firefox | 63+ | Fully Supported |
| Safari | 12.1+ | Fully Supported |
| Opera | 60+ | Fully Supported |

---

## Installation

### From Source

```bash
# Clone the repository
git clone https://github.com/Daniel085/montage-webrtc.git
cd montage-webrtc

# Install dependencies
npm install
```

### As a Dependency

Add to your `package.json`:

```json
{
  "dependencies": {
    "montage-webrtc": "github:Daniel085/montage-webrtc#modern-webrtc"
  }
}
```

Then run:

```bash
npm install
```

---

## Quick Start

### Basic WebRTC Connection

```javascript
var RTCService = require("montage-webrtc/client").RTCService;

// Initialize RTC service with your client ID and STUN servers
var rtcService = new RTCService().init(clientId, [
    { urls: "stun:stun.l.google.com:19302" }
]);

// Connect to a room
rtcService.connect(roomId).then(function() {
    console.log("Connected to room:", roomId);
});

// Send data to peers
rtcService.send({
    type: 'message',
    data: 'Hello, peer!'
});

// Listen for messages from peers
rtcService.addEventListener('message', function(event) {
    console.log("Received:", event.detail);
});

// Listen for connection events
rtcService.addEventListener('connectionClose', function(event) {
    console.log("Peer disconnected:", event.detail);
});
```

### Using Presence Client

```javascript
var WsPresenceClient = require("montage-webrtc/wsPresenceClient").WsPresenceClient;

// Initialize presence client with WebSocket server URL
var presenceClient = new WsPresenceClient().init("ws://your-server.com/presence");

// Connect to presence server
presenceClient.connect().then(function() {
    console.log("Connected to presence server");

    // Create or join a room
    return presenceClient.createRoom("My Room");
}).then(function(room) {
    console.log("Room created:", room);
});

// Join an existing room
presenceClient.joinRoom(roomId).then(function() {
    console.log("Joined room:", roomId);
});
```

### Attaching Media Streams

```javascript
// Get user media (camera and microphone)
navigator.mediaDevices.getUserMedia({
    video: true,
    audio: true
})
.then(function(stream) {
    // Show local video
    localVideoElement.srcObject = stream;

    // Attach stream to peer connection
    return rtcService.attachStream(stream);
})
.then(function() {
    console.log("Local stream attached and shared with peers");
});

// Listen for remote streams from peers
rtcService.addEventListener('addstream', function(event) {
    var remoteStream = event.detail.stream;
    var peerId = event.detail.peerId;

    console.log("Received stream from peer:", peerId);
    remoteVideoElement.srcObject = remoteStream;
});

// Listen for stream removal
rtcService.addEventListener('removestream', function(event) {
    console.log("Stream removed");
    remoteVideoElement.srcObject = null;
});

// Detach stream when done
rtcService.detachStream();
```

---

## Architecture

### System Overview

```
┌─────────────────────────────────────────────────────┐
│           APPLICATION (e.g., HiveClass)             │
└──────────────────────┬──────────────────────────────┘
                       │
        ┌──────────────┴──────────────┐
        │                             │
   ┌────▼───────────┐      ┌─────────▼─────────┐
   │ WsPresenceClient│      │RtcPresenceClient  │
   │   (WebSocket)   │      │   (RTC-based)     │
   └────┬────────────┘      └────────┬──────────┘
        │                            │
        └──────────────┬─────────────┘
                       │
            ┌──────────▼──────────┐
            │   RTCService        │
            │ (client.js)         │
            │                     │
            │ 3 Peer Connections: │
            │ • Signaling         │
            │ • Data              │
            │ • Media             │
            └──────┬───────┬──────┘
                   │       │
        ┌──────────┘       └──────────┐
        │                             │
   ┌────▼──────────────┐      ┌──────▼──────────┐
   │ WebRTC Connection │      │  WebRTC         │
   │ (Peer 1)          │      │  Connection     │
   │                   │      │  (Peer 2)       │
   └────────┬──────────┘      └──────┬──────────┘
            │                        │
        ┌───┴────────────┬───────────┴──┐
        │                │              │
   ┌────▼───┐      ┌─────▼───┐    ┌────▼────┐
   │  Audio │      │  Video  │    │   Data   │
   │ Tracks │      │ Tracks  │    │ Channels │
   └────────┘      └─────────┘    └──────────┘
```

### Core Components

#### RTCService (`client.js`)

The main WebRTC service managing all peer connections.

**Responsibilities:**
- Connection management for signaling, data, and media
- ICE candidate handling with STUN/TURN servers
- Modern track-based media management
- Reliable data channel messaging
- Health monitoring with ping/pong
- Event dispatching for application layer

**Key Features:**
- Separate `RTCPeerConnection` instances for each role
- Automatic reconnection on failure
- Graceful connection closure
- Cross-browser compatibility

#### ClientTopologyService (`client-topology-service.js`)

Simple peer list management for client-side applications.

**Responsibilities:**
- Add/remove peers from topology
- Get sorted list of connected peers
- Track peer connection state

**Used by:** HiveClass and other client applications

#### ServerTopologyService (`server-topology-service.js`)

Advanced network topology management for server-side orchestration.

**Responsibilities:**
- Node connection tracking
- Path computation for mesh networks
- Dynamic topology updates
- Configurable mesh patterns based on CPU strength
- Multi-hop path optimization

**Features:**
- Smart path computation for efficient routing
- Topology change event dispatching
- Node instance management
- Connection graph maintenance

#### RtcPresenceClient (`rtcPresenceClient.js`)

Peer-to-peer presence using RTC data channels for distributed architectures.

**Responsibilities:**
- Direct peer-to-peer communication
- Stream event forwarding
- Topology-aware peer management
- Presence state synchronization

**Use Case:** After initial WebSocket connection, can switch to P2P mode

#### WsPresenceClient (`wsPresenceClient.js`)

WebSocket-based presence and signaling for centralized architectures.

**Responsibilities:**
- Room creation and management
- WebSocket signaling server communication
- RTC service integration and lifecycle
- Initial connection establishment

**Use Case:** Primary presence system for most applications

---

## API Reference

### RTCService

#### Methods

##### `init(id, stunServers)`

Initialize the RTC service with client configuration.

**Parameters:**
- `id` (String): Unique client identifier
- `stunServers` (Array, optional): Array of STUN/TURN server configuration objects
  ```javascript
  [{ urls: "stun:stun.l.google.com:19302" }]
  ```

**Returns:** `this` (for method chaining)

**Example:**
```javascript
var rtcService = new RTCService().init("client-123", [
    { urls: "stun:stun.l.google.com:19302" }
]);
```

---

##### `connect(roomId)`

Connect to a room and establish peer connections.

**Parameters:**
- `roomId` (String): Room identifier to join

**Returns:** `Promise<void>`

**Example:**
```javascript
rtcService.connect("room-456").then(function() {
    console.log("Connected to room");
});
```

---

##### `connectToPeer(peerId)`

Establish a direct connection to a specific peer.

**Parameters:**
- `peerId` (String): Target peer identifier

**Returns:** `Promise<void>`

**Example:**
```javascript
rtcService.connectToPeer("peer-789").then(function() {
    console.log("Connected to peer");
});
```

---

##### `send(message)`

Send a message through the data channel to all connected peers.

**Parameters:**
- `message` (Object): Message object (will be JSON stringified)

**Example:**
```javascript
rtcService.send({
    type: 'chat',
    text: 'Hello, world!',
    timestamp: Date.now()
});
```

---

##### `attachStream(stream)`

Attach a media stream to the media peer connection and share with peers.

**Parameters:**
- `stream` (MediaStream): MediaStream object from `getUserMedia`

**Returns:** `Promise<void>`

**Example:**
```javascript
navigator.mediaDevices.getUserMedia({ video: true, audio: true })
    .then(function(stream) {
        return rtcService.attachStream(stream);
    });
```

---

##### `detachStream()`

Remove all tracks from the media peer connection.

**Example:**
```javascript
rtcService.detachStream();
```

---

##### `quit(sendMessage)`

Close all peer connections and clean up resources.

**Parameters:**
- `sendMessage` (Boolean): Whether to notify peers before disconnecting

**Returns:** `Promise<void>`

**Example:**
```javascript
rtcService.quit(true).then(function() {
    console.log("Disconnected from all peers");
});
```

---

#### Events

The RTCService dispatches the following events:

##### `message`

Fired when a message is received through the data channel.

**Event Detail:** Message object (parsed from JSON)

**Example:**
```javascript
rtcService.addEventListener('message', function(event) {
    console.log("Received message:", event.detail);
});
```

---

##### `addstream`

Fired when a remote stream is added from a peer.

**Event Detail:**
- `stream` (MediaStream): The remote media stream
- `peerId` (String): ID of the peer who sent the stream

**Example:**
```javascript
rtcService.addEventListener('addstream', function(event) {
    videoElement.srcObject = event.detail.stream;
    console.log("Stream from:", event.detail.peerId);
});
```

---

##### `removestream`

Fired when a remote stream is removed.

**Example:**
```javascript
rtcService.addEventListener('removestream', function(event) {
    videoElement.srcObject = null;
});
```

---

##### `signalingMessage`

Fired when a signaling message needs to be sent to a peer.

**Event Detail:** Signaling message object

**Example:**
```javascript
rtcService.addEventListener('signalingMessage', function(event) {
    // Forward to signaling server
    webSocket.send(JSON.stringify(event.detail));
});
```

---

##### `connectionClose`

Fired when a peer connection closes.

**Event Detail:** Peer identifier

**Example:**
```javascript
rtcService.addEventListener('connectionClose', function(event) {
    console.log("Peer disconnected:", event.detail);
});
```

---

##### `sendError`

Fired when sending a message fails.

**Event Detail:** Error information

**Example:**
```javascript
rtcService.addEventListener('sendError', function(event) {
    console.error("Send failed:", event.detail);
});
```

---

### ClientTopologyService

Simple topology management for tracking connected peers.

#### Methods

##### `addPeer(peerId)`

Add a peer to the topology.

**Parameters:**
- `peerId` (String): Peer identifier

---

##### `removePeer(peerId)`

Remove a peer from the topology.

**Parameters:**
- `peerId` (String): Peer identifier

---

##### `removePeers(peerIds)`

Remove multiple peers from the topology.

**Parameters:**
- `peerIds` (Array<String>): Array of peer identifiers

---

##### `getPeers()`

Get the current list of peers in sorted order.

**Returns:** `Array<String>` - Array of peer identifiers

---

### ServerTopologyService

Advanced topology management with path computation.

#### Methods

##### `updateNodeConnections(nodeId, connections)`

Update the connections for a node in the topology.

**Parameters:**
- `nodeId` (String): Node identifier
- `connections` (Array<String>): Array of connected node IDs

---

##### `removeNode(nodeId)`

Remove a node and all its instances from the topology.

**Parameters:**
- `nodeId` (String): Node identifier

**Returns:** `Array<String>` - Array of removed node IDs

---

##### `hasNode(nodeId)`

Check if a node exists in the topology.

**Parameters:**
- `nodeId` (String): Node identifier

**Returns:** `Boolean`

---

##### `getPaths()`

Get optimized paths through the network for efficient routing.

**Returns:** `Array<Array<String>>` - Array of paths (each path is an array of node IDs)

---

##### `addNode(id)`

Add a node to the topology.

**Parameters:**
- `id` (String): Node identifier

---

##### `getTopology()`

Get the current topology structure.

**Returns:** `Array` - List of nodes with their connections

---

#### Events

##### `topologyChanged`

Fired when the network topology changes.

**Example:**
```javascript
topologyService.addEventListener('topologyChanged', function(event) {
    console.log("Topology updated");
    updateNetworkVisualization();
});
```

---

### WsPresenceClient

WebSocket-based presence client for centralized signaling.

#### Methods

##### `init(presenceEndpointUrl, isServer)`

Initialize the presence client.

**Parameters:**
- `presenceEndpointUrl` (String): WebSocket server URL (e.g., `ws://localhost:8080`)
- `isServer` (Boolean): Whether this is a server instance (default: `false`)

**Returns:** `this` (for method chaining)

---

##### `connect()`

Connect to the presence server via WebSocket.

**Returns:** `Promise<void>`

---

##### `createRoom(name, id)`

Create a new room on the presence server.

**Parameters:**
- `name` (String): Human-readable room name
- `id` (String, optional): Room ID (auto-generated if not provided)

**Returns:** `Promise<Object>` - Room data with `id` and `name`

---

##### `joinRoom(roomId)`

Join an existing room.

**Parameters:**
- `roomId` (String): Room identifier

**Returns:** `Promise<void>`

---

## Connection Roles

Montage WebRTC uses **three distinct connection roles** for better isolation and control:

### 1. Signaling Connection (`ROLE_SIGNALING`)

**Purpose:** Initial connection establishment and WebRTC signaling

**Responsibilities:**
- Exchange SDP offers and answers
- Exchange ICE candidates
- Initial peer handshake

**Lifecycle:** Created first, can be replaced by P2P signaling later

---

### 2. Data Connection (`ROLE_DATA`)

**Purpose:** Reliable application messaging between peers

**Responsibilities:**
- Send/receive application data
- Health monitoring (ping/pong every 5 seconds)
- Connection state tracking
- Can handle signaling after initial connection

**Features:**
- 5-second ping timeout
- 5-miss threshold before disconnect
- Automatic reconnection
- JSON message serialization

---

### 3. Media Connection (`ROLE_MEDIA`)

**Purpose:** Audio/video stream transmission

**Responsibilities:**
- Share media tracks with peers
- Receive media tracks from peers
- Handle track add/remove events

**Features:**
- Modern track-based API
- Separate from data channel for better QoS
- Independent lifecycle management

---

Each role uses a **separate `RTCPeerConnection` instance** to provide:
- Better isolation and debugging
- Independent connection management
- Optimized configuration per role
- Fault tolerance (one connection can fail without affecting others)

---

## Common Use Cases

### Peer-to-Peer Video Chat

Build a simple video chat application with WebRTC.

```javascript
var RTCService = require("montage-webrtc/client").RTCService;

var rtcService = new RTCService().init(userId, [
    { urls: "stun:stun.l.google.com:19302" }
]);

// Get local video stream
navigator.mediaDevices.getUserMedia({
    video: true,
    audio: true
})
.then(function(stream) {
    // Display local video
    localVideo.srcObject = stream;

    // Attach to WebRTC
    return rtcService.attachStream(stream);
})
.then(function() {
    // Connect to room
    return rtcService.connect(roomId);
})
.then(function() {
    console.log("Video chat ready");
});

// Handle remote video
rtcService.addEventListener('addstream', function(event) {
    remoteVideo.srcObject = event.detail.stream;
    console.log("Connected to:", event.detail.peerId);
});

// Handle disconnect
rtcService.addEventListener('removestream', function(event) {
    remoteVideo.srcObject = null;
});
```

---

### Collaborative Application with Data Channels

Build a real-time collaborative application (whiteboard, document editor, etc.).

```javascript
var WsPresenceClient = require("montage-webrtc/wsPresenceClient").WsPresenceClient;

var client = new WsPresenceClient().init("ws://localhost:8080");

// Connect and join room
client.connect()
    .then(function() {
        return client.joinRoom(roomId);
    })
    .then(function() {
        console.log("Connected to collaborative session");
    });

// Send collaborative data
function sendUpdate(type, data) {
    client.rtcService.send({
        type: type,
        data: data,
        timestamp: Date.now()
    });
}

// Examples
sendUpdate('cursor', { x: 100, y: 200 });
sendUpdate('draw', { tool: 'pen', color: 'red', points: [...] });
sendUpdate('text', { content: 'Hello', position: { x: 50, y: 50 } });

// Receive collaborative updates
client.rtcService.addEventListener('message', function(event) {
    var message = event.detail;

    switch(message.type) {
        case 'cursor':
            updateRemoteCursor(message.data);
            break;
        case 'draw':
            drawRemotePath(message.data);
            break;
        case 'text':
            addRemoteText(message.data);
            break;
    }
});
```

---

### Dynamic Mesh Network

Manage a dynamic peer network with automatic topology updates.

```javascript
var ClientTopologyService = require("montage-webrtc/client-topology-service").ClientTopologyService;

var topology = new ClientTopologyService();

// Add peers as they connect
rtcService.addEventListener('peerConnected', function(event) {
    var peerId = event.detail;
    topology.addPeer(peerId);
    console.log("Peer added:", peerId);
});

// Broadcast to all peers
function broadcast(message) {
    var peers = topology.getPeers();
    peers.forEach(function(peerId) {
        rtcService.sendToPeer(peerId, message);
    });
}

// Handle disconnections
rtcService.addEventListener('connectionClose', function(event) {
    var peerId = event.detail;
    topology.removePeer(peerId);
    console.log("Peer removed:", peerId);
});

// Get network stats
function getNetworkStats() {
    var peers = topology.getPeers();
    return {
        peerCount: peers.length,
        peers: peers
    };
}
```

---

### Server-Side Mesh Topology

Use advanced topology management for server-orchestrated networks.

```javascript
var ServerTopologyService = require("montage-webrtc/server-topology-service").ServerTopologyService;

var topologyService = new ServerTopologyService();

// Add nodes
topologyService.addNode("node-1");
topologyService.addNode("node-2");
topologyService.addNode("node-3");

// Update connections
topologyService.updateNodeConnections("node-1", ["node-2"]);
topologyService.updateNodeConnections("node-2", ["node-1", "node-3"]);
topologyService.updateNodeConnections("node-3", ["node-2"]);

// Get optimal paths
var paths = topologyService.getPaths();
console.log("Optimal paths:", paths);
// Output: [["node-1", "node-2", "node-3"]]

// Listen for topology changes
topologyService.addEventListener('topologyChanged', function(event) {
    console.log("Network topology changed");
    recalculateRouting();
});

// Remove disconnected node
topologyService.removeNode("node-2");
// Automatically updates paths
```

---

## Development

### Project Structure

```
montage-webrtc/
├── client.js                      # Main RTCService implementation (650 lines)
├── client-topology-service.js     # Client-side peer management (48 lines)
├── server-topology-service.js     # Server-side topology management (180 lines)
├── rtcPresenceClient.js           # RTC-based presence client (230 lines)
├── wsPresenceClient.js            # WebSocket presence client (460 lines)
├── package.json                   # Package configuration
├── README.md                      # This file
└── MODERNIZATION_AUDIT.md         # Modernization documentation
```

### Running Tests

```bash
npm test
```

### Building

This library is written in CommonJS and designed to work with Montage's build system.

```bash
# In your Montage application
npm install
npm run build
```

### Debugging

Enable WebRTC debug logging in Chrome:

1. Open `chrome://webrtc-internals/`
2. Monitor connection states, ICE candidates, and stats
3. Check console for RTCService events

### Code Style

- Use Montage's `specialize` pattern for class definitions
- Follow existing naming conventions
- Add JSDoc comments for public APIs
- Use promises for async operations
- Emit events for state changes

---

## Troubleshooting

### Connection Issues

**Problem:** Peers can't connect

**Solutions:**
- Check STUN/TURN server configuration
- Verify firewall rules allow WebRTC traffic
- Check WebSocket connection to signaling server
- Monitor `chrome://webrtc-internals/` for ICE failures

---

### Media Not Flowing

**Problem:** Video/audio not working

**Solutions:**
- Verify `getUserMedia` permissions granted
- Check that media tracks are attached before connecting
- Ensure peer connection is established (check `addstream` event)
- Verify media elements have `srcObject` set correctly

---

### Messages Not Received

**Problem:** Data channel messages not arriving

**Solutions:**
- Check data channel is open (`readyState === 'open'`)
- Verify JSON serialization is working
- Check browser console for send errors
- Monitor ping/pong health checks

---

### High CPU Usage

**Problem:** Performance issues with many peers

**Solutions:**
- Adjust mesh pattern based on CPU (use `ServerTopologyService`)
- Reduce video resolution/bitrate
- Limit number of simultaneous connections
- Use server-side forwarding for large groups

---

## Contributing

Contributions are welcome! Here's how to get started:

### Getting Started

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Make your changes
4. Test thoroughly
5. Commit with clear messages (`git commit -m 'Add amazing feature'`)
6. Push to your branch (`git push origin feature/amazing-feature`)
7. Open a Pull Request

### Guidelines

- **Code Style:** Follow existing code patterns and Montage conventions
- **Testing:** Add tests for new features and bug fixes
- **Documentation:** Update README and API docs for changes
- **Commits:** Use clear, descriptive commit messages
- **Modernization:** Keep using modern WebRTC APIs (no deprecated methods)

### Pull Request Process

1. Update documentation for any API changes
2. Add tests for new functionality
3. Ensure all tests pass
4. Update MODERNIZATION_AUDIT.md if applicable
5. Request review from maintainers

### Reporting Issues

When reporting issues, please include:
- Browser and version
- Operating system
- Steps to reproduce
- Expected vs actual behavior
- Console logs and errors
- WebRTC internals dump (if connection issue)

---

## License

This project is part of the Montage ecosystem.

**Original Project:** [montagestudio/montage-webrtc](https://github.com/montagestudio/montage-webrtc)
**Fork:** [Daniel085/montage-webrtc](https://github.com/Daniel085/montage-webrtc)

---

## Related Projects

- [Montage Framework](https://github.com/montagejs/montage) - Modern JavaScript framework
- [HiveClass](https://github.com/HiveClass) - Collaborative classroom platform using this library
- [WebRTC.org](https://webrtc.org/) - Official WebRTC documentation
