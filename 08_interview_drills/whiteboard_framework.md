# Whiteboard Framework for Mobile System Design

**Audience:** Mobile engineers preparing for senior/staff-level interviews
**Goal:** Learn diagramming techniques and visual communication strategies that make your system design answers clear, memorable, and easy to follow.

---

## Why Diagramming Matters

A system design interview without a diagram is like giving directions without a map. Even a brilliant verbal explanation is hard to follow without visual anchors.

**Good diagrams:**
- Organize your own thinking
- Give the interviewer something to reference
- Create natural discussion points
- Demonstrate technical communication skills
- Make your answer memorable

**The goal isn't artistic beauty — it's clarity.**

---

## The Canvas: Space Management

### Physical Whiteboard

Divide the board into zones before you start:

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   REQUIREMENTS / ASSUMPTIONS          │    MAIN DIAGRAM         │
│   (top-left corner)                   │    (center, largest)    │
│                                       │                         │
│   • 100k DAU                          │                         │
│   • Offline required                  │                         │
│   • iOS first                         │                         │
│                                       │                         │
├───────────────────────────────────────┤                         │
│                                       │                         │
│   SCRATCH / DEEP DIVE                 │                         │
│   (bottom-left)                       │                         │
│                                       │                         │
│   [API shapes, state machines,        │                         │
│    detailed flows]                    │                         │
│                                       │                         │
└─────────────────────────────────────────────────────────────────┘
```

| Zone | Purpose | When to Use |
|------|---------|-------------|
| Top-left | Requirements, assumptions, constraints | Phase 1 (Clarify) |
| Center | Main architecture diagram | Phase 2 (High-Level) |
| Bottom-left | Deep dive details, sub-diagrams | Phase 3 (Deep Dive) |
| Right margin | Legend, key decisions | Throughout |

### Virtual Whiteboard (Zoom, Miro, etc.)

- Use **infinite canvas** — zoom out for context, zoom in for detail
- Create **named frames** for different sections
- Keep a **stable overview** visible; expand details in separate areas
- Practice with the specific tool beforehand (Excalidraw, Miro, FigJam)

---

## The Core Diagram Types

### 1. Component Diagram (The Default)

This is your primary diagram — boxes and arrows showing system components and data flow.

```
┌─────────────┐       HTTPS        ┌─────────────┐
│   Mobile    │───────────────────►│   API       │
│   Client    │◄───────────────────│   Gateway   │
└─────────────┘                    └──────┬──────┘
      │                                   │
      │                                   ▼
      ▼                            ┌─────────────┐
┌─────────────┐                    │  Services   │
│   Local     │                    │  (Auth,     │
│   Storage   │                    │   Feed,     │
│   (SQLite)  │                    │   Sync)     │
└─────────────┘                    └──────┬──────┘
                                          │
                                          ▼
                                   ┌─────────────┐
                                   │  Database   │
                                   │  + Cache    │
                                   └─────────────┘
```

**Rules:**
- **Boxes** = components (services, databases, clients)
- **Arrows** = data flow (label with protocol or data type)
- **Direction** matters (request vs response, or use bidirectional)
- **Group related items** visually (client-side left, server-side right)

### 2. Sequence Diagram (For Complex Flows)

Use when the **order of operations** matters:

```
  Client          API           Auth          Database
    │              │              │              │
    │──(1) Login───►              │              │
    │              │──(2) Validate►              │
    │              │              │──(3) Query───►
    │              │              │◄──(4) User───│
    │              │◄─(5) Token───│              │
    │◄─(6) Token───│              │              │
    │              │              │              │
    │──(7) Fetch───►              │              │
    │  + Token     │──────────────┼──(8) Query───►
    │              │◄─────────────┼──(9) Data────│
    │◄─(10) Data───│              │              │
```

**When to use:**
- Auth flows (token refresh, login)
- Multi-step transactions (checkout, booking)
- Real-time synchronization
- Error handling sequences

**Rules:**
- Vertical = time (top to bottom)
- Horizontal = different actors/systems
- Number the steps for easy reference
- Show both success and failure paths if relevant

### 3. State Machine Diagram (For Lifecycle)

Use when an entity has **distinct states and transitions**:

```
                    ┌─────────────┐
                    │   DRAFT     │
                    └──────┬──────┘
                           │ submit
                           ▼
                    ┌─────────────┐
        ┌──────────│   PENDING   │──────────┐
        │ timeout  └──────┬──────┘  approve │
        ▼                 │                 ▼
┌─────────────┐           │          ┌─────────────┐
│   EXPIRED   │           │ reject   │  APPROVED   │
└─────────────┘           │          └──────┬──────┘
                          ▼                 │ complete
                   ┌─────────────┐          ▼
                   │  REJECTED   │   ┌─────────────┐
                   └─────────────┘   │  COMPLETED  │
                                     └─────────────┘
```

**When to use:**
- Trip/order lifecycle
- Document/content states
- Connection states (WebSocket)
- Sync queue states

**Rules:**
- States in boxes (nouns: PENDING, APPROVED)
- Transitions on arrows (verbs: submit, approve)
- Show terminal states clearly
- Include error/timeout transitions

### 4. Data Flow Diagram (For Sync/Caching)

Use when showing **how data moves and transforms**:

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLIENT                                  │
│                                                                 │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐     │
│  │   UI    │───►│MemCache │───►│  Disk   │───►│  Sync   │     │
│  │ (Read)  │◄───│  (LRU)  │◄───│ (SQLite)│◄───│ Engine  │     │
│  └─────────┘    └─────────┘    └─────────┘    └────┬────┘     │
│                                                     │          │
└─────────────────────────────────────────────────────┼──────────┘
                                                      │
                                                      ▼
                                               ┌─────────────┐
                                               │   Server    │
                                               └─────────────┘
```

**When to use:**
- Caching layers
- Offline sync architecture
- Event pipelines
- Data transformation flows

### 5. Layer Diagram (For Architecture Depth)

Use when showing **abstraction layers**:

```
┌─────────────────────────────────────────────┐
│              Presentation Layer             │
│         (Views, ViewModels, UI State)       │
├─────────────────────────────────────────────┤
│              Domain Layer                   │
│      (Use Cases, Business Logic, Models)    │
├─────────────────────────────────────────────┤
│              Data Layer                     │
│   (Repositories, Network, Local Storage)    │
├─────────────────────────────────────────────┤
│              Infrastructure Layer           │
│    (HTTP Client, SQLite, Keychain, etc.)    │
└─────────────────────────────────────────────┘
```

**When to use:**
- Discussing clean architecture
- Explaining dependency direction
- Showing where specific logic lives

---

## Visual Conventions

### Consistent Shapes

Pick a convention and stick to it:

| Shape | Meaning |
|-------|---------|
| Rectangle | Service, component, process |
| Cylinder | Database, persistent storage |
| Rounded rectangle | Client/mobile app |
| Cloud shape | External service, 3rd party |
| Diamond | Decision point |
| Parallelogram | Queue, async buffer |

```
┌─────────┐    ╭─────────╮    ┌─────────┐    ☁─────────☁
│ Service │    │  Mobile │    ║Database ║    │  AWS S3  │
└─────────┘    ╰─────────╯    └─────────┘    ☁─────────☁
  Rectangle      Rounded        Cylinder        Cloud
```

### Arrow Styles

| Arrow | Meaning |
|-------|---------|
| `───►` | Request / data flow |
| `◄───` | Response |
| `◄──►` | Bidirectional |
| `- - ►` | Async / eventual |
| `═══►` | Batch / bulk |

### Labels

Always label:
- **Arrows** with what's flowing (data type, protocol, or action)
- **Boxes** with what they are (not implementation details)
- **Connections** with transport (HTTPS, WebSocket, gRPC)

```
            POST /events
┌─────────┐  (JSON)     ┌─────────┐
│ Client  │────────────►│   API   │
└─────────┘             └─────────┘
```

### Color Coding (If Available)

| Color | Use For |
|-------|---------|
| Black | Primary components and flows |
| Blue | Client-side components |
| Green | Success paths, data stores |
| Red | Error paths, failure states |
| Orange | Async, background, eventual |

**Tip:** In virtual whiteboards, use color. On physical whiteboards, use different line styles (solid, dashed, thick) if multiple markers aren't available.

---

## The Drawing Process

### Step 1: Anchor with the Client

Always start with the mobile client on the left:

```
┌─────────────┐
│   Mobile    │
│   Client    │
└─────────────┘
```

> "Let me start with the mobile client, since that's where the user interacts with the system."

### Step 2: Add the Primary Server

Draw the main backend the client talks to:

```
┌─────────────┐              ┌─────────────┐
│   Mobile    │─────────────►│   API       │
│   Client    │              │   Server    │
└─────────────┘              └─────────────┘
```

> "The client communicates with our API server over HTTPS."

### Step 3: Add Local Storage (Mobile-Specific)

This is critical for mobile — add it early:

```
┌─────────────┐              ┌─────────────┐
│   Mobile    │─────────────►│   API       │
│   Client    │              │   Server    │
└──────┬──────┘              └─────────────┘
       │
       ▼
┌─────────────┐
│   Local     │
│   Storage   │
└─────────────┘
```

> "For offline support, I'm adding local storage. The client reads from here first, and syncs with the server when online."

### Step 4: Expand Based on Requirements

Add components as needed:

```
┌─────────────┐              ┌─────────────┐       ┌─────────────┐
│   Mobile    │─────────────►│   API       │──────►│  Services   │
│   Client    │◄─────────────│   Gateway   │◄──────│             │
└──────┬──────┘   WebSocket  └─────────────┘       └──────┬──────┘
       │              │                                   │
       │              │                                   ▼
       ▼              ▼                            ┌─────────────┐
┌─────────────┐  ┌─────────────┐                   │  Database   │
│   Local     │  │   Push      │                   └─────────────┘
│   Storage   │  │   (APNs)    │
└─────────────┘  └─────────────┘
```

> "I'm adding a WebSocket connection for real-time updates, and APNs for when the app is backgrounded."

### Step 5: Show Data Flow Direction

Make sure arrows indicate the direction of data:

```
                    ──── Request ────►
┌─────────────┐                          ┌─────────────┐
│   Client    │                          │   Server    │
└─────────────┘                          └─────────────┘
                    ◄─── Response ────
```

---

## Narration Techniques

### Narrate as You Draw

Never draw silently. Explain every mark:

> "I'm drawing the mobile client here on the left... [draws box] ...and connecting it to the API gateway... [draws arrow] ...this is HTTPS for the primary request path."

### Point and Reference

Use your diagram as a visual anchor:

> "When the user opens the app, the UI reads from local storage here [point], not directly from the network. If we have cached data, we show it immediately, then sync with the server [trace the arrow] in the background."

### Number Complex Flows

For multi-step sequences, add numbers:

```
┌─────────────┐   (1)    ┌─────────────┐   (2)    ┌─────────────┐
│   Client    │─────────►│   API       │─────────►│   Auth      │
└─────────────┘          └─────────────┘          └──────┬──────┘
       ▲                        ▲                        │
       │         (4)            │          (3)           │
       └────────────────────────┴────────────────────────┘
```

> "Let me walk through the auth flow by number: Step 1, the client sends credentials to the API. Step 2, the API forwards to the auth service. Step 3, auth validates and returns a token. Step 4, the token flows back to the client."

### Use "Here" and "This" While Pointing

Physical/virtual pointing creates engagement:

> "When we go offline, this sync engine [point] queues mutations here [point], and flushes them when connectivity returns via this path [trace]."

---

## Deep Dive Diagrams

When diving deep, create **sub-diagrams** in a separate space rather than cluttering your main diagram.

### Zoom-In Pattern

Main diagram shows high-level:
```
┌─────────────┐
│  Sync       │
│  Engine     │
└─────────────┘
```

Deep dive expands it:
```
┌──────────────────────────────────────────────────────────────┐
│                       SYNC ENGINE                            │
│                                                              │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐  │
│  │ Mutation│───►│ Durable │───►│ Retry   │───►│ Network │  │
│  │ Creator │    │ Queue   │    │ Manager │    │ Client  │  │
│  └─────────┘    └─────────┘    └─────────┘    └─────────┘  │
│                      │                             │        │
│                      ▼                             ▼        │
│               ┌─────────┐                   ┌─────────┐    │
│               │ SQLite  │                   │  HTTP   │    │
│               └─────────┘                   └─────────┘    │
└──────────────────────────────────────────────────────────────┘
```

> "Let me zoom into the sync engine. I'll draw this in detail over here [move to scratch area]..."

### API Shape Sketches

When discussing API design, sketch the shape:

```
GET /trips/{id}
{
  "trip_id": "tr_123",
  "state": "IN_PROGRESS",
  "driver": { ... },
  "pickup_eta": { "seconds": 180 }
}
```

> "The API returns the trip state here. Notice I'm using a state machine pattern — the state field is an enum, and we return allowed_transitions so the client knows what actions are valid."

### State Transitions Table

Sometimes a table is clearer than a diagram:

```
┌─────────────┬────────────┬─────────────────┐
│ From State  │ Event      │ To State        │
├─────────────┼────────────┼─────────────────┤
│ PENDING     │ accept     │ ACCEPTED        │
│ PENDING     │ timeout    │ EXPIRED         │
│ ACCEPTED    │ arrive     │ ARRIVED         │
│ ARRIVED     │ start      │ IN_PROGRESS     │
│ IN_PROGRESS │ complete   │ COMPLETED       │
│ *           │ cancel     │ CANCELLED       │
└─────────────┴────────────┴─────────────────┘
```

---

## Common Diagram Mistakes

### Mistake 1: Too Much Detail Too Early

**Bad:** Starting with every microservice, cache layer, and queue.

**Good:** Start with 3-5 boxes. Add detail as needed.

> "Let me start simple with the core components, and we can drill into any of these."

### Mistake 2: No Labels

**Bad:** Boxes connected by unlabeled arrows.

**Good:** Every arrow labeled with what flows through it.

### Mistake 3: Inconsistent Direction

**Bad:** Some arrows go left-to-right, others right-to-left, randomly.

**Good:** Consistent flow direction. Typically:
- Left-to-right for client → server
- Top-to-bottom for time/sequence
- Bidirectional arrows for back-and-forth

### Mistake 4: Cluttered Single Diagram

**Bad:** Everything crammed into one space.

**Good:** Main diagram stays clean. Detail goes in separate areas.

### Mistake 5: Drawing Without Explaining

**Bad:** Five minutes of silent drawing.

**Good:** Continuous narration. Every stroke explained.

### Mistake 6: Forgetting Mobile Components

**Bad:** Architecture diagram with only server-side boxes.

**Good:** Always include:
- Local storage
- Sync mechanism
- Push notification path
- Offline indicators

---

## Virtual Whiteboard Tips

### Tool Familiarity

Practice with the actual tool before the interview:
- **Excalidraw** — Simple, free, commonly used
- **Miro** — Feature-rich, can be overwhelming
- **FigJam** — Clean, but requires Figma account
- **Zoom Whiteboard** — Basic but built into Zoom

### Zoom and Pan Smoothly

- Zoom out to show the big picture
- Zoom in when discussing details
- Don't make the interviewer dizzy with rapid navigation

### Use Frames/Sections

Create named areas:
- "Main Architecture"
- "Sync Engine Detail"
- "State Machine"

### Keyboard Shortcuts

Learn the shortcuts for:
- Rectangle (R)
- Arrow (A)
- Text (T)
- Select (V)
- Undo (Cmd+Z)

### Backup Plan

If the tool fails:
- Have a backup (share screen with a drawing app)
- Worst case: describe verbally with clear structure

> "Let me describe this as layers: At the top, we have the UI layer. Below that, the domain layer. Below that, the data layer..."

---

## Physical Whiteboard Tips

### Marker Management

- Test markers before starting (dead markers are common)
- Use **black** for main diagrams
- Use **blue** or **red** for emphasis or secondary flows
- Write larger than you think necessary

### Space Planning

- Don't start in the center — leave room to expand
- Start in the upper-left for requirements
- Use the largest continuous space for the main diagram

### Handwriting

- Print, don't write cursive
- ALL CAPS is often more readable
- Keep labels short (abbreviate if needed)

### Erasing Gracefully

- If you need to change something, it's fine
- "Let me adjust this..." [erase and redraw]
- Don't apologize excessively — iteration is normal

---

## Template: The Mobile System Design Canvas

Here's a reusable template for any mobile system design:

```
┌─────────────────────────────────────────────────────────────────────────┐
│ REQUIREMENTS                                                            │
│ • [User count, scale]                                                   │
│ • [Offline: yes/no]                                                     │
│ • [Real-time: yes/no]                                                   │
│ • [Platform: iOS/Android/Both]                                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│                         MAIN ARCHITECTURE                               │
│                                                                         │
│  ╭─────────────╮                           ┌─────────────┐              │
│  │   MOBILE    │──────── HTTPS ───────────►│    API      │              │
│  │   CLIENT    │◄──────────────────────────│   GATEWAY   │              │
│  ╰──────┬──────╯        WebSocket          └──────┬──────┘              │
│         │                  │                      │                     │
│         │                  │                      ▼                     │
│         ▼                  ▼               ┌─────────────┐              │
│  ┌─────────────┐    ┌─────────────┐        │  SERVICES   │              │
│  │   LOCAL     │    │    PUSH     │        │             │              │
│  │   STORAGE   │    │   (APNs)    │        └──────┬──────┘              │
│  └─────────────┘    └─────────────┘               │                     │
│                                                   ▼                     │
│                                            ┌─────────────┐              │
│                                            │  DATABASE   │              │
│                                            │  + CACHE    │              │
│                                            └─────────────┘              │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│ DEEP DIVE AREA                           │ KEY DECISIONS                │
│                                          │                              │
│ [State machine, API shapes,              │ • Local-first architecture   │
│  sequence diagrams, etc.]                │ • Idempotent sync with keys  │
│                                          │ • WebSocket + REST fallback  │
│                                          │ • Field-level conflict merge │
│                                          │                              │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Key Concepts

> "A diagram is a thinking tool, not just a presentation aid. Start with the mobile client and local storage — this anchors your design in mobile reality. Draw progressively: client first, then server, then expand. Narrate continuously; silent drawing loses the interviewer. Use consistent visual conventions so your diagram is self-explanatory. When diving deep, create sub-diagrams rather than cluttering your main view. The best diagrams are simple enough to understand at a glance, but detailed enough to drive a 20-minute conversation."
