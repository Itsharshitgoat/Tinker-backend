# Tinker — Privacy-Preserving On-Device Visual Browser Agent

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Problem Analysis & Motivation](#2-problem-analysis--motivation)
3. [Proposed Solution Overview](#3-proposed-solution-overview)
4. [Technology Stack](#4-technology-stack)
5. [System Architecture](#5-system-architecture)
6. [Detailed Component Design](#6-detailed-component-design)
7. [Privacy & Redaction Pipeline](#7-privacy--redaction-pipeline)
8. [Server-Side AI Engine](#8-server-side-ai-engine)
9. [User Interface & Interaction Model](#9-user-interface--interaction-model)
10. [Data Flow & Communication Protocol](#10-data-flow--communication-protocol)
11. [Evaluation Strategy (Mapped to Hackathon Metrics)](#11-evaluation-strategy-mapped-to-hackathon-metrics)
12. [SDLC — Development Phases & Timeline](#12-sdlc--development-phases--timeline)
13. [Agility & Adaptability Mechanisms](#13-agility--adaptability-mechanisms)
14. [Risk Analysis & Mitigation](#14-risk-analysis--mitigation)
15. [Testing Strategy](#15-testing-strategy)
16. [Deployment & Demo Plan](#16-deployment--demo-plan)
17. [Future Scope & Extensions](#17-future-scope--extensions)
18. [Appendix — Key References & Papers](#18-appendix--key-references--papers)

---

## 1. Executive Summary

**Tinker** is a privacy-preserving, on-device visual perception agent that runs as a browser extension (Chrome & Firefox). It enables an AI agent to "see" the user's screen, understand the visual context, and assist with complex web workflows — **without ever transmitting sensitive user data to any server**.

The system works by:
1. **Locally capturing** the browser viewport and parsing the DOM.
2. **Detecting and redacting** all sensitive/PII data on-device using a multi-layered pipeline (DOM inspection + regex patterns + face detection + targeted OCR) — all powered by WebGPU-accelerated ML models running directly inside the browser.
3. **Transmitting only the sanitized, anonymized visual context** to a central server running an open-source Vision-Language Model (VLM).
4. The server **interprets the sanitized context** and returns actionable UI commands (click, type, scroll, navigate) that the local client executes.

**Zero PII ever leaves the browser. The server is fully aware of the redaction scheme and operates on anonymized data only.**

---

## 2. Problem Analysis & Motivation

### 2.1 The Core Challenge

| Dimension | Current State | Desired State |
|-----------|--------------|---------------|
| **Agent Deployment** | Server-side only; user must share full screen data | On-device perception with server-side reasoning |
| **Privacy** | All visual data sent to cloud, exposing PII | PII detected & redacted locally before any network call |
| **Resource Constraints** | Local machines can't host full VLM pipelines | Lightweight client-side models via WebGPU/WASM; heavy reasoning offloaded to server |
| **Adaptability** | Agents hardcoded for specific websites | Agent works on any webpage, any task, any domain |

### 2.2 Why This Matters

- **Government portals** (DigiLocker, income tax filing, passport services) contain Aadhaar numbers, PAN cards, bank details — none of this should ever leave the user's machine.
- **Enterprise workflows** involve internal documents, trade secrets, authentication credentials.
- **Healthcare & Education** portals display medical records, student IDs, and personal information.
- A truly privacy-preserving browser agent **unlocks AI assistance for all these sensitive domains** where cloud-only agents are currently unusable.

### 2.3 Key Constraints from Problem Statement

1. Client-side Vision Transformer (ViT) or equivalent must run **in the browser** via WebGPU/WebAssembly.
2. Sensitive data must be **dynamically detected and redacted** (faces blurred, passwords blacked out, PII masked).
3. Only **anonymized, unidentifiable data** transmitted to server.
4. Server must be **aware of the redaction scheme** and process data accordingly.
5. Server returns **actionable commands** (UI actions) that the local client executes.
6. Must work on **popular browsers** (Chrome, Firefox).
7. Server can use any **open-source/open-weights** model.

---

## 3. Proposed Solution Overview

### 3.1 The Split-Agent Architecture

We adopt a **dual-agent split architecture** where responsibilities are cleanly divided:

```
┌─────────────────────────────────────────────┐     ┌─────────────────────────────────────────────┐
│           🖥️ EDGE CLIENT AGENT              │     │           ☁️ CLOUD SERVER AGENT              │
│           (Browser Extension)                │     │           (Python Backend)                   │
├─────────────────────────────────────────────┤     ├─────────────────────────────────────────────┤
│                                             │     │                                             │
│  • Screen Capture & DOM Parsing             │     │  • Receive sanitized visual context          │
│  • On-device Vision Model (WebGPU)          │     │  • VLM reasoning over redacted screenshots   │
│  • Multi-layer PII Detection                │ ──► │  • Task planning & action generation         │
│    - DOM attribute scanning                 │     │  • Coordinate prediction (grounding)         │
│    - Regex pattern matching                 │     │  • Multi-step workflow orchestration          │
│    - Face detection (MediaPipe BlazeFace)   │ ◄── │  • Action command JSON response              │
│    - Targeted OCR (Tesseract.js)            │     │                                             │
│  • Canvas-based semantic redaction          │     │  Models: Qwen2.5-VL-7B / UI-TARS-7B         │
│  • Action execution (click/type/scroll)     │     │  Serving: vLLM / SGLang                     │
│                                             │     │                                             │
└─────────────────────────────────────────────┘     └─────────────────────────────────────────────┘
```

### 3.2 The Three Pillars

| Pillar | What It Does | Where It Runs |
|--------|-------------|---------------|
| **1. Perception** | Captures the screen, extracts DOM structure, runs local ViT for visual understanding | Browser (Client) |
| **2. Privacy** | Detects all PII/sensitive data and redacts it using semantic masking before transmission | Browser (Client) |
| **3. Reasoning** | Interprets the sanitized context, plans next actions, generates executable commands | Server |

---

## 4. Technology Stack

### 4.1 Client-Side (Browser Extension)

| Layer | Technology | Version | Purpose |
|-------|-----------|---------|---------|
| **Extension Framework** | Chrome Manifest V3 / Firefox WebExtensions | MV3 | Cross-browser extension architecture |
| **ML Runtime** | ONNX Runtime Web | v1.19+ | WebGPU/WASM inference engine for running vision models |
| **ML Abstraction** | Transformers.js | v3.x (`@huggingface/transformers`) | High-level pipeline API for running HuggingFace models in browser |
| **Face Detection** | MediaPipe Tasks Vision | Latest | Ultra-fast BlazeFace model (230KB, <10ms on GPU) |
| **OCR Engine** | Tesseract.js | v5.x | Targeted text recognition on image regions |
| **NER (Optional)** | compromise.js / Xenova/bert-base-NER | Latest | Named entity recognition for unstructured PII |
| **GPU Acceleration** | WebGPU API | Baseline 2025 | Hardware-accelerated compute shaders for ML inference |
| **Fallback Compute** | WebAssembly SIMD + Multi-threading | WASM v2 | CPU fallback when WebGPU unavailable |
| **Build Tool** | Vite + Rollup | v6.x | Module bundling, code splitting, WASM asset handling |
| **Language** | TypeScript | v5.x | Type-safe client code |

### 4.2 Server-Side (AI Backend)

| Layer | Technology | Version | Purpose |
|-------|-----------|---------|---------|
| **Primary VLM** | Qwen2.5-VL-7B-Instruct | Latest | Visual understanding + action grounding on sanitized screenshots |
| **Alternative VLM** | UI-TARS-7B (ByteDance) | v1.5/v2 | Native GUI agent model — perceives pixels, outputs executable actions |
| **Fallback VLM** | MiniCPM-V 2.6 (8B) | Latest | Lightweight; runs on single 16GB GPU |
| **Model Serving** | vLLM | v0.6+ | High-throughput serving with PagedAttention, continuous batching |
| **API Framework** | FastAPI | v0.115+ | Async REST + WebSocket endpoints |
| **WebSocket Server** | uvicorn + websockets | Latest | Persistent bidirectional communication with client |
| **Language** | Python | 3.11+ | Server application code |
| **Containerization** | Docker + Docker Compose | Latest | Reproducible deployment |
| **GPU Runtime** | CUDA 12.x + cuDNN 9.x | Latest | Server-side GPU acceleration |

### 4.3 Development & DevOps

| Tool | Purpose |
|------|---------|
| **Git + GitHub** | Version control, collaboration, CI/CD |
| **ESLint + Prettier** | Client-side code quality |
| **Ruff + Black** | Server-side Python linting/formatting |
| **Pytest** | Server-side unit & integration tests |
| **Vitest** | Client-side unit tests |
| **GitHub Actions** | CI pipeline for build, lint, test |

### 4.4 Why These Specific Technologies?

> [!IMPORTANT]
> **WebGPU over WebGL:** WebGPU provides native compute shaders (WGSL), workgroup shared memory, and async buffer readback — delivering **3x–13x faster ML inference** compared to WebGL's texture-packing workarounds. It is now part of the Web Baseline (Chrome 113+, Firefox 141+, Safari 26+).

> [!IMPORTANT]
> **ONNX Runtime Web + Transformers.js:** This combination gives us the best of both worlds — ONNX Runtime Web provides the optimized low-level WebGPU inference engine, while Transformers.js provides the high-level pipeline abstraction with automatic model downloading, preprocessing, and quantization support from the HuggingFace Hub.

> [!IMPORTANT]
> **Qwen2.5-VL-7B:** Chosen as the primary server VLM because it supports dynamic resolution processing (handles native aspect ratios without distorting UI elements), outputs grounding coordinates in normalized `[0, 1000]` format, and runs efficiently on a single RTX 4090 or A100 via vLLM. It is fully open-weights.

> [!IMPORTANT]
> **MediaPipe BlazeFace over TensorFlow.js:** BlazeFace via MediaPipe Tasks Vision is only 230KB, runs in <10ms on GPU delegate, and is production-battle-tested across billions of Google devices. No other face detection model comes close to this latency-size ratio.

---

## 5. System Architecture

### 5.1 High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CHROME / FIREFOX BROWSER                       │
│                                                                             │
│  ┌───────────────────┐    postMessage     ┌────────────────────────────┐    │
│  │   🌐 Web Page     │ ◄════════════════► │  📜 Content Script         │    │
│  │   (User's Tab)    │                    │  (Isolated World)          │    │
│  │                   │                    │                            │    │
│  │  - Visible DOM    │                    │  - DOM Tree Extraction     │    │
│  │  - User Data      │                    │  - Sensitive Field Scan    │    │
│  │  - Forms/Inputs   │                    │  - Text Node PII Regex    │    │
│  │                   │                    │  - Bounding Box Mapping   │    │
│  └───────────────────┘                    │  - Action Execution       │    │
│                                           │    (click/type/scroll)    │    │
│                                           └────────────┬───────────────┘    │
│                                                        │                    │
│                                             chrome.runtime.message          │
│                                                        │                    │
│                                                        ▼                    │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  ⚙️ Background Service Worker                                      │    │
│  │                                                                     │    │
│  │  - Tab management (chrome.tabs)                                     │    │
│  │  - Screenshot capture (chrome.tabs.captureVisibleTab)               │    │
│  │  - WebSocket client → Server                                        │    │
│  │  - Offscreen Document lifecycle management                          │    │
│  │  - Agent loop orchestration (Observe → Redact → Send → Execute)     │    │
│  │                                                                     │    │
│  │  ⚠️ NO DOM, NO Canvas, NO WebGPU access (MV3 limitation)            │    │
│  └─────────────────────────┬───────────────────────────────────────────┘    │
│                            │                                                │
│                 chrome.runtime.message / port                               │
│                            │                                                │
│                            ▼                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  🧠 Offscreen Document (ML Inference Engine)                       │    │
│  │                                                                     │    │
│  │  - Full DOM, Canvas 2D, WebGPU, WebAssembly access                  │    │
│  │  - MediaPipe BlazeFace → Face Detection (<10ms)                     │    │
│  │  - EfficientViT-M0 via ONNX Runtime Web → Visual Understanding     │    │
│  │  - Tesseract.js → Targeted OCR on image regions                     │    │
│  │  - Canvas Redaction Compositor → Semantic masking                    │    │
│  │  - WebP Compression → Bandwidth-optimized output                    │    │
│  │                                                                     │    │
│  │  Output: Sanitized Image (WebP) + Redaction Manifest (JSON)         │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │
                              WebSocket (wss://) — Binary + JSON
                              Only sanitized data transmitted
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          ☁️ CENTRAL AI SERVER                               │
│                                                                             │
│  ┌─────────────────────┐    ┌─────────────────────┐    ┌────────────────┐  │
│  │  🔌 WebSocket       │    │  🤖 VLM Engine      │    │  📋 Session    │  │
│  │  Gateway            │───►│  (vLLM / SGLang)    │    │  Manager       │  │
│  │  (FastAPI +         │    │                     │    │                │  │
│  │   uvicorn)          │    │  Qwen2.5-VL-7B      │    │  - Task state  │  │
│  │                     │◄───│  or UI-TARS-7B      │    │  - History     │  │
│  │  Accepts sanitized  │    │                     │    │  - Multi-step  │  │
│  │  frames + DOM tree  │    │  Understands         │    │    planning    │  │
│  └─────────────────────┘    │  [REDACTED: ...] tags│    └────────────────┘  │
│                             │  and works around    │                        │
│                             │  them                │                        │
│                             └─────────────────────┘                        │
│                                                                             │
│  Output: Action JSON → { type: "CLICK", target_id: 3, coordinates: ... }   │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Component Interaction Sequence

```
 User              Content Script       Service Worker       Offscreen Doc        Server VLM
  │                     │                    │                    │                    │
  │  "Help me fill      │                    │                    │                    │
  │   this form"        │                    │                    │                    │
  │────────────────────►│                    │                    │                    │
  │                     │                    │                    │                    │
  │                     │  1. Scan DOM for   │                    │                    │
  │                     │  sensitive fields   │                    │                    │
  │                     │  & extract tree    │                    │                    │
  │                     │───────────────────►│                    │                    │
  │                     │  DOM tree + PII    │                    │                    │
  │                     │  bounding boxes    │                    │                    │
  │                     │                    │                    │                    │
  │                     │                    │ 2. captureVisibleTab()                  │
  │                     │                    │  → Raw screenshot  │                    │
  │                     │                    │───────────────────►│                    │
  │                     │                    │  3. Raw image +    │                    │
  │                     │                    │     PII boxes      │                    │
  │                     │                    │                    │                    │
  │                     │                    │                    │ 4. Run ML Pipeline │
  │                     │                    │                    │  - BlazeFace       │
  │                     │                    │                    │  - Targeted OCR    │
  │                     │                    │                    │  - Canvas redact   │
  │                     │                    │                    │  - WebP compress   │
  │                     │                    │                    │                    │
  │                     │                    │◄───────────────────│                    │
  │                     │                    │  5. Sanitized WebP │                    │
  │                     │                    │  + Redaction JSON  │                    │
  │                     │                    │                    │                    │
  │                     │                    │───────────────────────────────────────►│
  │                     │                    │  6. WebSocket: Sanitized image          │
  │                     │                    │     + DOM tree + Redaction manifest     │
  │                     │                    │                                         │
  │                     │                    │                    │  7. VLM Reasoning  │
  │                     │                    │                    │  over sanitized    │
  │                     │                    │                    │  context           │
  │                     │                    │                    │                    │
  │                     │                    │◄────────────────────────────────────────│
  │                     │                    │  8. Action: { type: "CLICK",            │
  │                     │                    │     target_id: 3, coords: {x,y} }      │
  │                     │                    │                    │                    │
  │                     │◄───────────────────│                    │                    │
  │                     │  9. Execute click  │                    │                    │
  │                     │  on target element │                    │                    │
  │◄────────────────────│                    │                    │                    │
  │  10. Page updates   │                    │                    │                    │
  │  (form submitted,   │                    │                    │                    │
  │   next page loads)  │                    │                    │                    │
```

### 5.3 Extension File Structure

```
Tinker-extension/
├── manifest.json                    # MV3 manifest (Chrome) / manifest.json (Firefox)
├── background.js                    # Service Worker — orchestrator
├── content.js                       # Content Script — DOM scanner & action executor
├── offscreen.html                   # Hidden document for ML inference
├── offscreen.js                     # ML pipeline (face detection, OCR, redaction)
├── popup.html                       # Extension popup UI
├── popup.js                         # Popup logic (start/stop agent, settings)
├── sidepanel.html                   # Side panel UI (task input, conversation, logs)
├── sidepanel.js                     # Side panel logic
├── styles/
│   ├── popup.css
│   └── sidepanel.css
├── lib/
│   ├── domExtractor.js              # DOM tree extraction & interactive element mapping
│   ├── piiDetector.js               # Regex + DOM-based PII detection
│   ├── piiPatterns.js               # PII regex patterns (global + India-specific)
│   ├── actionExecutor.js            # Click, type, scroll, navigate dispatchers
│   ├── coordinateMapper.js          # CSS ↔ Device pixel ↔ VLM coordinate transforms
│   └── messageProtocol.js           # Client-server message schemas & serialization
├── models/
│   ├── blaze_face_short_range.tflite    # MediaPipe face detection model (230KB)
│   └── efficientvit_m0.onnx            # EfficientViT for visual understanding (optional)
├── wasm/
│   └── tasks-vision/                    # MediaPipe WASM runtime files
├── vendor/
│   └── tesseract/                       # Tesseract.js worker & core WASM
├── icons/
│   ├── icon-16.png
│   ├── icon-48.png
│   └── icon-128.png
└── _locales/                            # i18n support
```

---

## 6. Detailed Component Design

### 6.1 Content Script — DOM Scanner & Action Executor

The content script is the bridge between the web page and the extension. It has two primary responsibilities:

#### 6.1.1 DOM Tree Extraction

The content script walks the visible DOM and extracts a **compact interactive element tree** — a structured representation of all clickable, typeable, and navigable elements with their bounding boxes, labels, and states.

**Algorithm:**
1. Query all interactive elements: `a[href]`, `button`, `input`, `select`, `textarea`, `[role="button"]`, `[role="link"]`, `[contenteditable="true"]`, `[tabindex]`.
2. For each element, compute visibility:
   - CSS visibility checks (`display`, `visibility`, `opacity`).
   - Geometry check (`getBoundingClientRect()` must have `width > 3` and `height > 3`).
   - Viewport intersection check (element must be within the visible viewport).
   - Occlusion check via `document.elementFromPoint()` (element must not be hidden behind a modal/overlay).
3. Extract the **accessible name** using priority: `aria-label` → `innerText` → `placeholder` → `title` → `alt`.
4. Tag each element with a temporary `data-agent-id` attribute for later action targeting.
5. Return the array as a compact serialized format:

```
[PAGE TITLE: Income Tax Filing - e-Filing Portal]
[VIEWPORT: 1280x800 | SCROLL: 0/2100]
[INTERACTIVE ELEMENTS]
[1] input:text "PAN Number" [x:120, y:240, w:300, h:40] value="ABCDE1234F"
[2] input:text "Full Name" [x:120, y:300, w:300, h:40] value="Rajesh Kumar"
[3] input:password "Password" [x:120, y:360, w:300, h:40] value="••••••••"
[4] button "Login" [x:120, y:420, w:300, h:45]
[5] a "Forgot Password?" [x:280, y:470, w:120, h:20]
```

This compact format reduces token usage by **60–80%** compared to sending raw HTML.

#### 6.1.2 Sensitive Field Detection (DOM-Level)

Before any screenshot is even taken, the content script identifies sensitive form fields using DOM metadata:

| Detection Method | Target | Examples |
|-----------------|--------|----------|
| `input[type="password"]` | Password fields | Login forms, registration forms |
| `input[autocomplete="cc-number"]` | Credit card numbers | Checkout pages |
| `input[autocomplete="cc-csc"]` | CVV/CVC codes | Payment forms |
| `input[autocomplete="cc-exp"]` | Card expiry dates | Payment forms |
| `input[autocomplete="tel"]` | Phone numbers | Contact forms |
| `input[autocomplete="email"]` | Email addresses | Registration forms |
| `input[autocomplete="bday"]` | Birthdays | Profile forms |
| Regex on `name`/`id`/`placeholder` attributes | Any sensitive field | `/password\|ssn\|aadhaar\|pan\|credit.?card\|cvv\|pin\|token\|api.?key/i` |
| Regex on visible `TextNode` content | On-screen PII text | Aadhaar numbers, PAN cards, phone numbers, emails |

**India-Specific PII Patterns (Critical for ISRO evaluation):**

| PII Type | Regex Pattern | Description |
|----------|--------------|-------------|
| Aadhaar Number | `\b[2-9]\d{3}\s?\d{4}\s?\d{4}\b` | 12 digits, cannot start with 0 or 1 |
| PAN Card | `\b[A-Z]{5}[0-9]{4}[A-Z]{1}\b` | 5 letters + 4 digits + 1 letter |
| Indian Phone | `\b(?:(?:\+91\|0)?\s?)?[6-9]\d{9}\b` | 10 digits starting with 6-9 |
| Indian Passport | `\b[A-Z]{1}[0-9]{7}\b` | 1 letter + 7 digits |
| Voter ID | `\b[A-Z]{3}[0-9]{7}\b` | 3 letters + 7 digits |

**Global PII Patterns:**

| PII Type | Regex Pattern |
|----------|--------------|
| Email | `[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}` |
| Credit Card | `\b(?:\d[ -]*?){13,16}\b` + Luhn checksum validation |
| US SSN | `\b\d{3}-\d{2}-\d{4}\b` |
| Phone (International) | `(?:\+\|00)[1-9]\d{1,3}[-.\s]?\(?\d{1,4}\)?[-.\s]?\d{1,4}[-.\s]?\d{1,9}` |

#### 6.1.3 DOM Text Node PII Detection with Exact Bounding Boxes

For PII found in visible text (not inside form inputs), we use the **Range API** to get pixel-perfect bounding boxes:

```javascript
function scanTextNodesForPII(rootElement) {
  const walker = document.createTreeWalker(rootElement, NodeFilter.SHOW_TEXT);
  const piiMatches = [];

  while (walker.nextNode()) {
    const textNode = walker.currentNode;
    const text = textNode.textContent;

    for (const [piiType, regex] of Object.entries(PII_REGEX_PATTERNS)) {
      let match;
      regex.lastIndex = 0; // Reset regex state
      while ((match = regex.exec(text)) !== null) {
        const range = document.createRange();
        range.setStart(textNode, match.index);
        range.setEnd(textNode, match.index + match[0].length);
        const rect = range.getBoundingClientRect();

        piiMatches.push({
          type: piiType,
          text: match[0],
          bbox: { x: rect.left, y: rect.top, w: rect.width, h: rect.height }
        });
      }
    }
  }
  return piiMatches;
}
```

#### 6.1.4 Action Execution Engine

When the server returns an action command, the content script executes it. The executor handles **React/Vue/Angular controlled components** properly:

**Click Execution:**
```javascript
function executeClick(element) {
  element.scrollIntoView({ behavior: 'instant', block: 'center' });
  ['mousedown', 'mouseup', 'click'].forEach(eventType => {
    element.dispatchEvent(new MouseEvent(eventType, {
      view: window, bubbles: true, cancelable: true, buttons: 1
    }));
  });
}
```

**Type Execution (React/Vue Compatible):**
```javascript
function executeType(inputElement, text) {
  inputElement.focus();
  // Use prototype setter to bypass React's synthetic event system
  const prototypeValueSetter = Object.getOwnPropertyDescriptor(
    Object.getPrototypeOf(inputElement), 'value'
  )?.set;
  prototypeValueSetter?.call(inputElement, text);
  inputElement.dispatchEvent(new Event('input', { bubbles: true }));
  inputElement.dispatchEvent(new Event('change', { bubbles: true }));
}
```

**Scroll & Navigate:**
```javascript
function executeScroll(direction, amount = 500) {
  window.scrollBy({ top: direction === 'down' ? amount : -amount, behavior: 'smooth' });
}
// Navigation handled by background service worker: chrome.tabs.update(tabId, { url })
```

---

### 6.2 Background Service Worker — Orchestrator

The service worker is the **central coordinator** of the entire agent loop. It manages:

1. **Tab Management:** `chrome.tabs.captureVisibleTab()` for screenshot capture.
2. **WebSocket Client:** Persistent connection to the central AI server.
3. **Offscreen Document Lifecycle:** Creates/destroys the offscreen document for ML inference.
4. **Agent Loop Orchestration:** The Observe → Redact → Send → Execute cycle.
5. **Keep-Alive:** Uses `chrome.runtime.Port` connections and `chrome.alarms` to prevent MV3 service worker from sleeping during active agent tasks.

> [!WARNING]
> **MV3 Critical Limitation:** The background service worker has **NO access to DOM, Canvas, WebGPU, or WebGL**. All ML inference and image processing **must** happen inside the Offscreen Document.

**Keep-Alive Strategy:**
```javascript
// Maintain open port to prevent service worker sleep
let keepAlivePort = null;

function startKeepAlive() {
  keepAlivePort = chrome.runtime.connect({ name: 'keepalive' });
  keepAlivePort.onDisconnect.addListener(() => {
    keepAlivePort = null;
    startKeepAlive(); // Reconnect
  });
}

// Fallback: Chrome Alarms API
chrome.alarms.create('keepAlive', { periodInMinutes: 0.4 }); // Every 24 seconds
chrome.alarms.onAlarm.addListener((alarm) => {
  if (alarm.name === 'keepAlive') { /* Service worker wakes up */ }
});
```

---

### 6.3 Offscreen Document — The ML Powerhouse

The offscreen document is where **all client-side machine learning** happens. It has full access to DOM, Canvas 2D, WebGPU, WebGL, and WebAssembly.

**ML Pipeline Execution Order:**

```
Raw Screenshot (PNG from captureVisibleTab)
           │
           ▼
  ┌─────────────────────────┐
  │  1. Load onto Canvas    │  Convert base64 → HTMLCanvasElement
  └────────────┬────────────┘
               │
               ▼
  ┌─────────────────────────┐
  │  2. Face Detection      │  MediaPipe BlazeFace (230KB, <10ms on GPU)
  │     → Face bounding     │  Returns: [{bbox: {x,y,w,h}, confidence: 0.98}]
  │       boxes             │
  └────────────┬────────────┘
               │
               ▼
  ┌─────────────────────────┐
  │  3. Targeted OCR        │  Tesseract.js — ONLY on <img>/<canvas> regions
  │     (if image regions   │  within the screenshot, NOT on full frame
  │      detected in DOM)   │  Returns: [{text, bbox, confidence}]
  └────────────┬────────────┘
               │
               ▼
  ┌─────────────────────────┐
  │  4. Merge All PII       │  Combine:
  │     Bounding Boxes      │  - DOM-detected PII boxes (from content script)
  │                         │  - Face detection boxes (from step 2)
  │                         │  - OCR-detected PII boxes (from step 3)
  │                         │  Scale all by window.devicePixelRatio
  └────────────┬────────────┘
               │
               ▼
  ┌─────────────────────────┐
  │  5. Canvas Redaction    │  For each PII box:
  │     Compositor          │  - ctx.fillRect() with solid dark color
  │                         │  - ctx.strokeRect() with highlight border
  │                         │  - ctx.fillText("[REDACTED: TYPE]") label
  └────────────┬────────────┘
               │
               ▼
  ┌─────────────────────────┐
  │  6. WebP Compression    │  canvas.toBlob('image/webp', 0.85)
  │     & Output            │  → Sanitized binary + Redaction manifest JSON
  └─────────────────────────┘
```

> [!TIP]
> **Why Targeted OCR, Not Full-Screen OCR?**
> Running Tesseract.js across an entire 1080p screenshot takes **1.5–3.5 seconds** and spikes CPU to 100%. Since **95% of webpage text lives in the DOM** (and is already captured by the content script's text node scanner), we only run OCR on embedded image regions (`<img>`, `<canvas>`, uploaded file previews) where text is rendered as pixels and not accessible via DOM. This keeps OCR latency under **200ms**.

**Performance Budget for Client-Side Pipeline:**

| Step | Model/Technique | Expected Latency | Memory |
|------|----------------|-------------------|--------|
| Screenshot capture | `captureVisibleTab` | ~5 ms | ~3 MB (PNG) |
| DOM extraction | Content script tree walker | <10 ms | <1 MB |
| DOM PII scan | Regex + Range API | <5 ms | <1 MB |
| Face detection | MediaPipe BlazeFace (WebGPU) | 5–12 ms | ~15 MB |
| Targeted OCR | Tesseract.js (WASM Worker) | 50–200 ms (only if images found) | ~25 MB |
| Canvas redaction | 2D Canvas fillRect/fillText | <2 ms | ~3 MB |
| WebP compression | Canvas.toBlob | ~10 ms | ~1 MB |
| **Total Client Processing** | — | **<30 ms typical, <250 ms worst-case** | **~45 MB peak** |

---

## 7. Privacy & Redaction Pipeline

### 7.1 The Dual-Tier Architecture

Our privacy pipeline uses a **two-tier approach** that maximizes both speed and coverage:

```
                    Web Page State
                         │
             ┌───────────┴──────────────────────────────────┐
             │                                              │
             ▼                                              ▼
    ┌─────────────────────┐                      ┌────────────────────────┐
    │  TIER 1: DOM SCAN   │                      │  TIER 2: VISION SCAN  │
    │  (<5ms, 95% recall) │                      │  (<200ms, covers      │
    │                     │                      │   visual-only PII)    │
    │  • input[type=pwd]  │                      │                       │
    │  • autocomplete     │                      │  • MediaPipe Face     │
    │  • Regex on text    │                      │    Detection          │
    │    nodes (Aadhaar,  │                      │  • Tesseract.js OCR   │
    │    PAN, email, etc) │                      │    on <img>/<canvas>  │
    │  • Name/id/class    │                      │  • bert-base-NER      │
    │    attribute scan   │                      │    (optional, for     │
    │                     │                      │     unstructured      │
    │  Returns: Exact     │                      │     person names)     │
    │  DOM bounding boxes │                      │                       │
    └─────────┬───────────┘                      └──────────┬─────────────┘
              │                                             │
              └─────────────────┬───────────────────────────┘
                                │
                                ▼
                 ┌───────────────────────────┐
                 │  MERGE & DEDUPLICATE      │
                 │  All PII Bounding Boxes   │
                 │  Scale by devicePixelRatio│
                 └─────────────┬─────────────┘
                               │
                               ▼
                 ┌───────────────────────────┐
                 │  CANVAS SEMANTIC REDACT   │
                 │  Solid mask + typed label │
                 │  [REDACTED: PASSWORD]     │
                 │  [REDACTED: AADHAAR]      │
                 │  [REDACTED: FACE]         │
                 └─────────────┬─────────────┘
                               │
                               ▼
                    Sanitized Image + Manifest
```

### 7.2 Why Solid Masking, NOT Blurring

> [!CAUTION]
> **Never use Gaussian Blur or Pixelation for PII redaction.** Multiple published research papers (Unredacter, Depix, DeepMosaics) have demonstrated that blurred text and pixelated digits can be **reconstructed using deep learning deconvolution models**. Only irreversible solid-fill masking (`ctx.fillRect`) guarantees zero information leakage.

### 7.3 Semantic Masking — Why It's Critical

Instead of just painting a blank black box, we draw a **color-coded bounding box with an explicit type label**:

```
┌──────────────────────────────────────────────┐
│  Login Form                                  │
│                                              │
│  Username: [rajesh.kumar@email.com]          │
│  Password: ┌──────────────────────────────┐  │
│            │ [REDACTED: PASSWORD]          │  │
│            └──────────────────────────────┘  │
│                                              │
│  Aadhaar:  ┌──────────────────────────────┐  │
│            │ [REDACTED: AADHAAR]           │  │
│            └──────────────────────────────┘  │
│                                              │
│       [ Login ]    [ Forgot Password? ]      │
└──────────────────────────────────────────────┘
```

**Why this matters for the server VLM:** The VLM retains full **situational awareness** of the page layout. It knows there's a password field at position (x, y) and an Aadhaar field below it — it just can't see the actual values. This allows the VLM to generate precise actions like *"Click the Login button below the [REDACTED: PASSWORD] field"* without ever needing the sensitive data.

### 7.4 Redaction Manifest Schema

Every redacted region is tracked in a JSON manifest that accompanies the sanitized image:

```json
{
  "redactions": [
    {
      "redaction_id": "REDACTED_FACE_1",
      "type": "FACE",
      "source": "mediapipe_blazeface",
      "confidence": 0.97,
      "bbox": { "x": 540, "y": 120, "w": 80, "h": 80 },
      "method": "solid_fill"
    },
    {
      "redaction_id": "REDACTED_AADHAAR_1",
      "type": "AADHAAR",
      "source": "dom_regex",
      "confidence": 1.0,
      "bbox": { "x": 200, "y": 310, "w": 180, "h": 30 },
      "method": "solid_fill_labeled"
    },
    {
      "redaction_id": "REDACTED_PASSWORD_1",
      "type": "PASSWORD",
      "source": "dom_input_type",
      "confidence": 1.0,
      "bbox": { "x": 200, "y": 360, "w": 300, "h": 40 },
      "method": "solid_fill_labeled"
    }
  ],
  "total_pii_detected": 3,
  "processing_time_ms": 22
}
```

### 7.5 DOM Pre-Masking (Before Screenshot)

For maximum safety, we temporarily mask sensitive DOM text **before** taking the screenshot:

```javascript
function preMaskSensitiveDOM() {
  const maskedElements = [];

  // 1. Replace password field values visually
  document.querySelectorAll('input[type="password"]').forEach(el => {
    const original = el.value;
    maskedElements.push({ element: el, originalValue: original, type: 'password' });
    // Password fields already show dots — no action needed
  });

  // 2. Mask text nodes containing PII
  // (Temporarily replace Aadhaar "1234 5678 9012" with "XXXX XXXX XXXX")
  const textPII = scanTextNodesForPII(document.body);
  textPII.forEach(match => {
    const node = match.textNode;
    const original = node.textContent;
    maskedElements.push({ node, originalText: original, type: 'text' });
    node.textContent = original.replace(match.regex, '[REDACTED]');
  });

  return maskedElements; // Return for restoration after screenshot
}

function restoreMaskedDOM(maskedElements) {
  maskedElements.forEach(item => {
    if (item.type === 'text') {
      item.node.textContent = item.originalText;
    }
  });
}
```

---

## 8. Server-Side AI Engine

### 8.1 VLM Selection & Justification

We evaluated 6 open-weights VLMs for server-side deployment:

| Model | Params | Architecture | GUI Grounding | Latency (A100) | Memory | Verdict |
|-------|--------|-------------|---------------|-----------------|--------|---------|
| **Qwen2.5-VL-7B** | 7B | Dynamic-res ViT + LLM | Normalized [0,1000] coords | ~400ms | ~16GB FP16 | ✅ **Primary Choice** |
| **UI-TARS-7B** | 7B | Native GUI agent | `click(x,y)`, `type(text)` | ~350ms | ~16GB FP16 | ✅ **Secondary Choice** |
| InternVL 2.5-8B | 8B | Tiled ViT + LLM | Bounding box JSON | ~450ms | ~18GB FP16 | Good alternative |
| MiniCPM-V 2.6 | 8B | Efficient tiled ViT | Coordinate output | ~300ms | ~16GB FP16 | Lightweight fallback |
| CogAgent-18B | 18B | Dual-resolution ViT | High-res GUI grounding | ~800ms | ~36GB FP16 | Too large for hackathon |
| Fuyu-8B | 8B | Decoder-only (no ViT) | Direct patch projection | ~350ms | ~16GB FP16 | Limited grounding quality |

**Primary: Qwen2.5-VL-7B-Instruct**
- Dynamic resolution — handles native aspect ratios without distorting UI elements.
- Outputs grounding coordinates in `[0, 1000]` normalized format — easy to convert to client pixels.
- Excellent OCR and document understanding (reads text in redacted screenshots accurately).
- Fully open-weights under Apache 2.0 license.
- Deployable on a single RTX 4090 (24GB) or A100 (40/80GB).

**Secondary: UI-TARS-7B**
- Purpose-built GUI agent — natively outputs executable actions (`click(x,y)`, `type(text)`, `scroll()`).
- Incorporates System-2 reasoning ("think before act") for complex multi-step workflows.
- Trained on millions of GUI interaction trajectories.

### 8.2 Server System Prompt

The server VLM receives a carefully crafted system prompt that makes it **aware of the redaction scheme**:

```markdown
You are Tinker, an autonomous Web Navigation Agent. You assist users by analyzing 
sanitized screenshots of their browser and executing UI actions.

## Privacy Awareness
The screenshot you receive has been pre-processed by a client-side privacy filter:
- Sensitive elements (passwords, credit cards, government IDs, personal information) 
  are masked with solid dark rectangles labeled [REDACTED: TYPE].
- Human faces are covered with [REDACTED: FACE] labels.
- These redacted regions represent REAL, EXISTING elements on the page.
- When you need to interact with a redacted element (e.g., clicking a password field), 
  reference its center coordinates. The client will handle the actual data.
- NEVER attempt to guess, reconstruct, or hallucinate the content under a redaction.

## Interactive Element Reference
You will also receive a structured list of interactive DOM elements with their IDs, 
types, labels, and bounding boxes. Use the element `target_id` to precisely identify 
which element to interact with.

## Output Format
Respond with a single JSON object:
{
  "thought": "Brief reasoning about what you observe and what action to take next",
  "action": "click" | "type" | "scroll" | "navigate" | "wait" | "finish" | "fail",
  "target_id": <integer ID from interactive element list, or null>,
  "coordinates": { "x": <0-1000>, "y": <0-1000> },
  "text": "text to type (only for 'type' action)",
  "direction": "up" | "down" (only for 'scroll' action),
  "url": "https://..." (only for 'navigate' action),
  "summary": "result summary (only for 'finish' action)"
}
```

### 8.3 Server Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                        SERVER APPLICATION                            │
│                                                                      │
│  ┌────────────────────┐    ┌──────────────────────┐                  │
│  │  FastAPI App        │    │  Session Manager     │                  │
│  │  (uvicorn)          │    │                      │                  │
│  │                     │    │  - Per-client state   │                  │
│  │  /ws  → WebSocket   │───►│  - Task history       │                  │
│  │  /api → REST (health│    │  - Multi-step context │                  │
│  │         check, etc) │    │  - Conversation turns  │                  │
│  └─────────┬──────────┘    └──────────┬───────────┘                  │
│            │                          │                               │
│            ▼                          ▼                               │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │  VLM Inference Engine                                        │    │
│  │                                                              │    │
│  │  vLLM Server (or SGLang)                                     │    │
│  │  ┌──────────────────────────────────────────────────────┐    │    │
│  │  │  Qwen2.5-VL-7B-Instruct (FP16 / AWQ-INT4)          │    │    │
│  │  │                                                      │    │    │
│  │  │  • PagedAttention → Efficient KV cache management    │    │    │
│  │  │  • Continuous Batching → Multiple clients served     │    │    │
│  │  │  • Tensor Parallelism (multi-GPU) → Scalability      │    │    │
│  │  └──────────────────────────────────────────────────────┘    │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  Deployment: Docker Compose                                          │
│  GPU: NVIDIA RTX 4090 / A100 / H100                                 │
│  CUDA: 12.x + cuDNN 9.x                                             │
└──────────────────────────────────────────────────────────────────────┘
```

### 8.4 Server API Design

```python
# server/main.py (FastAPI)

from fastapi import FastAPI, WebSocket
import json, base64
from PIL import Image
from io import BytesIO

app = FastAPI(title="Tinker Server")

@app.websocket("/ws/{session_id}")
async def agent_websocket(websocket: WebSocket, session_id: str):
    await websocket.accept()
    session = SessionManager.get_or_create(session_id)

    try:
        while True:
            # Receive sanitized observation from client
            raw_data = await websocket.receive_bytes()
            observation = parse_observation(raw_data)

            # Build VLM prompt with redaction-aware system prompt
            vlm_prompt = build_vlm_prompt(
                system_prompt=REDACTION_AWARE_SYSTEM_PROMPT,
                sanitized_image=observation.image,
                dom_tree=observation.interactive_dom,
                redaction_manifest=observation.redaction_manifest,
                task_instruction=session.current_task,
                history=session.action_history[-5:]  # Last 5 steps for context
            )

            # Run VLM inference via vLLM
            action_response = await vlm_engine.generate(vlm_prompt)
            action_json = parse_action_response(action_response)

            # Track in session
            session.add_step(observation, action_json)

            # Send action command back to client
            await websocket.send_json(action_json)

    except WebSocketDisconnect:
        SessionManager.cleanup(session_id)
```

---

## 9. User Interface & Interaction Model

### 9.1 Interaction Paradigm

The agent operates through a **browser side panel** — a dedicated panel that slides in from the right side of the browser window. This is the primary interaction surface.

**Why Side Panel (not popup, not separate app, not CLI)?**
- **Persistent visibility:** Unlike popups (which close on click-away), the side panel stays open alongside the web page.
- **Non-intrusive:** Does not overlay the web content; occupies a dedicated panel space.
- **Native browser integration:** Chrome's Side Panel API (`chrome.sidePanel`) is the recommended pattern for extensions that need persistent UI.
- **Real-time feedback:** Users can watch the agent's reasoning and actions in real-time while seeing the web page respond.

### 9.2 UI Layout

```
┌────────────────────────────────────────────┬──────────────────────────────┐
│                                            │     🛡️ Tinker Agent       │
│                                            │     ─────────────────────   │
│                                            │                             │
│                                            │  🟢 Connected | Privacy: ON │
│                                            │                             │
│         USER'S WEB PAGE                    │  ┌───────────────────────┐  │
│         (Normal browsing)                  │  │ What would you like   │  │
│                                            │  │ me to help with?      │  │
│                                            │  │                       │  │
│                                            │  │ ▸ Fill out this form  │  │
│                                            │  │ ▸ Navigate to...      │  │
│                                            │  │ ▸ Find information    │  │
│                                            │  │                       │  │
│                                            │  │ [Type your task...]   │  │
│                                            │  │ [  📎   🎙️   ➤  ]    │  │
│                                            │  └───────────────────────┘  │
│                                            │                             │
│                                            │  📋 Activity Log            │
│                                            │  ──────────────────         │
│                                            │  ✅ DOM scanned (12 elems) │
│                                            │  ✅ 3 PII regions detected │
│                                            │  ✅ Screenshot sanitized   │
│                                            │  ⏳ Waiting for server...  │
│                                            │  ✅ Action: Click "Submit" │
│                                            │                             │
│                                            │  🔒 Privacy Report          │
│                                            │  ──────────────────         │
│                                            │  Redacted: 2 passwords,    │
│                                            │  1 Aadhaar, 1 face         │
│                                            │  Data sent: 0 PII items    │
│                                            │                             │
│                                            │  [⏸ Pause] [⏹ Stop] [⚙]   │
└────────────────────────────────────────────┴──────────────────────────────┘
```

### 9.3 UI Components

| Component | Purpose | Technology |
|-----------|---------|-----------|
| **Task Input Bar** | User types natural language instructions | HTML `<textarea>` + submit button |
| **Conversation Thread** | Shows agent's reasoning, planned actions, confirmations | Scrollable message list |
| **Activity Log** | Real-time pipeline status (DOM scan, PII detection, redaction, server round-trip) | Auto-scrolling log with status icons |
| **Privacy Report** | Summary of what was redacted in the current step | Collapsible panel with redaction counts |
| **Visual Overlay** | Highlights elements being interacted with on the web page | Content script injects temporary CSS highlights |
| **Settings Panel** | Configure privacy sensitivity, server URL, model selection | Accessible via gear icon |
| **Confirmation Modal** | For high-risk actions (form submission, payment, navigation to new domain) | Modal overlay on the side panel |

### 9.4 User Interaction Flow

1. **User opens the side panel** by clicking the Tinker extension icon.
2. **User types a task** in natural language: *"Help me fill this scholarship application form"*.
3. **Agent begins the Observe-Redact-Reason-Act loop:**
   - Side panel shows real-time activity log entries.
   - Privacy report updates with each detected PII item.
   - Web page shows brief visual highlights on elements being analyzed.
4. **Before executing high-risk actions** (submitting a form, making a payment, navigating away), the agent **pauses and asks for user confirmation** via the side panel.
5. **User can pause, resume, or stop** the agent at any time using the control buttons.
6. **On task completion**, the agent shows a summary of what was accomplished.

### 9.5 Visual Feedback on Web Page (Content Script Overlays)

When the agent is analyzing or interacting with an element, the content script injects temporary visual feedback:

```css
/* Injected by content script */
[data-Tinker-highlight="analyzing"] {
  outline: 2px dashed #3b82f6 !important;  /* Blue dashed outline */
  outline-offset: 2px !important;
  transition: outline 0.2s ease !important;
}

[data-Tinker-highlight="acting"] {
  outline: 3px solid #22c55e !important;   /* Green solid outline */
  outline-offset: 2px !important;
  animation: Tinker-pulse 0.5s ease-in-out !important;
}

[data-Tinker-highlight="redacted"] {
  outline: 2px solid #ef4444 !important;   /* Red outline for redacted regions */
  outline-offset: 1px !important;
}

@keyframes Tinker-pulse {
  0%, 100% { outline-width: 3px; }
  50% { outline-width: 5px; }
}
```

---

## 10. Data Flow & Communication Protocol

### 10.1 Protocol Choice: WebSocket

We use **WebSocket (wss://)** for client-server communication:

| Criterion | WebSocket ✅ | REST + SSE ❌ |
|-----------|-------------|--------------|
| Roundtrip latency | **1–5 ms** (persistent TCP tunnel) | 50–200 ms per HTTP handshake |
| Duplex communication | Full duplex | Unidirectional SSE |
| Binary transmission | Native binary frames (raw WebP) | Base64 string overhead (+33%) |
| Connection recovery | Heartbeat ping/pong + auto-reconnect | Naturally stateless |
| Fit for agent loop | **Ideal** — continuous conversational loop | Suitable only for single actions |

### 10.2 Message Schemas

#### Client → Server: Sanitized Observation

```json
{
  "protocol_version": "1.0",
  "message_type": "OBSERVATION",
  "session_id": "sess_98a7df6b-2341",
  "step_index": 4,
  "timestamp": 1725884210512,
  "task_instruction": "Fill the scholarship application form",
  "page_context": {
    "url": "https://scholarship.gov.in/apply",
    "title": "Scholarship Application - National Portal",
    "viewport": { "width": 1280, "height": 800 },
    "scroll_offset": { "x": 0, "y": 450 },
    "device_pixel_ratio": 2.0
  },
  "sanitized_image": "<binary WebP sent as separate binary frame>",
  "redaction_manifest": [
    {
      "redaction_id": "REDACTED_FACE_1",
      "type": "FACE",
      "source": "mediapipe_blazeface",
      "confidence": 0.97,
      "bbox": [540, 120, 80, 80]
    },
    {
      "redaction_id": "REDACTED_AADHAAR_1",
      "type": "AADHAAR",
      "source": "dom_regex",
      "confidence": 1.0,
      "bbox": [200, 310, 180, 30]
    }
  ],
  "interactive_dom": [
    {
      "id": 1, "tag": "input", "type": "text", "role": "textbox",
      "name": "Applicant Name", "value": "[REDACTED_NAME_1]",
      "bbox": { "x": 120, "y": 240, "w": 300, "h": 40 },
      "disabled": false
    },
    {
      "id": 2, "tag": "input", "type": "text", "role": "textbox",
      "name": "Aadhaar Number", "value": "[REDACTED_AADHAAR_1]",
      "bbox": { "x": 120, "y": 310, "w": 300, "h": 40 },
      "disabled": false
    },
    {
      "id": 3, "tag": "button", "role": "button",
      "name": "Submit Application",
      "bbox": { "x": 120, "y": 420, "w": 300, "h": 45 },
      "disabled": false
    }
  ]
}
```

#### Server → Client: Action Command

```json
{
  "protocol_version": "1.0",
  "message_type": "ACTION",
  "session_id": "sess_98a7df6b-2341",
  "step_index": 4,
  "thought": "The form fields appear filled. The submit button is ready. Clicking it to finalize.",
  "action": {
    "type": "CLICK",
    "target_id": 3,
    "coordinates": { "x": 270, "y": 442 }
  },
  "requires_confirmation": true,
  "status": "RUNNING"
}
```

#### Supported Action Types

| Action | Fields | Description |
|--------|--------|-------------|
| `CLICK` | `target_id`, `coordinates` | Click an interactive element |
| `TYPE` | `target_id`, `text`, `clear_first` | Type text into an input field |
| `SCROLL` | `direction` (`up`/`down`), `amount` | Scroll the page |
| `NAVIGATE` | `url` | Navigate to a different URL |
| `WAIT` | `duration_ms` | Wait for page to load/update |
| `FINISH` | `summary` | Task completed successfully |
| `FAIL` | `error_message` | Task cannot be completed |

### 10.3 Coordinate System & Mapping

A crucial aspect of the system is the coordinate transformation pipeline:

```
  DOM CSS Pixels          Screenshot Canvas Pixels       VLM Normalized [0-1000]
  (Content Script)         (Offscreen Document)            (Server)
       │                         │                            │
       │  × devicePixelRatio     │                            │
       ├────────────────────────►│                            │
       │                         │  ÷ (canvas / 1000)         │
       │                         ├───────────────────────────►│
       │                         │                            │
       │                         │  × (canvas / 1000)         │
       │                         │◄────────────────────────────│
       │  ÷ devicePixelRatio     │                            │
       │◄────────────────────────│                            │
```

**CSS → Canvas:** `canvas_x = css_x × window.devicePixelRatio`

**Canvas → VLM:** `vlm_x = (canvas_x / canvas_width) × 1000`

**VLM → CSS:** `css_x = (vlm_x / 1000) × viewport_width`

---

## 11. Evaluation Strategy (Mapped to Hackathon Metrics)

| # | Metric | Weight | Our Strategy | Expected Score |
|---|--------|--------|-------------|----------------|
| **1** | Accuracy of visual context from screen | **25%** | **Hybrid DOM + Vision:** Send both Set-of-Marks annotated WebP screenshot AND compact structured DOM tree. Server VLM cross-verifies visual positions against exact DOM bounds. High-DPI capture via `captureVisibleTab` scaled by `devicePixelRatio` ensures pixel-perfect fidelity. | **High** — dual-signal approach eliminates single-modality failures |
| **2** | Recall and precision for PII detection | **20%** | **Dual-Tier Detection:** DOM-level inspection guarantees **100% recall** on HTML form inputs (password, CC, email). Regex patterns cover Aadhaar, PAN, Voter ID. MediaPipe face detection + Tesseract.js handle visual pixel-level PII. India-specific patterns included. | **High** — DOM scan alone achieves near-perfect recall on structured data |
| **3** | Precision of redaction | **20%** | **Exact bounding box masking** using `getBoundingClientRect()` + Range API for text nodes. Solid-fill masking (NOT blur) with type-preserving semantic labels. 15% safety margin padding around detected regions. | **High** — pixel-exact redaction with zero information leakage |
| **4** | Client-side resource utilization | **20%** | **WebGPU-accelerated** inference: BlazeFace <10ms, DOM scan <5ms, no full-screen OCR (targeted only). Total client processing <30ms typical. Peak memory ~45MB. CPU stays under 5% via async Web Workers. | **High** — lightweight pipeline avoids heavy computation |
| **5** | End-to-end latency | **15%** | **Persistent WebSocket** (1-5ms latency) + **WebP compression** (10x smaller than PNG). Client processing <30ms + Network ~50ms + Server VLM ~400ms = **total <500ms per step**. | **Medium-High** — VLM inference is the bottleneck |

### 11.1 Benchmarking Plan

We will create a **benchmark suite** of 15–20 test scenarios covering:

| Category | Test Scenarios |
|----------|---------------|
| **Government Portals** | DigiLocker login, income tax e-filing, passport application, Aadhaar update |
| **Banking/Finance** | Online banking login, UPI payment, credit card application, loan EMI calculator |
| **E-Commerce** | Product checkout, address entry, payment page with CC details |
| **Healthcare** | Patient registration, medical history form, prescription upload |
| **Social Media** | Profile with face photos, bio with phone/email, direct messages |
| **Enterprise** | Internal dashboards, API key management, employee directory |

For each scenario, we measure:
1. **PII Detection Recall:** # PII items correctly detected / # total PII items present
2. **PII Detection Precision:** # PII items correctly detected / # total items flagged as PII
3. **Redaction Completeness:** Visual inspection — is any PII pixel visible in the sanitized image?
4. **Client Processing Time:** Measured via `performance.now()` across the pipeline.
5. **End-to-End Latency:** From user action trigger to agent action execution completion.
6. **Memory Usage:** `performance.memory.usedJSHeapSize` (Chrome) during inference.

---

## 12. SDLC — Development Phases & Timeline

### Phase 0: Environment Setup & Boilerplate (Day 1 — 4 hours)

| Task | Details | Deliverable |
|------|---------|-------------|
| Repository setup | Git repo, monorepo structure (`/client`, `/server`, `/shared`) | Clean repo with README |
| Client boilerplate | Vite + TypeScript project, MV3 manifest, content script, service worker, offscreen doc stubs | Extension loads in Chrome |
| Server boilerplate | FastAPI project, Docker Compose, health check endpoint | Server runs with `docker-compose up` |
| Model downloads | Download BlazeFace `.tflite`, ONNX models, Tesseract WASM to `/client/models` | Models bundled with extension |
| VLM setup | Pull Qwen2.5-VL-7B via HuggingFace, configure vLLM serving | `vllm serve` running with model loaded |

---

### Phase 1: Core Perception Pipeline (Day 1–2 — 12 hours)

| Task | Priority | Details |
|------|----------|---------|
| **DOM Extractor** | P0 | Content script that walks DOM, extracts interactive elements, computes bounding boxes, generates compact serialized tree |
| **Screenshot Capture** | P0 | Service worker `captureVisibleTab` → base64 PNG → forward to offscreen document |
| **Offscreen ML Setup** | P0 | Offscreen document initialization with MediaPipe WASM runtime, ONNX Runtime Web session warm-up |
| **Canvas Pipeline** | P0 | Receive raw image → load onto HTMLCanvasElement → prepare for ML processing |
| **Face Detection** | P0 | MediaPipe BlazeFace running in offscreen document, returning face bounding boxes |

**Milestone:** Extension captures screen, extracts DOM, detects faces — all locally.

---

### Phase 2: Privacy & Redaction Pipeline (Day 2–3 — 10 hours)

| Task | Priority | Details |
|------|----------|---------|
| **DOM PII Scanner** | P0 | Detect sensitive form fields via `type`, `autocomplete`, `name`/`id` regex. Extract text node PII via Range API bounding boxes. |
| **India-Specific PII Patterns** | P0 | Aadhaar, PAN, Indian phone, passport, voter ID regex patterns |
| **Global PII Patterns** | P0 | Email, credit card (+ Luhn validation), SSN, international phone |
| **Canvas Redaction Compositor** | P0 | Merge all PII boxes (DOM + face + OCR), scale by devicePixelRatio, draw semantic masks with typed labels |
| **Targeted OCR** | P1 | Tesseract.js integration — run only on `<img>` and `<canvas>` element regions |
| **WebP Compression** | P1 | Compress redacted canvas to WebP format for efficient transmission |
| **DOM Pre-Masking** | P1 | Temporarily mask DOM text nodes before screenshot capture for double safety |

**Milestone:** Extension produces fully sanitized screenshots with zero PII leakage.

---

### Phase 3: Server-Side VLM Integration (Day 3–4 — 10 hours)

| Task | Priority | Details |
|------|----------|---------|
| **vLLM Model Serving** | P0 | Deploy Qwen2.5-VL-7B with vLLM, configure GPU memory, test with sample images |
| **WebSocket Server** | P0 | FastAPI WebSocket endpoint that receives sanitized observations and returns action commands |
| **VLM Prompt Engineering** | P0 | Craft redaction-aware system prompt, test with various sanitized screenshot types |
| **Action Parser** | P0 | Parse VLM JSON output into typed action commands |
| **Session Manager** | P1 | Track per-client task state, action history (last N steps), multi-step planning context |
| **Error Handling** | P1 | Handle VLM hallucinations, invalid actions, timeouts, reconnection logic |

**Milestone:** Server correctly interprets sanitized screenshots and returns valid action commands.

---

### Phase 4: Client-Side Action Execution (Day 4 — 6 hours)

| Task | Priority | Details |
|------|----------|---------|
| **Action Dispatcher** | P0 | Route action commands to appropriate executor (click, type, scroll, navigate) |
| **Click Executor** | P0 | Reliable click with `mousedown` → `mouseup` → `click` event sequence |
| **Type Executor** | P0 | React/Vue/Angular-compatible typing via prototype property setter |
| **Scroll Executor** | P0 | Smooth scrolling with configurable direction and amount |
| **Navigate Executor** | P0 | `chrome.tabs.update` for URL navigation |
| **Coordinate Transformer** | P0 | VLM [0-1000] → CSS pixels transformation with devicePixelRatio correction |
| **Confirmation System** | P1 | Pause before high-risk actions (form submit, payment, navigation), ask user via side panel |

**Milestone:** Full agent loop works end-to-end: Observe → Redact → Send → Reason → Act.

---

### Phase 5: User Interface & Polish (Day 4–5 — 8 hours)

| Task | Priority | Details |
|------|----------|---------|
| **Side Panel UI** | P0 | Task input, conversation thread, activity log, privacy report |
| **Visual Overlays** | P1 | Content script CSS highlights on analyzed/acted elements |
| **Settings Panel** | P1 | Server URL config, privacy sensitivity level, model selection |
| **Error States** | P1 | Connection lost, server unavailable, WebGPU not supported — graceful degradation |
| **Firefox Compatibility** | P2 | Adapt MV3 manifest for Firefox, test WebGPU availability |

**Milestone:** Polished, user-friendly interface ready for demo.

---

### Phase 6: Testing, Benchmarking & Demo (Day 5–6 — 8 hours)

| Task | Priority | Details |
|------|----------|---------|
| **End-to-End Test Suite** | P0 | 15–20 test scenarios across government, banking, e-commerce, healthcare |
| **PII Detection Benchmark** | P0 | Measure recall/precision across all PII types |
| **Latency Profiling** | P0 | Measure each pipeline stage, identify bottlenecks |
| **Resource Utilization** | P0 | Monitor CPU, GPU, memory during agent operation |
| **Demo Scenario** | P0 | Prepare polished end-to-end demo: user asks agent to fill a government form → agent detects PII → redacts → server reasons → agent fills form |
| **Documentation** | P1 | Setup guide, architecture docs, API documentation |

**Milestone:** Production-ready prototype with benchmarks and demo-ready presentation.

---

### Development Timeline (Gantt Chart)

```
Day 1 ──── Day 2 ──── Day 3 ──── Day 4 ──── Day 5 ──── Day 6
 │           │           │           │           │           │
 ├─ Phase 0 ─┤           │           │           │           │
 │ (Setup)   │           │           │           │           │
 │           │           │           │           │           │
 ├─── Phase 1 (Core Perception) ────┤           │           │
 │                                   │           │           │
 │           ├─── Phase 2 (Privacy Pipeline) ───┤           │
 │           │                                   │           │
 │           │           ├─── Phase 3 (Server VLM) ────────┤
 │           │           │                       │           │
 │           │           │           ├── Phase 4 ─┤           │
 │           │           │           │ (Actions)  │           │
 │           │           │           │           │           │
 │           │           │           ├──── Phase 5 (UI) ────┤
 │           │           │           │                       │
 │           │           │           │           ├── Phase 6 ─┤
 │           │           │           │           │ (Test/Demo)│
```

---

## 13. Agility & Adaptability Mechanisms

### 13.1 Website-Agnostic Design

The agent is **not hardcoded for any specific website**. It works on ANY webpage because:

1. **DOM Extraction is universal:** `querySelectorAll` with interactive element selectors works on every website.
2. **PII patterns are language-independent:** Regex patterns for Aadhaar, PAN, email, phone work across any website.
3. **Visual perception is layout-agnostic:** The ViT/face detection models process raw pixels — they don't care about CSS frameworks or design systems.
4. **Server VLM generalizes:** Qwen2.5-VL is trained on millions of web screenshots — it understands any UI layout.

### 13.2 Graceful Degradation

```
 Is WebGPU available?
      │
      ├── YES → Use WebGPU for all ML inference (fastest path)
      │
      └── NO → Is WebGL available?
               │
               ├── YES → Use WebGL backend for MediaPipe, WASM for ONNX
               │
               └── NO → Use pure WebAssembly SIMD (CPU-only, slower but works)
                         │
                         └── Is WASM SIMD supported?
                              │
                              ├── YES → Multi-threaded WASM with SIMD
                              │
                              └── NO → Single-threaded WASM (slowest fallback)
```

Every ML component has a fallback chain:
- **Face Detection:** MediaPipe WebGPU → MediaPipe WebGL → MediaPipe WASM → Skip (DOM-only PII)
- **OCR:** Tesseract.js WASM Worker → Tesseract.js main thread → Skip (DOM text only)
- **Visual Understanding:** EfficientViT WebGPU → EfficientViT WASM → Skip (DOM tree only)

### 13.3 Adaptive PII Detection

The PII detection engine is **extensible** — new patterns can be added without code changes:

```javascript
// piiPatterns.js — Add new patterns at runtime
const customPatterns = {
  // Healthcare
  MEDICAL_RECORD_NUMBER: /\bMRN[-:]?\s?\d{6,10}\b/gi,
  
  // Custom enterprise patterns
  EMPLOYEE_ID: /\bEMP[-]?\d{5,8}\b/gi,
  
  // International
  UK_NHS_NUMBER: /\b\d{3}\s?\d{3}\s?\d{4}\b/g,
  CANADIAN_SIN: /\b\d{3}[-\s]?\d{3}[-\s]?\d{3}\b/g,
};

// Users can add custom patterns via settings
function addCustomPattern(name, regexString) {
  PII_REGEX_PATTERNS[name] = new RegExp(regexString, 'gi');
}
```

### 13.4 Multi-VLM Failover

If the primary VLM is unavailable or underperforming, the server automatically falls back:

```
Qwen2.5-VL-7B (Primary)
    │
    ├── Available → Use it
    │
    └── Unavailable/Error → UI-TARS-7B (Secondary)
                                │
                                ├── Available → Use it
                                │
                                └── Unavailable → MiniCPM-V 2.6 (Lightweight Fallback)
                                                    │
                                                    └── Unavailable → Return error to client
```

### 13.5 Adaptive Agent Behavior

The agent adapts its behavior based on context:

| Context | Adaptation |
|---------|------------|
| **Slow network** | Increase WebP compression, reduce image resolution, send DOM-only without screenshot |
| **No GPU on client** | Skip vision models, rely entirely on DOM extraction + regex PII detection |
| **Complex SPA (React/Angular)** | Use prototype setter for typing, wait for re-renders after actions |
| **Bot-protected websites** | Switch from synthetic events to CDP `Input.dispatchMouseEvent` if `chrome.debugger` permission is available |
| **Mobile viewport** | Adjust coordinate mapping for touch events, smaller element targets |
| **Multiple tabs/windows** | Track active tab, pause agent when user switches away |

---

## 14. Risk Analysis & Mitigation

| Risk | Severity | Likelihood | Mitigation |
|------|----------|------------|------------|
| **WebGPU not available on target browser** | High | Low (Chrome 113+, Firefox 141+ support it) | Multi-backend fallback: WebGPU → WebGL → WASM SIMD → CPU |
| **VLM hallucinates incorrect actions** | High | Medium | Human-in-the-loop confirmation for high-risk actions; action validation (check target element exists before executing) |
| **PII detection misses a sensitive field** | High | Low | Dual-tier detection (DOM + vision) provides redundancy; users can manually flag missed fields |
| **MV3 service worker sleeps during agent loop** | Medium | High | Keep-alive via `chrome.runtime.Port` + `chrome.alarms`; offscreen document maintains long-running inference |
| **Offscreen document killed by browser** | Medium | Medium | Auto-recreate offscreen document on demand; cache model sessions for fast restart |
| **Server VLM inference too slow** | Medium | Medium | Use AWQ/GPTQ INT4 quantization; enable continuous batching; fallback to smaller MiniCPM-V model |
| **Large model files bloat extension size** | Medium | Low | BlazeFace is only 230KB; Tesseract WASM lazy-loaded on demand; ONNX models loaded from CDN |
| **Website blocks synthetic events** | Low | Low | Escalate to CDP `Input.dispatchMouseEvent` for OS-level event emulation |
| **Prompt injection via malicious website** | High | Low | Privacy filter runs before any data reaches the server; DOM content is sanitized; server prompt has safety guardrails |
| **Cross-origin iframe content** | Medium | Medium | Set `all_frames: true` in content script manifest; DOM extractor recursively scans accessible frames |

---

## 15. Testing Strategy

### 15.1 Unit Tests

| Component | Test Framework | Key Tests |
|-----------|---------------|-----------|
| DOM Extractor | Vitest + JSDOM | Extract interactive elements from mock HTML; verify visibility filtering; verify bounding box computation |
| PII Detector | Vitest | Verify all regex patterns match expected PII strings; verify Luhn checksum; test India-specific patterns |
| Coordinate Mapper | Vitest | CSS → Canvas → VLM → CSS roundtrip accuracy; devicePixelRatio scaling |
| Action Executor | Vitest + JSDOM | Verify click dispatches correct event sequence; verify React-compatible typing |
| Message Protocol | Vitest | Serialization/deserialization of observation and action messages |
| Server VLM Prompt | Pytest | Verify prompt template generation; verify action JSON parsing |

### 15.2 Integration Tests

| Test | Description |
|------|-------------|
| **Full Pipeline** | Load a test HTML page → capture → extract DOM → detect PII → redact → send to server → receive action → execute |
| **WebSocket Round-trip** | Client sends observation → server processes → client receives action → verify action correctness |
| **PII Redaction Completeness** | Render a page with known PII → capture sanitized screenshot → OCR the sanitized image → verify NO PII text remains |
| **Multi-Step Task** | Agent completes a 5-step form-filling task end-to-end |

### 15.3 Manual Testing Scenarios

| Scenario | What to Verify |
|----------|---------------|
| **Government portal login** | Aadhaar, PAN detected and redacted; login button clicked correctly |
| **E-commerce checkout** | CC number, CVV, address redacted; correct form field targeting |
| **Social media profile** | Face photos detected and redacted; profile details masked |
| **Banking dashboard** | Account numbers, balances masked; navigation works |
| **Dynamic SPA (React)** | Form inputs properly filled despite React's controlled component system |

---

## 16. Deployment & Demo Plan

### 16.1 Server Deployment

```yaml
# docker-compose.yml
version: '3.8'
services:
  vllm:
    image: vllm/vllm-openai:latest
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    volumes:
      - ./models:/models
    command: >
      --model Qwen/Qwen2.5-VL-7B-Instruct
      --tensor-parallel-size 1
      --max-model-len 4096
      --gpu-memory-utilization 0.9
    ports:
      - "8000:8000"

  api:
    build: ./server
    ports:
      - "8080:8080"
    environment:
      - VLLM_URL=http://vllm:8000
      - WEBSOCKET_PORT=8080
    depends_on:
      - vllm
```

### 16.2 Extension Installation

1. Build extension: `cd client && npm run build`
2. Open `chrome://extensions`
3. Enable Developer Mode
4. Click "Load Unpacked" → select `client/dist/`
5. Extension icon appears in toolbar

### 16.3 Demo Script (5-Minute Presentation)

1. **Introduction (30s):** Problem statement — AI agents can't assist with sensitive tasks because they require sharing screen data with servers.
2. **Architecture Overview (60s):** Show the split-agent architecture diagram. Highlight the privacy guarantee.
3. **Live Demo — Government Form (120s):**
   - Navigate to a government form (mock or real).
   - Activate Tinker agent.
   - Show the side panel activity log in real-time.
   - Show the privacy report: "2 Aadhaar numbers redacted, 1 PAN card redacted, 1 face redacted."
   - Show the sanitized screenshot (with visible `[REDACTED: ...]` labels).
   - Watch the agent fill the non-sensitive parts of the form automatically.
4. **Technical Deep Dive (60s):** Show WebGPU inference performance in DevTools. Show the network tab — only sanitized images transmitted.
5. **Benchmarks (30s):** PII detection recall/precision numbers. Latency breakdown.

---

## 17. Future Scope & Extensions

| Extension | Description | Complexity |
|-----------|-------------|------------|
| **Multi-tab agent** | Agent operates across multiple browser tabs for complex workflows | Medium |
| **Voice commands** | Whisper.js running locally for speech-to-task conversion | Medium |
| **Task recording & replay** | Record agent workflows for repetitive tasks | Medium |
| **Custom PII policies** | Let enterprises define their own sensitive data categories | Low |
| **On-device SLM** | Run SmolVLM-500M or Moondream2 entirely in-browser, eliminating server dependency for simple tasks | High |
| **Mobile browser support** | Adapt for Chrome Android / Safari iOS (WebGPU support expanding) | High |
| **Federated learning** | Improve PII detection models using federated learning across users without sharing data | Very High |
| **Accessibility agent** | Help visually impaired users navigate complex websites | High |
| **Multi-language PII** | Detect PII in Hindi, Tamil, Telugu, Bengali scripts using multilingual NER | Medium |

---

## 18. Appendix — Key References & Papers

### 18.1 Academic Papers & Research

| Paper | Relevance |
|-------|-----------|
| **GUIGuard (2026)** — Privacy-preserving GUI agent framework | Direct inspiration for our 3-stage pipeline (Recognize → Protect → Execute) |
| **WebPII & WebRedact (2026)** — 44K annotated web UIs for PII detection | Benchmark methodology for evaluating PII detection precision/recall |
| **MaskClaw (2026)** — Edge-side privacy arbitrator | Type-preserving visual placeholders concept → our `[REDACTED: TYPE]` labels |
| **SeeAct** — Visual grounding for web agents | Two-stage action generation + grounding architecture |
| **Browser Use** — Autonomous browser agent | Set-of-Marks annotation + CDP action execution |

### 18.2 Key Libraries & Documentation

| Library | Documentation URL |
|---------|-------------------|
| ONNX Runtime Web | https://onnxruntime.ai/docs/get-started/with-javascript/web.html |
| Transformers.js v3 | https://huggingface.co/docs/transformers.js |
| MediaPipe Tasks Vision | https://ai.google.dev/edge/mediapipe/solutions/vision/face_detector/web_js |
| Tesseract.js | https://github.com/naptha/tesseract.js |
| Qwen2.5-VL | https://huggingface.co/Qwen/Qwen2.5-VL-7B-Instruct |
| UI-TARS | https://huggingface.co/bytedance-research/UI-TARS-7B-DPO |
| vLLM | https://docs.vllm.ai |
| Chrome Extension MV3 | https://developer.chrome.com/docs/extensions/develop |
| WebGPU API | https://developer.mozilla.org/en-US/docs/Web/API/WebGPU_API |
| Chrome Side Panel API | https://developer.chrome.com/docs/extensions/reference/api/sidePanel |
| Chrome Offscreen API | https://developer.chrome.com/docs/extensions/reference/api/offscreen |

### 18.3 Model Performance Quick Reference

| Model | Size | Browser Latency (WebGPU) | Purpose |
|-------|------|-------------------------|---------|
| MediaPipe BlazeFace | 230 KB | 5–12 ms | Face detection |
| EfficientViT-M0 | 2.3M params | 12–22 ms | Visual feature extraction |
| MobileViT-XXS | 1.3M params | 18–32 ms | Visual understanding (alternative) |
| Xenova/bert-base-NER | ~45 MB (Q8) | 15–30 ms per sentence | Named entity recognition |
| Tesseract.js (WASM) | ~15 MB worker | 50–200 ms (targeted) | OCR on image regions |
| Qwen2.5-VL-7B (Server) | 7B params | ~400 ms (A100) | Visual reasoning + action grounding |

---

> [!NOTE]
> This document is a living artifact. It will be updated as the project evolves, new requirements emerge, or better technologies become available. The architecture is designed to be **modular** — any component (face detector, VLM, PII patterns) can be swapped without affecting the rest of the system.