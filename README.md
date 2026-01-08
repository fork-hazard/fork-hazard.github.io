# ForkHazard - Decentralized Mesh Network Messenger

A censorship-resistant messaging app that creates a peer-to-peer mesh network using Bluetooth LE and WiFi Direct, allowing communication even when traditional internet infrastructure is blocked or surveilled.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        INTERNET (via Starlink)                       │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      GATEWAY NODE (has internet)                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────────┐ │
│  │ BLE Service │──│ Mesh Router │──│ Internet Gateway Service    │ │
│  └─────────────┘  └─────────────┘  └─────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
         │                                       │
         │ BLE/WiFi Direct                       │ BLE/WiFi Direct
         ▼                                       ▼
┌─────────────────────┐                 ┌─────────────────────┐
│    RELAY NODE       │◄───────────────►│    RELAY NODE       │
│  (extends mesh)     │   BLE/WiFi      │  (extends mesh)     │
└─────────────────────┘                 └─────────────────────┘
         │                                       │
         │ BLE                                   │ BLE
         ▼                                       ▼
┌─────────────────────┐                 ┌─────────────────────┐
│    END USER         │                 │    END USER         │
│  (sends/receives)   │                 │  (sends/receives)   │
└─────────────────────┘                 └─────────────────────┘
```

## Core Components

### 1. Mesh Layer (`mesh/`)
- **BleManager** - Handles BLE advertising, scanning, and GATT connections
- **WifiDirectManager** - Higher bandwidth peer connections for nearby devices
- **MeshRouter** - Routes messages through the network using flooding + TTL
- **PeerDiscovery** - Maintains list of nearby peers and their capabilities

### 2. Crypto Layer (`crypto/`)
- **KeyManager** - Generates and stores Ed25519 keypairs
- **MessageEncryption** - End-to-end encryption using X25519 + ChaCha20-Poly1305
- **IdentityManager** - Manages pseudonymous identities

### 3. Data Layer (`data/`)
- **MessageStore** - SQLite database for messages (encrypted at rest)
- **PeerStore** - Known peers and their public keys
- **OutboxManager** - Queue of messages waiting to be sent

### 4. Service Layer (`service/`)
- **MeshService** - Foreground service that keeps the mesh alive
- **GatewayService** - Internet relay for gateway nodes

## Security Model

### Threat Model
- **Adversary capabilities**: Can seize devices, monitor radio traffic, run fake nodes
- **Goals**: Protect message content, protect user identities, provide plausible deniability

### Protections
1. **End-to-end encryption**: Only recipient can read messages
2. **Forward secrecy**: Compromised keys don't reveal past messages
3. **Onion routing** (future): Hide who is talking to whom
4. **Plausible deniability**: Messages look like random data, app has "decoy mode"

### What's NOT protected (yet)
- Metadata (who is near whom, when devices are active)
- Traffic analysis (message timing patterns)
- Device seizure (if unlocked)

## Building

### Prerequisites
- Android Studio Arctic Fox or later
- Android SDK 31+
- A physical Android device (emulators don't support BLE properly)

### Build Steps
```bash
# Clone and open in Android Studio
git clone <repo>
cd fork-hazard

# Build debug APK
./gradlew assembleDebug

# Install on connected device
./gradlew installDebug
```

## Testing the Mesh

### Minimum Setup
You need at least 2 Android phones to test basic peer discovery and messaging.

### Test Scenarios
1. **Direct messaging**: Two phones in BLE range
2. **Multi-hop**: Three phones in a line (A can reach B, B can reach C, A cannot reach C)
3. **Gateway**: One phone connected to WiFi with internet, others using mesh

## Protocol Specification

### Message Format
```
┌────────────────────────────────────────────────────────────┐
│ Header (32 bytes)                                          │
├────────────────────────────────────────────────────────────┤
│ version: u8        │ Message format version (currently 1) │
│ type: u8           │ MESSAGE, ACK, PEER_ANNOUNCE, etc     │
│ ttl: u8            │ Remaining hops (max 10)              │
│ flags: u8          │ ENCRYPTED, NEEDS_ACK, etc            │
│ message_id: [u8;16]│ Random unique ID                     │
│ timestamp: u64     │ Unix timestamp (for ordering)        │
│ reserved: [u8;4]   │ Future use                           │
├────────────────────────────────────────────────────────────┤
│ Routing (64 bytes)                                         │
├────────────────────────────────────────────────────────────┤
│ sender_id: [u8;32] │ Sender's public key fingerprint      │
│ dest_id: [u8;32]   │ Recipient's public key fingerprint   │
├────────────────────────────────────────────────────────────┤
│ Payload (variable, max 4KB)                                │
├────────────────────────────────────────────────────────────┤
│ encrypted_payload  │ ChaCha20-Poly1305 encrypted content  │
├────────────────────────────────────────────────────────────┤
│ Signature (64 bytes)                                       │
├────────────────────────────────────────────────────────────┤
│ signature: [u8;64] │ Ed25519 signature over all above     │
└────────────────────────────────────────────────────────────┘
```

### Routing Algorithm
1. **Flooding with TTL**: Messages broadcast to all peers, TTL decrements each hop
2. **Seen message cache**: Don't rebroadcast messages we've already seen
3. **Selective forwarding**: Gateway nodes can prioritize messages for internet

## Power Management

BLE scanning and advertising are battery-intensive. The app uses:
- **Duty cycling**: Scan for 5 seconds, sleep for 25 seconds
- **Adaptive intervals**: More aggressive when user is active
- **WiFi Direct on demand**: Only for large transfers or when BLE is saturated

## Legal Disclaimer

This software is provided for educational and humanitarian purposes. Users are responsible for complying with local laws. The developers do not endorse illegal activity but recognize that unjust laws exist and that communication is a human right.

## Contributing

This is a proof-of-concept. Key areas needing work:
- [ ] Onion routing for metadata protection
- [ ] iOS port (harder due to BLE restrictions)
- [ ] LoRa integration for longer range
- [ ] Decoy/steganography modes
- [ ] Formal security audit

## License

MIT License - Use freely, modify freely, help people communicate freely.
