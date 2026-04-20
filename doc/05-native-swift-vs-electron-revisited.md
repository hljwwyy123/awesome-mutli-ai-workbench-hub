---
title: "Building Native macOS in 2026: SwiftUI vs Electron Revisited"
slug: "native-swift-vs-electron-revisited"
target_query: "SwiftUI vs Electron"
secondary_queries:
  - "native macOS vs Electron app"
  - "menu bar app Swift vs Electron"
  - "why not Electron for Mac apps"
canonical_url: "https://agentbell.dev/blog/native-swift-vs-electron-revisited"
author: "AgentBell Team"
date: "2026-05-02"
tags: ["macos", "swift", "swiftui", "electron", "devtools"]
disclosure: "AgentBell is a native Swift/SwiftUI app. This article states my biases upfront and still tries to be fair to Electron."
---

# Building Native macOS in 2026: SwiftUI vs Electron Revisited

If you are shipping a **menu bar utility** or a **small desktop companion** in 2026, someone will ask: *Why not Electron? Ship faster, hire from the web, one codebase for Windows.*

That is not wrong for every product. For **AgentBell**, I chose **Swift + SwiftUI** anyway. This post is the decision record: what native buys you, what it costs, and where Electron still wins — with numbers that matched our build, not generic benchmarks.

> **Disclosure**: [AgentBell](https://agentbell.dev) is native macOS. I am not neutral; I am **explicit**.

---

## The question we actually answered

Not "which framework is morally better" but:

> *Can we deliver a credible **menu-bar-first**, **low-memory**, **system-integrated** experience without fighting the platform?*

For that shape of app, native Swift tends to win. For a **cross-platform SaaS shell** with a huge web UI, Electron often wins.

---

## What we measured (order-of-magnitude)

These are **realistic indie scale** numbers, not lab benchmarks. Your Electron app may be smaller if you obsess over bundle diet; your Swift app may grow if you embed heavy assets.

| Metric | Our Swift/SwiftUI menu bar app | Typical Electron wrapper (same *class* of utility) |
|---|---|---|
| **Installed footprint** | Tens of MB (app + embedded assets vary) | Often **150–400 MB+** with Chromium + Node runtime |
| **Idle RAM** | Low tens to low hundreds of MB depending on Live2D/Metal | Often **hundreds of MB** at idle |
| **Cold start** | Feels instant for menu bar utilities | Noticeable warm-up on older Macs |
| **Menu bar / NSStatusItem** | First-class, documented | Achievable via `Tray`, still "foreign" to AppKit |
| **Code signing / notarization** | Standard Xcode pipeline | Also standard, but different sharp edges |
| **Accessibility / Focus / Notifications** | Deep integration if you do the work | Possible, always bridging |

**The headline**: For a **resident menu bar agent**, users *feel* memory and wake-from-sleep behavior even if they never open Activity Monitor. Native makes "barely there" believable.

---

## Why SwiftUI in 2026 (and not AppKit-only)

**SwiftUI** is not perfect. It still has sharp corners around:

- Complex `NSViewController` bridging
- Some animation and layout bugs across OS versions
- Documentation lag for edge cases

But for **new** Mac-only utilities, SwiftUI is the default path:

1. **Declarative UI** matches how small teams iterate — less boilerplate than classic AppKit.
2. **Live previews** (when they work) shorten UI cycles.
3. **Apple's direction** is SwiftUI-first; fighting that buys you little unless you need a specific AppKit control.

We still drop to **AppKit / Metal** where SwiftUI is the wrong layer (e.g. **Live2D** rendering, some window behaviors). Hybrid is normal.

---

## Where Electron is genuinely stronger

### 1. Team and hiring

If your team is **all TypeScript/React**, Electron is the shortest path. Swift expertise is a real constraint.

### 2. Cross-platform revenue

One codebase for **macOS + Windows + Linux** can outweigh memory cost. Many dev tools **must** be on Windows — native Swift does not solve that alone.

### 3. Rapid UI iteration

If the product **is** a complex web app stuffed in a shell (Notion-like, Figma-like), Electron matches the architecture.

### 4. Ecosystem

npm, React components, and web debugging tools are unmatched for UI velocity.

**Rule**: If your **primary surface** is a large interactive web UI and macOS is one of several targets, Electron (or Tauri — see below) is rational.

---

## Tauri and other middle grounds

**Tauri** (Rust + system WebView) shrinks bundle and RAM versus bundling full Chromium. For many teams it is the **best compromise** in 2026.

We did not pick Tauri because:

- AgentBell is **not** a web-first UI — the heavy lifting is **menu bar + optional Metal/Live2D**.
- We wanted **zero** webview tax for the core experience.

If you are **web UI + need smaller than Electron**, Tauri deserves a serious spike before you default to Electron.

---

## macOS-specific reasons native won for us

### Menu bar identity

`NSStatusItem` + `NSMenu` + activation policy (`.accessory` vs `.regular`) are **core** to how users perceive AgentBell. Doing this in Electron works; doing it *natively* is fewer layers of "pretend we are a Mac app."

### Sandboxing and IDE integration

We ship **DMG + Hardened Runtime** (not Mac App Store) because **App Sandbox** makes writing to IDE config paths (`~/.cursor/`, `~/.claude/`, etc.) painful for users. That decision is **orthogonal** to Electron vs Swift — but native tooling integrates cleanly with **Security-Scoped Bookmarks**, **Xcode signing**, and **Sparkle** patterns the Mac community already documents.

### Metal and Live2D

Our **video-slice** and **Live2D** paths touch **AVFoundation**, **Metal**, and vendor SDKs. A web stack would still bridge to native for serious rendering — at which point you are maintaining **two** worlds.

---

## SwiftUI pain points we actually hit

Honest list — if you are considering native, budget for these:

1. **Version matrix**: Users on macOS N vs N-1; test both.
2. **Window lifecycle**: Menu bar apps that also show floating companion windows need careful focus and `NSPanel` behavior.
3. **Distribution**: Notarization, stapling, and Developer ID are **learnable** but not zero — same as any shipped Mac binary.
4. **CI**: `xcodebuild` on GitHub Actions is slower than `npm run build` — optimize caches.

None of these are deal-breakers; they are **real costs**.

---

## Decision framework (copy this)

| If your product is… | Lean toward… |
|---|---|
| Menu bar / lightweight utility / deep OS hooks | **Native Swift** (or Obj-C if legacy) |
| Large SPA, team is web-first, need Win+Mac fast | **Electron** or **Tauri** |
| Need smallest cross-platform shell with web UI | **Tauri** |
| Game-like rendering, Metal, low latency | **Native** + maybe game engine — not Electron |

---

## TL;DR

- **Electron** is the rational default for **web-first, cross-platform** products.
- **Swift/SwiftUI** is the rational default for **Mac-native, menu-bar-heavy, low-footprint** tools — especially when you touch Metal, hooks, and system integration.
- **Tauri** sits between them for many teams.
- We picked native for AgentBell because the product is **ambient infrastructure**, not a big web surface — and users should not pay **hundreds of megabytes** for a bell in the menu bar.

---

*If you care about *why* menu bar state matters for AI workflows, see [Best way to track multiple AI coding agents](./01-track-multiple-ai-coding-agents.md).*
