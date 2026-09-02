# `pdf_viewer`: Native PDF Viewer for Zed

A native, high performance PDF viewing engine and GPUI workspace item engineered for Zed.

---

## 0. Architectural Identity & Delivery Model

> **Important**: This is a **native workspace crate**, not an installable WASM extension.  
> As of current Zed versions, Zed's WebAssembly extension API (`zed_extension_api`) contains no custom UI or file preview surfaces. `pdf_viewer` is designed as a native crate registered within the Zed workspace (`crates/pdf_viewer`).

---

## 1. Core Architectural Pillars

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                             PDF_VIEWER CORE PIPELINE                             │
└──────────────────────────────────────────────────────────────────────────────────┘
                                        │
                      [PDF Document on Disk / Memory Buffer]
                                        │
                                        ▼
                  [Thread Safe Document Handle (`document.rs`)]
                                        │
                                        ▼
             [Async Background Worker Pool (`gpui::Task` / Threads)]
                                        │
                        (Zero UI Thread Blocking Rasterization)
                                        │
                                        ▼
                  [Memory Budgeted LRU Cache (`cache.rs`)]
                                        │
                       (Visible Viewport + 1 Page Margin)
                                        │
                                        ▼
                 [Luminosity Tone Mapping Engine (`ui.rs`)]
                                        │
                       (Inverts White/Black; Preserves Images)
                                        │
                                        ▼
                  [Native GPUI Canvas / Bitmap Painter (`view.rs`)]
```

1. **Zero UI Thread Blocking**: PDF parsing and rasterization never execute on GPUI's main render loop. Work is dispatched to background tasks that stream RGBA framebuffers into memory.
2. **Viewport Virtualization & LRU Caching**: Pages are rasterized on demand for the visible viewport plus a 1 page prefetch margin. Stale or passed by in flight render requests are cancelled during fast scrolls to prevent thread pool starvation.
3. **Luminosity Threshold Dark Mode**: Unlike simple blanket RGB inversion (which turns photos and charts into negative ghosts), `pdf_viewer` implements selective tone mapping, remapping near white canvas backgrounds to editor background tones and near black text to theme light text, while preserving saturated color images.
4. **Scroll & Zoom Preserving Live Reload**: Watches the underlying PDF on disk (via file system notifications) and reloads seamlessly without resetting the user's scroll percentage or zoom level.

---

## 2. Scope Boundaries

- **Continuous Scroll & Text Selection**: Supports fluid high frame rate document browsing and visual text selection.
- **Excluded from initial scope**:
  - Interactive PDF form field editing.
- *Rationale*: Delivering an ultra fast, stutter free viewing experience with zero UI locks takes precedence.

---

## 3. Security Posture & Threat Model

- **In Process Parsing**: PDF rasterization is performed via Google's native C++ [Pdfium](https://pdfium.googlesource.com/pdfium/) engine linked in process with the editor.
- **Accepted Risk**: Because Pdfium runs in process without sandboxing (for example, without separate process IPC or WebAssembly memory isolation), malformed or malicious PDFs could theoretically trigger memory vulnerabilities in the underlying C++ library.
- **Mitigation & Future Hardening**: Memory budgets are strictly capped, document handles are isolated behind thread boundaries, and a future sandboxed worker process model is planned for untrusted file browsing.

---

## 4. Repository Layout

```
crates/pdf_viewer/
├── Cargo.toml               # Native crate manifest
├── README.md                # Architecture & operational guide
├── tests/
│   └── integration_tests.rs # End to end integration tests
└── src/
    ├── pdf_viewer.rs        # Crate entrypoint & exports
    ├── cache.rs             # Memory budgeted generic PageLruCache
    ├── document.rs          # Thread safe PdfDocument handle
    ├── pdfium.rs            # Pdfium engine binding & coordinate normalization
    ├── rasterizer.rs        # Async background rendering pipeline
    ├── settings.rs          # PDF viewer configuration options
    ├── ui.rs                # Toolbar state & action models
    ├── view.rs              # GPUI Render & Workspace Item implementation
    └── watcher.rs           # Live reload file watcher & scroll state
```

---

## 5. Local Development & Testing

Run unit and integration tests across the caching and rasterization engines:

cargo test -p pdf_viewer
