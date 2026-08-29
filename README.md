# Alertbox.org

<div align="center">
  <p><strong>Open-source, privacy-first livestream tipping overlay and creator monetization platform.</strong></p>
  <p>
    <a href="https://alertbox.org">Website</a> •
    <a href="https://tip-to.me">TipTo.Me</a> •
    <a href="https://github.com/Ponlponl123-Labs/alertbox-org-webapp">Frontend</a> •
    <a href="https://github.com/Ponlponl123-Labs/alertbox-org-api">Backend API</a>
  </p>
</div>

---

## 1. System Architecture & Topology

Alertbox operates on a **zero-custody, real-time fan-out** architecture. Tips flow directly to creator payment gateways, while webhook signals are validated in constant time and broadcasted over WebSockets to OBS in <50ms.

```mermaid
flowchart TB
    subgraph Ingress["Ingress & Edge Routing"]
        DNS["Edge DNS / CDN\n(Cloudflare / Reverse Proxy)"]
        DomainWeb["tip-to.me / alertbox.org"]
        DomainAPI["api.alertbox.org"]
    end

    subgraph WebApp["Frontend WebApp (alertbox-org-webapp)"]
        direction TB
        SSR["Next.js 16 App Router\n(SSR Profile Pages & SEO)"]
        Studio["Creator Studio\n(OAuth, Theme Customizer, Badges)"]
        Sim["Alert Simulator\n(Interactive Visual Canvas)"]
        Overlay["OBS Browser Source Widget\n(WebGL, CSS FX, Web Audio, Web Speech TTS)"]
    end

    subgraph APICore["Backend API Engine (alertbox-org-api)"]
        direction TB
        Elysia["ElysiaJS Core Server\n(Bun Native Runtime)"]
        AuthModule["Auth & Session Guard\n(Discord OAuth2, ASN & Geo Tracking)"]
        WebhookEngine["Webhook Verification Engine\n(Constant-time HMAC-SHA256 & Token Cache)"]
        RelayWorker["Streamlabs Relay Forwarder\n(Async HTTP Dispatcher)"]
        WSBroadcaster["WebSocket Server\n(/v1/widget/:token)"]
    end

    subgraph Gateways["Direct Payment Gateways (Zero-Custody)"]
        BMAC["Buy Me a Coffee"]
        Kofi["Ko-fi"]
        Stripe["Stripe Direct"]
        FFP["FeelFreePay"]
        Xendit["Xendit"]
    end

    subgraph DataPlane["Data & Message Bus Layer"]
        DB[("MariaDB / MySQL\n(Prisma ORM: Users, Profiles, Widgets, TxLogs)")]
        Redis[("Redis / Dragonfly\n(Pub/Sub Channels & Token Cache)")]
    end

    subgraph ThirdParty["External Integrations"]
        StreamlabsAPI["Streamlabs Donations API\n(v2.0 Direct Relay)"]
    end

    %% Network Routing
    DNS --> DomainWeb
    DNS --> DomainAPI
    DomainWeb --> WebApp
    DomainAPI --> APICore

    %% User & Donor Actions
    Donor(["Donor / Supporter"]) -->|1. Visits tip-to.me/@creator| SSR
    Donor -->|2. Direct Payment| Gateways
    Streamer(["Content Creator"]) -->|Manage Settings & Themes| Studio
    Studio <-->|REST API / Session Auth| Elysia
    OBS(["OBS / Streamlabs Desktop"]) -->|Connects via Widget Token| Overlay
    Overlay <-->|Persistent WebSocket| WSBroadcaster

    %% Payment Webhooks
    Gateways -->|3. Signed HTTP POST Webhook| WebhookEngine
    WebhookEngine -->|4. Verify Signature & Invalidate Cache| DB
    WebhookEngine -->|5. Idempotent Tx Logging| DB
    WebhookEngine -->|6. Optional Donation Forwarding| RelayWorker
    RelayWorker -.->|Async HTTP POST| StreamlabsAPI

    %% Real-Time Broadcast Bus
    WebhookEngine -->|7. PUBLISH alertbox-org:alerts:widgetId| Redis
    Redis -->|8. Pattern Subscription Message| WSBroadcaster
    WSBroadcaster -->|9. Push Real-Time Alert Event <50ms| Overlay

    classDef edgeStyle fill:#1e1b4b,stroke:#818cf8,color:#fff,stroke-width:2px;
    classDef webStyle fill:#0f172a,stroke:#38bdf8,color:#fff,stroke-width:2px;
    classDef apiStyle fill:#4c0519,stroke:#fb7185,color:#fff,stroke-width:2px;
    classDef payStyle fill:#14532d,stroke:#4ade80,color:#fff,stroke-width:2px;
    classDef dataStyle fill:#1c1917,stroke:#a8a29e,color:#fff,stroke-width:2px;
    classDef extStyle fill:#312e81,stroke:#c084fc,color:#fff,stroke-width:2px;

    class DNS,DomainWeb,DomainAPI edgeStyle;
    class SSR,Studio,Sim,Overlay webStyle;
    class Elysia,AuthModule,WebhookEngine,RelayWorker,WSBroadcaster apiStyle;
    class BMAC,Kofi,Stripe,FFP,Xendit payStyle;
    class DB,Redis dataStyle;
    class StreamlabsAPI extStyle;
```

---

## 2. End-to-End Real-Time Event Pipeline

```mermaid
sequenceDiagram
    autonumber
    actor Donor as Donor
    participant Gateway as Payment Gateway (BMAC / Ko-fi / Stripe)
    participant Webhook as API Webhook Ingestion (ElysiaJS)
    participant DB as MariaDB (Prisma)
    participant SL as Streamlabs API
    participant Redis as Redis Pub/Sub
    participant WS as WebSocket Broadcaster
    participant OBS as OBS Browser Overlay

    Note over Donor,Gateway: Phase 1: Zero-Custody Direct Payment
    Donor->>Gateway: Direct transaction to Creator Account
    Gateway-->>Donor: Payment Confirmation & Receipt

    Note over Gateway,Webhook: Phase 2: Webhook Ingestion & Cryptographic Verification
    Gateway->>Webhook: POST /v1/webhook/:provider (Raw Body + Signature Header)
    Webhook->>DB: Fetch Creator Integration Secrets (Cache-Aside Pattern)
    critical Constant-Time HMAC Signature Check
        Webhook->>Webhook: crypto.timingSafeEqual(ComputedHMAC, HeaderSignature)
    option Valid Signature
        Webhook->>DB: Deduplicate & Record TransactionLog (COMPLETED)
    option Invalid Signature / Tampered
        Webhook-->>Gateway: HTTP 401 Unauthorized (Drop Request)
    end

    Note over Webhook,SL: Phase 3: Optional Streamlabs Donation Relay
    opt Streamlabs Relay Enabled
        Webhook->>DB: Record StreamlabsRelayLog (PENDING)
        Webhook->>Redis: PUBLISH streamlabs-relay-logs:userId (Created)
        Webhook->>SL: POST /api/v2.0/donations (Bearer StreamlabsSecret)
        SL-->>Webhook: HTTP 200 OK / Response
        Webhook->>DB: Update StreamlabsRelayLog (COMPLETED / FAILED)
        Webhook->>Redis: PUBLISH streamlabs-relay-logs:userId (Updated)
    end

    Note over Webhook,OBS: Phase 4: Lightweight Event Broadcast & Dynamic Rendering
    Webhook->>Redis: PUBLISH `alertbox-org:alerts:${widgetId}` (Tiny Event JSON ~120B)
    Redis->>WS: Pattern Message Received
    WS->>OBS: WebSocket Message Dispatch (<50ms)
    OBS->>OBS: Merge with Local Cached Settings -> Interpolate Template -> Trigger SFX/TTS/FX
    Webhook-->>Gateway: HTTP 200 OK

    Note over DB,OBS: Out-of-Band: Live Config Mutation (Push-Only)
    opt Creator Edits Widget Settings in Dashboard
        DB->>Redis: PUBLISH `alertbox-org:alerts:${widgetId}` (type: settings:update)
        Redis->>WS: Broadcast new configuration
        WS->>OBS: Hot-reload theme/audio/TTS settings in-memory (Zero Page Refresh)
    end
```

---

## 3. Security, Privacy & Reliability Matrix

```mermaid
flowchart LR
    subgraph ZeroCustody["Zero-Custody Guarantee"]
        direction TB
        ZC1["100% Direct Payouts"]
        ZC2["No Intermediary Wallet"]
        ZC3["No Fund Holding Accounts"]
    end

    subgraph Cryptography["Cryptographic Security"]
        direction TB
        CS1["Constant-Time HMAC-SHA256 Verification"]
        CS2["Replay Attack Prevention via Unique Tx ID"]
        CS3["Timing-Safe Equality Checks"]
    end

    subgraph Privacy["Streamer Privacy & Access Control"]
        direction TB
        PR1["Obfuscated Widget Tokens"]
        PR2["Streamer-Mode UI Leak Masking"]
        PR3["Token Rotation & Audit Logs"]
        PR4["GeoIP & ASN Session Fingerprinting"]
    end

    subgraph HighAvailability["Resilience & Performance (K3s Friendly)"]
        direction TB
        HA1["Tiny Alert Payloads (~120B) & Zero-Polling"]
        HA2["Push-Only Config Invalidation (No Interval Requests)"]
        HA3["In-Memory Client-Side Template Interpolation"]
    end

    ZeroCustody --- Cryptography --- Privacy --- HighAvailability
```

---

## 4. Repositories & Tech Stack Breakdown

| Component | Repository | Description | Key Technologies |
| :--- | :--- | :--- | :--- |
| **Frontend / WebApp** | [`alertbox-org-webapp`](https://github.com/Ponlponl123-Labs/alertbox-org-webapp) | Creator dashboard, public tip profile (`tip-to.me/@user`), live simulator, interactive WebGL overlays | Next.js 16, React 19, Tailwind CSS v4, Motion, Zustand, Lucide/Phosphor Icons |
| **Backend / API** | [`alertbox-org-api`](https://github.com/Ponlponl123-Labs/alertbox-org-api) | High-throughput WebSocket broadcaster, user sessions, secure webhook ingestion, relay workers | Bun, ElysiaJS, Prisma ORM, Redis / Dragonfly, TypeBox, MariaDB/MySQL |

---

## 5. Optimized WebSocket & Event Specifications

### 1. Connection Handshake & Initial Config Sync
Endpoint: `WS /v1/widget/:token` or `GET /v1/widget/:token/settings`
```json
// Server -> Client (Handshake & Cached Configuration)
{
  "type": "settings:init",
  "widgetId": "cm3a9bc0...",
  "settings": {
    "globalVolume": 0.8,
    "events": {
      "TIP": {
        "prefix": "Thank you {{user}}",
        "subfix": "for the {{amount}} {{currency}} tip!",
        "messageLayout": "image-above",
        "minVisibleDuration": 4.0,
        "animIn": "fade_in_up",
        "animOut": "fade_out_up",
        "image": "https://cdn.alertbox.org/assets/sparkle.gif",
        "sound": "https://cdn.alertbox.org/assets/chime.mp3",
        "soundVolume": 0.8,
        "accentColor": 16007006,
        "ttsEnabled": true,
        "ttsMinTip": 500,
        "ttsVoice": "en-US-Standard-C"
      }
    }
  }
}
```

### 2. High-Frequency Alert Broadcast (Tiny Delta ~120 Bytes)
```json
// Server -> Client (Dispatched per donation event)
{
  "type": "alert",
  "id": "e2c3497d-6f78-4a5c-897b-cf10972b2100",
  "event": "TIP",
  "name": "Alex",
  "amount": 2500,
  "currency": "USD",
  "message": "Keep up the awesome streaming!"
}
```

### 3. Reactive Config Push (Push-Only on Dashboard Save)
```json
// Server -> Client (Hot-reloaded in overlay memory without refresh)
{
  "type": "settings:update",
  "updatedAt": 1724930000,
  "settings": {
    /* Updated widget settings diff or snapshot */
  }
}
```

---

## 6. Features

- **Direct Creator Payouts**: Connect Buy Me a Coffee, Ko-fi, Stripe, FeelFreePay, or Xendit without platform cuts.
- **Sub-Millisecond Alert Broadcaster**: Redis Pub/Sub + Bun WebSocket engine delivers instant alerts to OBS/Streamlabs overlays.
- **Customizable Visual Overlays**: Glassmorphism, Neon Glow, and Minimal Bold themes with WebGL shader enhancements.
- **Automated Text-To-Speech (TTS)**: Multi-voice speech synthesis with customizable minimum donation thresholds, volume, speed, and pitch.
- **Streamlabs Forwarding Relay**: Mirror donations automatically to existing Streamlabs setups with real-time delivery logs.
- **Streamer-Mode Protection**: Obfuscate sensitive credentials, widget tokens, and integration keys during live broadcasts.
- **Multi-Language Support**: Full English & Thai localized interfaces.

---

## 7. Quick Navigation & Setup

- **Frontend Guide**: Follow setup instructions in [alertbox-org-webapp/README.md](https://github.com/Ponlponl123-Labs/alertbox-org-webapp#readme)
- **Backend Guide**: Follow setup instructions in [alertbox-org-api/README.md](https://github.com/Ponlponl123-Labs/alertbox-org-api#readme)
- **Contributing**: Refer to `CONTRIBUTING.md` across both repositories.

---

## License

All Alertbox.org projects are open-source under the [MIT License](https://opensource.org/licenses/MIT).