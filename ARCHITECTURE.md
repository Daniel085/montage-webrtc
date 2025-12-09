# Montage WebRTC Architecture

> Deep technical documentation of the Montage WebRTC library architecture, design patterns, and implementation details.

**Version**: 0.1.0
**Branch**: modern-webrtc
**Last Updated**: December 2025

---

## Table of Contents

1. [Architectural Overview](#architectural-overview)
2. [Design Decisions & Rationale](#design-decisions--rationale)
3. [Connection Architecture](#connection-architecture)
4. [State Management](#state-management)
5. [Message Flow & Protocols](#message-flow--protocols)
6. [Topology Algorithms](#topology-algorithms)
7. [Health Monitoring System](#health-monitoring-system)
8. [P2P Mode Switching](#p2p-mode-switching)
9. [Media Management](#media-management)
10. [Event System](#event-system)
11. [Error Handling & Resilience](#error-handling--resilience)
12. [Performance Characteristics](#performance-characteristics)
13. [Security Architecture](#security-architecture)
14. [Extension Points](#extension-points)

---

## Architectural Overview

### High-Level Architecture

Montage WebRTC implements a **multi-connection, role-based WebRTC architecture** with support for both centralized (WebSocket-based) and distributed (P2P) signaling.

```
┌─────────────────────────────────────────────────────────────┐
│                    Application Layer                        │
│              (HiveClass, Custom Apps, etc.)                 │
└───────────────────────┬─────────────────────────────────────┘
                        │
        ┌───────────────┴────────────────┐
        │                                │
┌───────▼────────────┐         ┌────────▼──────────────┐
│  WsPresenceClient  │         │  RtcPresenceClient    │
│  (Centralized)     │         │  (Distributed)        │
│                    │         │                       │
│  • Room Management │         │  • P2P Presence       │
│  • WS Signaling    │         │  • Stream Forwarding  │
│  • Initial Setup   │         │  • Topology-Aware     │
└───────┬────────────┘         └────────┬──────────────┘
        │                               │
        └───────────────┬───────────────┘
                        │
              ┌─────────▼──────────┐
              │    RTCService      │
              │   (Core Engine)    │
              │                    │
              │  3 RTCPeerConnections:
              │  ┌──────────────┐ │
              │  │  Signaling   │ │ • SDP Exchange
              │  └──────────────┘ │ • ICE Candidates
              │  ┌──────────────┐ │
              │  │    Data      │ │ • Application Messages
              │  └──────────────┘ │ • Health Monitoring
              │  ┌──────────────┐ │
              │  │    Media     │ │ • Audio/Video Tracks
              │  └──────────────┘ │ • Stream Management
              └─────────┬──────────┘
                        │
        ┌───────────────┼───────────────┐
        │               │               │
┌───────▼───────┐ ┌─────▼──────┐ ┌─────▼──────┐
│ ClientTopology│ │ServerTopology│ │ Network   │
│   Service     │ │   Service    │ │ Transport │
│               │ │              │ │           │
│ • Peer List   │ │ • Path Comp  │ │ • STUN    │
│ • Simple Mgmt │ │ • Mesh Opt   │ │ • TURN    │
└───────────────┘ └──────────────┘ └───────────┘
```

### Core Design Principles

1. **Separation of Concerns**: Three distinct connection roles with dedicated `RTCPeerConnection` instances
2. **Progressive Enhancement**: Start with WebSocket signaling, optionally switch to P2P
3. **Fault Tolerance**: Health monitoring, automatic cleanup, graceful degradation
4. **Flexibility**: Support for both centralized and distributed architectures
5. **Modern Standards**: Use latest WebRTC APIs (Promises, tracks, async/await)

---

## Design Decisions & Rationale

### Why Three Separate RTCPeerConnection Instances?

**Decision**: Use three separate `RTCPeerConnection` objects for SIGNALING, DATA, and MEDIA roles.

**Rationale**:

1. **Isolation**: Failures in one connection type don't affect others
   - Media can fail while data channel continues working
   - Signaling can be renegotiated without affecting data transfer

2. **Independent Configuration**: Each connection can have optimized settings
   - Media connection: Higher bandwidth, lower latency
   - Data connection: Reliable, ordered delivery
   - Signaling connection: Lightweight, ephemeral

3. **Debugging & Monitoring**: Easier to track connection states independently
   - `chrome://webrtc-internals/` shows clear separation
   - Connection health per role

4. **Lifecycle Management**: Independent connection/disconnection
   - Can close media while keeping data channel open
   - Can renegotiate signaling without disrupting streams

**Trade-offs**:
- ❌ More connections = more overhead (ICE, STUN, bandwidth)
- ✅ Better fault isolation and flexibility
- ✅ Clearer code organization
- ✅ Easier to add/remove capabilities

**Code Reference**: `client.js:9-11`

```javascript
ROLE_SIGNALING = 'signaling',
ROLE_DATA = 'data',
ROLE_MEDIA = 'media',
```

---

### Why P2P Mode Switching?

**Decision**: Support switching from WebSocket signaling to RTC data channel signaling.

**Rationale**:

1. **Reduced Server Load**: After initial connection, peers can signal directly
2. **Lower Latency**: Direct peer-to-peer signaling faster than server relay
3. **Scalability**: Server only needed for initial handshake
4. **Resilience**: Can fallback to WebSocket if P2P fails

**Implementation**: `client.js:595-606`

```javascript
_switchToP2P: {
    value: function() {
        if (!this._isP2P) {
            this._isP2P = true;
            this._sendSignaling({
                type: 'mode',
                cmd: 'p2p'
            });
        }
        this.dispatchEventNamed('switchToP2P');
    }
}
```

**Trade-offs**:
- ✅ Better scalability for large deployments
- ✅ Reduced server costs
- ❌ More complex signaling logic
- ❌ Requires handling two signaling modes

---

### Why Montage's Target Pattern?

**Decision**: Use Montage's `Target.specialize()` instead of ES6 classes.

**Rationale**:

1. **Montage Framework Integration**: Seamless integration with Montage ecosystem
2. **Event System**: Built-in event dispatching and listening
3. **Property Descriptors**: Fine-grained control over property behavior
4. **Backward Compatibility**: Works with Montage's serialization and binding

**Code Reference**: `client.js:23`

```javascript
var RTCService = Target.specialize({
    id: { value: null },
    _roomId: { value: null },
    // ...
});
```

---

## Connection Architecture

### Three-Role Connection Model

#### 1. Signaling Connection (`ROLE_SIGNALING`)

**Purpose**: WebRTC signaling (SDP offer/answer, ICE candidates)

**Lifecycle**:
```
Create → Send Offer → Receive Answer → Exchange ICE →
Create Data Channel → [Switch to P2P] → Optional: Close
```

**Characteristics**:
- First connection established
- Creates data channel for signaling messages
- Can be replaced by P2P signaling via DATA role
- Lightweight, only used for connection setup

**Code**: `client.js:66`

---

#### 2. Data Connection (`ROLE_DATA`)

**Purpose**: Application-level messaging and health monitoring

**Lifecycle**:
```
Create → Send Offer → Receive Answer → Exchange ICE →
Create Data Channel → Ping/Pong Loop → Message Handling → Close
```

**Characteristics**:
- **Primary channel** for application data
- Implements ping/pong health monitoring (5-second interval)
- Can forward signaling messages in P2P mode
- 5-miss threshold before disconnect
- Random jitter added to ping interval (prevent thundering herd)

**Health Monitoring**:
```javascript
// Ping every 5 seconds + random jitter
setTimeout(pingRemote, PING_TIMEOUT + (Math.random() * 1000));

// Track missed pongs
dataChannel.missedPongs++;
if (dataChannel.missedPongs > 5) {
    // Close connection
}
```

**Code**: `client.js:67, 512-541`

---

#### 3. Media Connection (`ROLE_MEDIA`)

**Purpose**: Audio/video stream transmission

**Lifecycle**:
```
Create → Attach Tracks → Send Offer → Receive Answer →
Exchange ICE → Stream Transmission → Detach Tracks → Close
```

**Characteristics**:
- Created on-demand when `attachStream()` is called
- Uses modern track-based API (`addTrack`, `removeTrack`)
- Fires `ontrack` events for remote streams
- Independent from data/signaling connections
- Can be created/destroyed without affecting data channel

**Modern Track API**:
```javascript
// Add tracks individually
stream.getTracks().forEach(function(track) {
    peerConnection.addTrack(track, stream);
});

// Remove tracks individually
var senders = peerConnection.getSenders();
senders.forEach(function(sender) {
    peerConnection.removeTrack(sender);
});
```

**Code**: `client.js:141-187`

---

### Connection Creation Flow

```
┌──────────────────────────────────────────────────────────┐
│ RTCService.connect(roomId)                               │
└────────────┬─────────────────────────────────────────────┘
             │
             ├─► Create SIGNALING PeerConnection
             │   ├─► Generate offer
             │   ├─► Set local description
             │   ├─► Send offer via WebSocket
             │   └─► Wait for answer
             │
             ├─► Create DATA PeerConnection
             │   ├─► Generate offer
             │   ├─► Set local description
             │   ├─► Send offer via WebSocket
             │   └─► Wait for answer
             │
             └─► Switch to P2P mode
                 └─► Signal via SIGNALING data channel
```

**Sequential Creation**: Signaling connection must complete before data connection (ensures signaling channel available).

**Code**: `client.js:62-76`

---

## State Management

### Connection State Machine

Montage WebRTC uses **bitflags** to track connection state progression.

#### State Flags

```javascript
CONNECTION_STATES = {
    descriptionCreated:     1,   // 0b00000001
    localDescriptionSet:    2,   // 0b00000010
    descriptionSent:        4,   // 0b00000100
    remoteDescriptionSet:   8,   // 0b00001000
    candidatesSent:         16,  // 0b00010000
    candidatesReceived:     32   // 0b00100000
}

CONNECTION_READY_TO_EXCHANGE_CANDIDATES = 15; // 0b00001111
```

**Code**: `client.js:12-21`

#### State Progression

```
Initial State: 0

Create Offer/Answer:
  state += descriptionCreated (1)
  → state = 1

Set Local Description:
  state += localDescriptionSet (2)
  → state = 3

Send Description:
  state += descriptionSent (4)
  → state = 7

Receive Remote Description:
  state += remoteDescriptionSet (8)
  → state = 15

Ready to Exchange ICE Candidates:
  state === CONNECTION_READY_TO_EXCHANGE_CANDIDATES (15)
  → Can call addIceCandidate()
```

#### Why Bitflags?

**Advantages**:
1. **Compact**: Single integer represents multiple states
2. **Fast**: Bitwise operations are O(1)
3. **Flexible**: Can check multiple states with `&`, `|`, `%`
4. **Atomic**: State transitions are simple additions

**State Checks**:

```javascript
// Check if ready for ICE candidates
if (peerConnection.state % CONNECTION_READY_TO_EXCHANGE_CANDIDATES === 0) {
    peerConnection.addIceCandidate(candidate);
}
```

**Code**: `client.js:333, 363, 376, 421, 495`

---

### Data Channel States

Each data channel tracks additional runtime state:

```javascript
dataChannel.missedPongs = 0;           // Health counter
dataChannel.isWaitingForPong = false;  // Ping state
```

**State Machine**:

```
[Open] → Send Ping → [WaitingForPong]
         ↓                    ↓
    [timeout]           [Receive Pong]
         ↓                    ↓
    missedPongs++       missedPongs = 0
         ↓                    ↓
    [WaitingForPong]        [Open]
         ↓
    missedPongs > 5?
         ↓
    [Close Connection]
```

**Code**: `client.js:510, 514, 523-537, 564-570`

---

## Message Flow & Protocols

### WebSocket Signaling Protocol

#### Message Structure

All WebSocket messages follow this JSON schema:

```json
{
  "source": "client-id",
  "type": "webrtc|close|mode|clientError",
  "cmd": "offer|answer|candidates|p2p",
  "data": {
    "targetRoom": "room-id",
    "targetClient": "peer-id",
    "role": "signaling|data|media",
    "state": 15,
    "descriptionVersion": "uuid",
    "description": { /* SDP */ },
    "candidates": [ /* ICE candidates */ ]
  }
}
```

#### Message Types

##### 1. Offer Message

```javascript
{
  source: "client-A",
  type: "webrtc",
  cmd: "offer",
  data: {
    targetRoom: "room-123",
    role: "signaling",
    state: 7,
    descriptionVersion: "uuid-v4",
    description: {
      type: "offer",
      sdp: "v=0\r\no=- ... "
    }
  }
}
```

##### 2. Answer Message

```javascript
{
  source: "client-B",
  type: "webrtc",
  cmd: "answer",
  data: {
    targetRoom: "room-123",
    targetClient: "client-A",
    role: "signaling",
    state: 15,
    descriptionVersion: "uuid-v4",
    description: {
      type: "answer",
      sdp: "v=0\r\no=- ... "
    }
  }
}
```

##### 3. ICE Candidates Message

```javascript
{
  source: "client-A",
  type: "webrtc",
  cmd: "candidates",
  data: {
    targetRoom: "room-123",
    targetClient: "client-B",
    role: "signaling",
    state: 15,
    candidates: [
      {
        candidate: "candidate:... ",
        sdpMid: "0",
        sdpMLineIndex: 0
      }
    ]
  }
}
```

##### 4. Close Message

```javascript
{
  type: "close",
  data: {
    role: "data",
    targetClient: "client-B"
  }
}
```

**Code**: `client.js:315-334, 473-490`

---

### P2P Signaling Protocol

Once P2P mode is enabled, signaling messages are sent through the SIGNALING data channel instead of WebSocket.

**Mode Switch Message**:

```javascript
{
  type: "mode",
  cmd: "p2p"
}
```

**Routing Logic**:

```javascript
_sendSignaling: {
    value: function(message) {
        if (this._isP2P && this._dataChannels[ROLE_SIGNALING]) {
            // Send via RTC data channel
            this._dataChannels[ROLE_SIGNALING].send(JSON.stringify(message));
        } else {
            // Send via WebSocket
            this.dispatchEventNamed('signalingMessage', true, true, message);
        }
    }
}
```

**Code**: `client.js:389-397`

---

### Data Channel Message Protocol

#### Ping/Pong Messages

```javascript
// Ping
{ "type": "ping" }

// Pong
{ "type": "pong" }
```

#### Application Messages

Application messages are dispatched as events without modification:

```javascript
{
  "type": "custom-message-type",
  "data": { /* application data */ },
  "timestamp": 1234567890
}
```

**Handling**:

```javascript
dataChannel.onmessage = function(event) {
    var message = JSON.parse(event.data);
    switch (message.type) {
        case 'ping':
            dataChannel.send('{ "type": "pong" }');
            break;
        case 'pong':
            dataChannel.missedPongs = 0;
            dataChannel.isWaitingForPong = false;
            break;
        default:
            self.dispatchEvent(event);  // Forward to application
            break;
    }
};
```

**Code**: `client.js:553-578`

---

### Complete Connection Establishment Sequence

```
Client A                WebSocket Server           Client B
   │                           │                      │
   ├──► connect(roomId) ───────┤                      │
   │                           │                      │
   ├──► offer (SIGNALING) ─────┼─────────────────────►│
   │                           │                      │
   │◄──── answer (SIGNALING) ──┼──────────────────────┤
   │                           │                      │
   ├──► ICE candidates ────────┼─────────────────────►│
   │◄──── ICE candidates ───────┼──────────────────────┤
   │                           │                      │
   │◄═══ SIGNALING Channel Established ══════════════►│
   │                           │                      │
   ├──► offer (DATA) ──────────┼─────────────────────►│
   │◄──── answer (DATA) ────────┼──────────────────────┤
   │                           │                      │
   │◄═══ DATA Channel Established ═══════════════════►│
   │                           │                      │
   ├──► mode: p2p ─────────────┼─────────────────────►│
   │                           │                      │
   │◄═══ Now Using P2P Signaling (via SIGNALING DC) ═►│
   │                           │                      │
   ├──► ping ──────────────────────────────────────────►│
   │◄──── pong ◄─────────────────────────────────────────┤
   │                           │                      │
   │   [5 seconds + jitter]    │                      │
   │                           │                      │
   ├──► ping ──────────────────────────────────────────►│
   │◄──── pong ◄─────────────────────────────────────────┤
   │                           │                      │
   │   [Application messages]  │                      │
   │◄═══════════════════════════════════════════════════►│
```

---

## Topology Algorithms

### ServerTopologyService Path Computation

The `ServerTopologyService` computes optimal paths through a mesh network for efficient message routing.

#### Algorithm: Greedy Path Building

**Goal**: Create paths that cover all nodes while minimizing path count.

**Code**: `server-topology-service.js:72-88`

```javascript
getPaths: {
    value: function() {
        var paths = [],
            orphanNodes = this._nodesList.map(function(x) { return x.id; });

        while (orphanNodes.length > 0) {
            var root = orphanNodes.shift(),
                path = [root],
                next = this._getNextNode(orphanNodes, root, path);

            while (next) {
                path.push(next);
                root = next;
                next = this._getNextNode(orphanNodes, root, path);
            }
            paths.push(path);
        }
        return paths;
    }
}
```

#### Algorithm Steps

```
1. Initialize: All nodes are "orphan" (unassigned to paths)
2. While orphanNodes is not empty:
   a. Take first orphan as path root
   b. Find next connected node (candidate selection)
   c. Add to path, repeat until no more candidates
   d. Store completed path
3. Return all paths
```

#### Candidate Selection Strategy

**Code**: `server-topology-service.js:95-130`

```javascript
_getNextNode: {
    value: function(orphanNodes, current, path) {
        var MAX_PATH_LENGTH = this._strongMesh ? 1 : 4;

        if (path.length < MAX_PATH_LENGTH) {
            var candidate,
                candidateConnections = this._nodesConnections[current];

            // Find orphan node connected to current
            for (var i = 0, count = candidateConnections.length; i < count; i++) {
                var connection = candidateConnections[i];
                if (orphanNodes.indexOf(connection) != -1) {
                    candidate = connection;
                    break;
                }
            }

            if (candidate) {
                orphanNodes.splice(orphanNodes.indexOf(candidate), 1);
                return candidate;
            }
        }
        return null;
    }
}
```

#### Mesh Patterns

##### Strong Mesh (`_strongMesh = true`)

- **MAX_PATH_LENGTH**: 1
- **Topology**: Star topology (hub-and-spoke)
- **Use Case**: Low CPU on teacher/server node
- **Characteristics**: All peers connect to central node

```
        Teacher
      /    |    \
     A     B     C
```

##### Weak Mesh (`_strongMesh = false`)

- **MAX_PATH_LENGTH**: 4
- **Topology**: Chain/line topology
- **Use Case**: Strong CPU on teacher/server node
- **Characteristics**: Peers form longer chains

```
Teacher → A → B → C → D
```

**Code**: `server-topology-service.js:91-93`

#### Path Example

**Network**:
```
Connections:
  Teacher: [A, B]
  A: [Teacher, C]
  B: [Teacher]
  C: [A, D]
  D: [C]
```

**Paths** (weak mesh):
```
[
  [Teacher, A, C, D],  // First path: longest chain
  [B]                  // Second path: orphan
]
```

**Paths** (strong mesh):
```
[
  [Teacher, A],
  [B],
  [C],
  [D]
]
```

---

### ClientTopologyService

**Simpler approach**: Just maintains a sorted list of peer IDs.

**Code**: `client-topology-service.js`

```javascript
exports.ClientTopologyService = Target.specialize({
    _peers: { value: null },

    addPeer: {
        value: function(peerId) {
            if (this._peers.indexOf(peerId) === -1) {
                this._peers.push(peerId);
                this._peers.sort();
            }
        }
    },

    getPeers: {
        value: function() {
            return this._peers;
        }
    }
});
```

**Use Case**: Client applications (like HiveClass) that just need a peer list.

---

## Health Monitoring System

### Ping/Pong Protocol

**Purpose**: Detect dead connections and clean up resources.

**Parameters**:
- **Ping Interval**: 5000ms + random jitter (0-1000ms)
- **Miss Threshold**: 5 missed pongs
- **Total Timeout**: ~30 seconds (5 × 6 seconds)

**Code**: `client.js:21, 513-541`

### State Machine

```
┌─────────┐
│  OPEN   │
└────┬────┘
     │ (every 5s + jitter)
     ▼
┌──────────────────┐
│  SEND PING       │
│ isWaitingForPong │
│     = true       │
└────┬─────────────┘
     │
     ├─────► [Wait 5s] ──────┐
     │                       │
     │ [Pong received]       │ [Timeout]
     ▼                       ▼
┌────────────┐      ┌──────────────┐
│ RESET      │      │ INCREMENT    │
│ missedPongs│      │ missedPongs  │
│    = 0     │      │              │
│ isWaiting  │      │ missedPongs  │
│   = false  │      │    > 5?      │
└────┬───────┘      └──────┬───────┘
     │                     │
     │                     ├─ No ──► [Continue]
     │                     │
     └─────────────────────┴─ Yes ─► [CLOSE CONNECTION]
                                     [Dispatch pongTimeout event]
```

### Implementation Details

#### Random Jitter

**Purpose**: Prevent all clients from pinging simultaneously (thundering herd problem).

```javascript
setTimeout(pingRemote, PING_TIMEOUT + (Math.random() * 1000));
```

**Jitter Range**: 0-1000ms (0-1 second)
**Total Interval**: 5000-6000ms

#### Missed Pong Handling

```javascript
if (dataChannel.missedPongs > 5) {
    // Dispatch timeout event
    self.dispatchEventNamed('pongTimeout', true, true, {
        client: self._targetClient,
        timeout: PING_TIMEOUT
    });

    // Force close connection
    try {
        dataChannel.close();
        peerConnection.close();
    } catch (err) {}
}
```

#### Pong Response

**Automatic response**: Server/client automatically responds to ping:

```javascript
case 'ping':
    dataChannel.send('{ "type": "pong" }');
    break;
```

**No application logic needed**: Health monitoring is fully automatic.

### Events

**Event**: `pongTimeout`

**Payload**:
```javascript
{
    client: "peer-id",
    timeout: 5000
}
```

**Use Case**: Application can listen for this event to update UI or trigger reconnection.

---

## P2P Mode Switching

### Motivation

**Problem**: WebSocket server becomes bottleneck for large-scale deployments.

**Solution**: After initial connection, switch to direct peer-to-peer signaling via the SIGNALING data channel.

### Mode Transition

```
┌─────────────────────┐
│  WebSocket Mode     │
│                     │
│  All signaling via  │
│  WebSocket server   │
└──────────┬──────────┘
           │
           │ connect() establishes
           │ SIGNALING + DATA channels
           ▼
┌─────────────────────┐
│ Transition          │
│                     │
│ Send "mode: p2p"    │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│   P2P Mode          │
│                     │
│  Signaling via      │
│  SIGNALING DC       │
└─────────────────────┘
```

### Implementation

**Code**: `client.js:389-397, 595-606`

```javascript
_sendSignaling: {
    value: function(message) {
        if (this._isP2P && this._dataChannels[ROLE_SIGNALING]) {
            // P2P mode: send via data channel
            this._dataChannels[ROLE_SIGNALING].send(JSON.stringify(message));
        } else {
            // WebSocket mode: dispatch event for WsPresenceClient
            this.dispatchEventNamed('signalingMessage', true, true, message);
        }
    }
}

_switchToP2P: {
    value: function() {
        if (!this._isP2P) {
            this._isP2P = true;
            // Notify server
            this._sendSignaling({
                type: 'mode',
                cmd: 'p2p'
            });
        }
        this.dispatchEventNamed('switchToP2P');
    }
}
```

### Message Forwarding in P2P Mode

When in P2P mode, the DATA channel can forward messages to other peers:

**Code**: `client.js:556-562`

```javascript
case 'webrtc':
case 'close':
    if (self._isP2P && message.data.targetClient &&
        self._removePeerId(message.data.targetClient) !== self.id) {
        // Forward to another peer
        self.dispatchEventNamed('forwardMessage', true, true, message);
    } else {
        // Handle locally
        self.dispatchEventNamed('signalingMessage', true, true, message);
    }
    break;
```

**Forwarding Logic**:
1. Check if in P2P mode
2. Check if message has `targetClient`
3. Check if target is NOT this client
4. If all true → forward to topology service for routing
5. Else → handle as signaling message

### Benefits vs Trade-offs

**Benefits**:
- ✅ Reduced server bandwidth (no message relay)
- ✅ Lower latency (direct peer connection)
- ✅ Better scalability
- ✅ Reduced server costs

**Trade-offs**:
- ❌ More complex routing logic
- ❌ Requires maintaining topology state
- ❌ Fallback handling needed
- ❌ Debugging is harder (distributed signaling)

---

## Media Management

### Track-Based API (Modern)

Montage WebRTC uses the **modern track-based API** instead of the deprecated stream-based API.

#### Adding Media Tracks

**Code**: `client.js:141-157`

```javascript
attachStream: {
    value: function(stream) {
        var self = this;
        this._localStream = stream;

        if (!this._peerConnections[ROLE_MEDIA]) {
            this._peerConnections[ROLE_MEDIA] = this._createPeerConnection(ROLE_MEDIA);
        }

        var mediaPeerConnection = this._peerConnections[ROLE_MEDIA];

        // Modern approach: add tracks individually
        stream.getTracks().forEach(function(track) {
            mediaPeerConnection.addTrack(track, stream);
        });

        return this._sendOffer(mediaPeerConnection);
    }
}
```

**Old (deprecated) way**:
```javascript
peerConnection.addStream(stream);  // ❌ Deprecated
```

**New (modern) way**:
```javascript
stream.getTracks().forEach(function(track) {
    peerConnection.addTrack(track, stream);  // ✅ Modern
});
```

#### Removing Media Tracks

**Code**: `client.js:159-187`

```javascript
detachStream: {
    value: function() {
        var mediaPeerConnection = this._peerConnections[ROLE_MEDIA];

        if (mediaPeerConnection) {
            // Modern approach: remove senders
            var senders = mediaPeerConnection.getSenders();
            senders.forEach(function(sender) {
                if (sender.track) {
                    mediaPeerConnection.removeTrack(sender);
                }
            });

            // Stop local tracks
            if (this._localStream) {
                this._localStream.getTracks().forEach(function(track) {
                    track.stop();
                });
                this._localStream = null;
            }
        }
    }
}
```

**Old (deprecated) way**:
```javascript
peerConnection.removeStream(stream);  // ❌ Deprecated
```

**New (modern) way**:
```javascript
var senders = peerConnection.getSenders();
senders.forEach(function(sender) {
    peerConnection.removeTrack(sender);  // ✅ Modern
});
```

### Receiving Media Tracks

**Code**: `client.js:283-291`

```javascript
peerConnection.ontrack = function(event) {
    // Event structure:
    // event.streams[0] = MediaStream containing track
    // event.track = MediaStreamTrack
    // event.receiver = RTCRtpReceiver

    var streamEvent = {
        stream: event.streams[0],
        remoteId: self._targetClient,
        type: 'addstream'
    };
    self.dispatchEvent(streamEvent);
};
```

**Old (deprecated) way**:
```javascript
peerConnection.onaddstream = function(event) {  // ❌ Deprecated
    var stream = event.stream;
};
```

**New (modern) way**:
```javascript
peerConnection.ontrack = function(event) {  // ✅ Modern
    var stream = event.streams[0];
    var track = event.track;
};
```

### Backward Compatibility

**Design**: The library maintains backward compatibility by dispatching an `addstream` event even though it uses the modern `ontrack` API internally.

**Why**: Applications using this library (like HiveClass) can still listen for `addstream` events without changes.

```javascript
// Application code (still works):
rtcService.addEventListener('addstream', function(event) {
    videoElement.srcObject = event.detail.stream;
});
```

---

## Event System

Montage WebRTC uses Montage's event system (inherited from `Target`) for all asynchronous notifications.

### Core Events

#### RTCService Events

| Event | When Fired | Payload | Use Case |
|-------|------------|---------|----------|
| `ready` | Data channel opened | `{ role: string }` | Know when channel is ready |
| `message` | Message received on DATA channel | Message object | Handle application messages |
| `addstream` | Remote track received | `{ stream, remoteId }` | Display remote video |
| `removestream` | Remote track removed | Event object | Remove remote video |
| `signalingMessage` | Need to send signaling | Message object | Forward to WS/signaling |
| `forwardMessage` | P2P message forwarding | Message object | Route to topology service |
| `connectionClose` | Connection closed | Peer ID | Update UI, cleanup |
| `sendError` | Send failed | Error object | Retry or notify user |
| `pongTimeout` | Health check failed | `{ client, timeout }` | Reconnect logic |
| `switchToP2P` | Switched to P2P mode | None | Update connection status |
| `clientError` | Peer error | Error object | Log/handle peer errors |

#### ServerTopologyService Events

| Event | When Fired | Payload | Use Case |
|-------|------------|---------|----------|
| `topologyChanged` | Topology updated | None | Recalculate paths, update UI |

### Event Dispatching

**Pattern**: Use Montage's `dispatchEventNamed()` for custom events.

```javascript
this.dispatchEventNamed('ready', true, true, { role: peerConnection.role });
//                        │      │     │      └─ payload
//                        │      │     └─ cancelable
//                        │      └─ bubbles
//                        └─ event name
```

**Code References**:
- `client.js:511` - ready event
- `client.js:290` - addstream event (dispatched from ontrack)
- `client.js:545` - connectionClose event
- `client.js:526` - pongTimeout event

### Event Listening

**Pattern**: Use Montage's `addEventListener()`.

```javascript
rtcService.addEventListener('message', function(event) {
    var message = event.detail;  // Payload is in event.detail
    console.log('Received:', message);
});
```

### Event Flow Example

```
User calls: rtcService.send({ type: 'chat', text: 'Hi' })
                │
                ▼
          JSON.stringify()
                │
                ▼
     dataChannel.send(jsonString)
                │
          [Network transmission]
                │
                ▼
  Remote: dataChannel.onmessage
                │
                ▼
          JSON.parse()
                │
                ▼
    Is it ping/pong? ──Yes──► Handle internally
                │
               No
                │
                ▼
    this.dispatchEvent(event)
                │
                ▼
  Application: addEventListener('message', handler)
```

---

## Error Handling & Resilience

### Error Handling Strategy

#### 1. Try/Catch Blocks

**Modern async/await** operations are wrapped in try/catch:

```javascript
_createOffer: {
    value: async function(peerConnection) {
        try {
            const offer = await peerConnection.createOffer();
            peerConnection.state += CONNECTION_STATES.descriptionCreated;
            return offer;
        } catch (error) {
            console.error('Failed to create offer:', error);
            throw error;  // Propagate to caller
        }
    }
}
```

**Code**: `client.js:359-370, 372-387, 399-437`

#### 2. Silent Failures

**Data channel operations** may fail silently with logging:

```javascript
try {
    dataChannel.send('{ "type": "ping"}');
} catch (err) {
    console.log('Unable to ping:', self._targetClient, err, dataChannel.readyState);
}
```

**Rationale**: Ping failures are recoverable; connection will be closed after 5 misses.

**Code**: `client.js:515-521`

#### 3. Cleanup on Error

**Connection close** wraps cleanup in try/catch:

```javascript
_closeConnectionWithRole: {
    value: function(role, isFromPeer) {
        if (this._dataChannels[role]) {
            try {
                this._dataChannels[role].close();
            } catch (err) {
            } finally {
                delete this._dataChannels[role];  // Always cleanup
            }
        }
        try {
            this._peerConnections[role].close();
        } catch (err) {
        } finally {
            delete this._peerConnections[role];  // Always cleanup
        }
    }
}
```

**Code**: `client.js:242-268`

#### 4. Error Events

**Signaling errors** dispatch events for application handling:

```javascript
catch (err) {
    var errorMessage = {
        source: this.id,
        type: 'clientError',
        cmd: 'send'
    };
    this.dispatchEventNamed('sendError', true, true, errorMessage);
}
```

**Code**: `client.js:231-238`

### Resilience Features

#### 1. Health Monitoring

- **Automatic detection** of dead connections
- **5-miss threshold** prevents false positives
- **Automatic cleanup** of failed connections

#### 2. Graceful Degradation

- **P2P fallback**: Can fallback to WebSocket if P2P fails
- **Role isolation**: Media failure doesn't affect data channel
- **Independent recovery**: Each connection role can be renegotiated

#### 3. State Verification

**Description version matching**: Prevents race conditions during renegotiation.

```javascript
case 'answer':
    if (data.descriptionVersion === this._peerConnections[role].descriptionVersion) {
        return this._receiveAnswer(this._peerConnections[role], data.description);
    } else {
        return Promise.resolve();  // Ignore stale answer
    }
```

**Code**: `client.js:216-223`

#### 4. ICE Candidate Buffering

**Problem**: ICE candidates may arrive before remote description is set.

**Solution**: Buffer candidates until ready.

```javascript
_receiveIceCandidates: {
    value: function(peerConnection, candidates) {
        if (peerConnection.state % CONNECTION_READY_TO_EXCHANGE_CANDIDATES === 0) {
            // Ready: add immediately
            for (var i = 0; i < candidates.length; i++) {
                peerConnection.addIceCandidate(new RTCIceCandidate(candidates[i]));
            }
        } else {
            // Not ready: buffer for later
            this._remoteIceCandidates[peerConnection.role] = candidates;
        }
    }
}
```

**Code**: `client.js:493-502`

---

## Performance Characteristics

### Connection Overhead

**Per Peer Connection**:
- 3 `RTCPeerConnection` objects (SIGNALING, DATA, MEDIA)
- 2-3 data channels (depends on roles active)
- ICE candidate collection (~10-50 candidates per connection)
- STUN/TURN queries (1-5 per connection)

**Memory Footprint** (approximate):
- RTCPeerConnection: ~1-2 MB each
- Data channel: ~100 KB each
- ICE candidates: ~1 KB each

**Total per peer**: ~3-7 MB

### Bandwidth Usage

**Signaling Phase** (one-time):
- SDP offer: ~2-5 KB
- SDP answer: ~2-5 KB
- ICE candidates: ~1-10 KB
- **Total**: ~5-20 KB per connection establishment

**Data Channel**:
- Ping/pong: ~50 bytes every 5 seconds = ~10 bytes/sec
- Application messages: Variable (user-controlled)

**Media Streams**:
- Video (720p): ~1-2 Mbps
- Video (1080p): ~2-4 Mbps
- Audio: ~50-100 Kbps

### Scalability Limits

#### Client-Side

**Browser limits** (Chrome):
- Max RTCPeerConnection objects: ~100-200
- Max data channels: ~500
- Max media tracks: ~100

**Practical limits** (this library):
- **With media**: 5-10 peers (bandwidth limited)
- **Data only**: 20-50 peers (CPU/memory limited)
- **Mesh topology**: Scales poorly beyond 10 nodes

#### Server-Side (WebSocket Signaling)

**Bottleneck**: Message relay for N peers = O(N²) messages

**Solution**: P2P mode switching reduces to O(N) after initial setup

### Optimization Strategies

#### 1. Lazy Connection Creation

Media connection only created when needed:

```javascript
if (!this._peerConnections[ROLE_MEDIA]) {
    this._peerConnections[ROLE_MEDIA] = this._createPeerConnection(ROLE_MEDIA);
}
```

#### 2. Connection Reuse

Renegotiation reuses existing connections instead of creating new ones:

```javascript
peerConnection.onnegotiationneeded = function() {
    if (peerConnection.state === CONNECTION_READY_TO_EXCHANGE_CANDIDATES) {
        peerConnection.isRenegociating = true;
    }
    self._sendOffer(peerConnection);
};
```

**Code**: `client.js:300-305`

#### 3. Mesh Pattern Optimization

**Strong mesh** (star topology) for low CPU:
- Teacher connects to all students
- Students don't connect to each other
- Better for weak teacher machine

**Weak mesh** (chain topology) for high CPU:
- Longer chains reduce connection count on teacher
- Better for strong teacher machine

**Code**: `server-topology-service.js:91-98`

---

## Security Architecture

### Network Security

#### 1. DTLS Encryption

**All WebRTC data is encrypted by default** using DTLS (Datagram Transport Layer Security).

- Data channels: DTLS-SRTP
- Media streams: SRTP

**No additional encryption needed** at application layer.

#### 2. STUN/TURN Authentication

**STUN servers**: Typically no authentication (public servers)

**TURN servers**: Require credentials (username/password)

```javascript
var stunServers = [
    { urls: "stun:stun.l.google.com:19302" },
    {
        urls: "turn:turn.example.com:3478",
        username: "user",
        credential: "pass"
    }
];

rtcService.init(clientId, stunServers);
```

### Application Security

#### 1. Message Validation

**Current**: Minimal validation (JSON parsing only)

**Recommendation**: Applications should validate message schemas:

```javascript
rtcService.addEventListener('message', function(event) {
    var message = event.detail;

    // Validate message structure
    if (!message.type || !message.data) {
        console.warn('Invalid message format');
        return;
    }

    // Validate message type
    var allowedTypes = ['chat', 'cursor', 'draw'];
    if (allowedTypes.indexOf(message.type) === -1) {
        console.warn('Unknown message type:', message.type);
        return;
    }

    // Handle message
    handleMessage(message);
});
```

#### 2. Connection Authorization

**Current**: No built-in authorization

**Recommendation**: Implement at application layer:

```javascript
// Server-side (before allowing room join)
function authorizeClient(clientId, roomId, token) {
    // Verify JWT token
    // Check permissions
    // Allow/deny room access
}
```

#### 3. DoS Protection

**Ping/pong timeout** provides basic DoS protection:
- Dead connections cleaned up after 30 seconds
- Prevents resource exhaustion from zombie connections

**Random jitter** prevents coordinated ping floods

---

## Extension Points

### 1. Custom Signaling Transport

**Current**: WebSocket via `WsPresenceClient`

**Extension**: Implement custom presence client

```javascript
var CustomPresenceClient = Target.specialize({
    rtcService: { value: null },

    init: {
        value: function(transportConfig) {
            this.rtcService = new RTCService().init(clientId, stunServers);

            // Listen for signaling messages
            this.rtcService.addEventListener('signalingMessage', function(event) {
                // Send via custom transport (HTTP long-polling, Socket.IO, etc.)
                customTransport.send(event.detail);
            });

            return this;
        }
    },

    receiveMessage: {
        value: function(message) {
            // Receive from custom transport
            this.rtcService.handleSignalingMessage(message);
        }
    }
});
```

### 2. Custom Topology Algorithm

**Current**: Greedy path building

**Extension**: Implement custom `ServerTopologyService`

```javascript
var CustomTopologyService = ServerTopologyService.specialize({
    getPaths: {
        value: function() {
            // Implement custom algorithm:
            // - Minimum spanning tree
            // - Shortest path first
            // - Load-balanced routing
            // - Geographic optimization

            return customPaths;
        }
    }
});
```

### 3. Message Middleware

**Pattern**: Intercept messages before dispatching

```javascript
rtcService._originalDispatchEvent = rtcService.dispatchEvent;
rtcService.dispatchEvent = function(event) {
    // Middleware: logging, filtering, transformation
    console.log('[Message]', event.data);

    // Call original
    return rtcService._originalDispatchEvent.call(this, event);
};
```

### 4. Custom Health Monitoring

**Current**: Ping/pong with 5-second interval

**Extension**: Override data channel initialization

```javascript
var CustomRTCService = RTCService.specialize({
    _initializeDataChannel: {
        value: function(peerConnection, dataChannel) {
            // Call parent
            this.super(peerConnection, dataChannel);

            // Add custom monitoring
            if (peerConnection.role === ROLE_DATA) {
                setInterval(function() {
                    // Custom health check (e.g., bandwidth monitoring)
                    measureBandwidth(dataChannel);
                }, 10000);
            }
        }
    }
});
```

### 5. Connection Plugins

**Pattern**: Lifecycle hooks

```javascript
var plugins = [];

// Plugin interface
var LoggingPlugin = {
    onConnectionCreate: function(peerConnection) {
        console.log('[Plugin] Connection created:', peerConnection.role);
    },

    onConnectionClose: function(peerConnection) {
        console.log('[Plugin] Connection closed:', peerConnection.role);
    }
};

plugins.push(LoggingPlugin);

// Call plugins at key points
peerConnection = this._createPeerConnection(role);
plugins.forEach(function(plugin) {
    plugin.onConnectionCreate(peerConnection);
});
```

---

## Integration Patterns

### HiveClass Integration

HiveClass uses **ClientTopologyService** for simple peer management:

```javascript
var topology = new ClientTopologyService();

// Track peers
rtcService.addEventListener('ready', function(event) {
    if (event.detail.role === 'data') {
        topology.addPeer(peerId);
    }
});

rtcService.addEventListener('connectionClose', function(event) {
    topology.removePeer(event.detail);
});

// Broadcast to all peers
function broadcast(message) {
    var peers = topology.getPeers();
    peers.forEach(function(peerId) {
        rtcService.send(message);
    });
}
```

### Best Practices

#### 1. Error Handling

```javascript
// Always handle connection failures
rtcService.addEventListener('connectionClose', function(event) {
    console.log('Peer disconnected:', event.detail);
    updateUI(event.detail);
});

rtcService.addEventListener('sendError', function(event) {
    console.error('Send failed:', event.detail);
    showErrorNotification();
});
```

#### 2. Resource Cleanup

```javascript
// Cleanup on page unload
window.addEventListener('beforeunload', function() {
    rtcService.quit(true);  // Notify peers
});
```

#### 3. Progressive Enhancement

```javascript
// Start with minimal features
rtcService.connect(roomId)
    .then(function() {
        // Connection established
        enableDataMessaging();
    })
    .then(function() {
        // Add media later if needed
        if (userWantsVideo) {
            return rtcService.attachStream(stream);
        }
    });
```

### Anti-Patterns

#### ❌ Creating Too Many Connections

```javascript
// DON'T: Create new service per peer
peers.forEach(function(peer) {
    var service = new RTCService();  // ❌ Memory leak
    service.connectToPeer(peer);
});

// DO: Use single service with topology
var service = new RTCService();
service.connect(roomId);  // ✅ Connects to all peers
```

#### ❌ Ignoring Connection State

```javascript
// DON'T: Send before ready
rtcService.send(message);  // ❌ May fail if not connected

// DO: Wait for ready event
rtcService.addEventListener('ready', function(event) {
    if (event.detail.role === 'data') {
        rtcService.send(message);  // ✅ Safe to send
    }
});
```

#### ❌ Not Handling Errors

```javascript
// DON'T: Ignore connection failures
rtcService.connect(roomId);  // ❌ No error handling

// DO: Handle errors
rtcService.connect(roomId)
    .catch(function(error) {
        console.error('Connection failed:', error);
        showRetryUI();
    });
```

---

## Conclusion

Montage WebRTC provides a **production-ready, fault-tolerant WebRTC implementation** with modern APIs and flexible architecture.

### Key Takeaways

1. **Three-role architecture** provides isolation and flexibility
2. **Bitflag state management** enables efficient state tracking
3. **P2P mode switching** improves scalability
4. **Health monitoring** ensures connection reliability
5. **Modern track API** future-proofs the implementation
6. **Event-driven design** enables loose coupling
7. **Extensible architecture** supports customization

### Future Enhancements

Potential areas for improvement:

1. **Perfect Negotiation**: Implement standard negotiation pattern
2. **Simulcast**: Support multiple quality layers
3. **E2E Encryption**: Add optional application-layer encryption
4. **Connection Stats**: Expose RTCStats for monitoring
5. **Bandwidth Estimation**: Adaptive bitrate control
6. **IPv6 Support**: Optimize for IPv6 networks

---

**Document Version**: 1.0
**Author**: Architecture analysis based on code review
**Last Updated**: December 2025
