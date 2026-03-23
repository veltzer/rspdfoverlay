# PDF Overlay Tool — Design Document

**March 2026 · Rust · Desktop Application**

---

## 1. Overview

A desktop application for overlaying content onto existing PDF documents. The tool provides a graphical interface that renders PDF pages and allows the user to interactively place text, dates, signatures, and images at precise locations. On save, overlays are written directly into the PDF file.

The primary use case is adding signatures, dates, and annotations to documents that require manual mark-up—contracts, forms, official paperwork—without relying on external PDF editors or web-based services.

## 2. Goals and Non-Goals

### 2.1 Goals

- Render PDF pages faithfully in a GUI window
- Allow interactive placement of text overlays (signatures, dates, freeform text)
- Allow placement of image overlays (scanned signatures, stamps)
- Write overlays back into the PDF without corrupting the original content
- Support zoom and pan for precise placement
- Keep the application fully local—no network calls, no cloud dependencies
- Minimal external runtime dependencies—single binary distribution where possible

### 2.2 Non-Goals

- Full PDF editing (reflow text, delete pages, merge documents)
- PDF form field filling (AcroForms / XFA)
- OCR or text extraction
- Handling encrypted or DRM-protected PDFs
- Mobile or web deployment

## 3. Architecture

The application is composed of three subsystems that communicate through a shared overlay data model:

### 3.1 PDF Rendering Engine

**Crate:** pdfium-render (Rust bindings to Google's PDFium library)

PDFium is the PDF engine used in Chromium. It handles the full PDF specification reliably, including complex content streams, embedded fonts, transparency, and color spaces. The Rust bindings provide a safe API for rendering individual pages to RGBA bitmaps at arbitrary DPI.

The application bundles the PDFium shared library (libpdfium.so / pdfium.dll) alongside the binary. This adds roughly 20–30 MB to the distribution but eliminates any system-level PDF dependency.

### 3.2 GUI Layer

**Framework:** egui / eframe (immediate-mode Rust GUI)

egui is an immediate-mode GUI library that is well-suited to tool-style applications. It provides built-in support for texture rendering, mouse/keyboard input handling, and scrollable/zoomable canvases. The rendering pipeline is:

- Load a PDF page bitmap from pdfium-render into an egui texture
- Display the texture in a scrollable, zoomable canvas panel
- Capture mouse clicks and drags to position overlay objects
- Render overlay previews as egui draw commands on top of the page texture
- Provide a sidebar/toolbar for overlay type selection, text input, and font controls

egui runs on all major desktop platforms (Linux, macOS, Windows) via its eframe backend, which uses wgpu or glow for GPU-accelerated rendering.

### 3.3 PDF Writing Engine

**Crate:** lopdf (low-level PDF manipulation)

lopdf provides direct access to PDF internal structures: the object tree, page dictionaries, content streams, and resource dictionaries. When the user saves, the application:

- Opens the original PDF via lopdf
- For each overlay, injects the appropriate PDF operators into the target page's content stream
- For text overlays: BT (begin text), Tf (set font), Td (position), Tj (draw string), ET (end text)
- For image overlays: embeds the image as a PDF XObject, registers it in the page resources, and references it via the Do operator
- Saves the modified PDF to the output path

## 4. Data Model

The core data structure is a list of overlay objects, each associated with a specific page:

| Field | Type | Description |
|-------|------|-------------|
| overlay_type | Enum | Text, Date, Image, or Signature |
| page_index | usize | Zero-based index of the target PDF page |
| position | (f32, f32) | Position in PDF points (origin: bottom-left of page) |
| content | String | Text content (for Text/Date types) |
| image_data | Option\<Vec\<u8\>\> | Raw image bytes (for Image/Signature types) |
| font_size | f32 | Font size in PDF points (default: 12.0) |
| font_family | String | Font name (default: Helvetica) |
| color | (u8, u8, u8) | RGB color tuple (default: black) |
| rotation | f32 | Rotation in degrees (default: 0.0) |
| scale | f32 | Scale factor for images (default: 1.0) |

## 5. Coordinate System

Coordinate mapping between screen space and PDF space is a critical detail. PDF uses a coordinate system with the origin at the bottom-left of the page, measured in points (1 point = 1/72 inch). The GUI canvas has its origin at the top-left, measured in pixels.

The conversion requires two values: the render scale (DPI used when rasterizing the page) and the current zoom level. The formulas are:

**Screen → PDF X:** `pdf_x = screen_x / (render_dpi / 72.0) / zoom`

**Screen → PDF Y:** `pdf_y = page_height_pts - (screen_y / (render_dpi / 72.0) / zoom)`

The Y-axis inversion accounts for the different coordinate origins. All overlay positions are stored in PDF points so they remain resolution-independent.

## 6. Crate Evaluation

The following crates were evaluated for each subsystem:

### 6.1 PDF Rendering

| Crate | Approach | Verdict |
|-------|----------|---------|
| pdfium-render | Bindings to Google PDFium (Chromium engine) | **Selected** — best spec coverage, reliable rendering |
| mupdf-rs | Bindings to MuPDF (Sumatra PDF engine) | Strong alternative; harder to build and link |
| poppler (via bindings) | Bindings to Poppler (Linux PDF library) | Less Rust-native; better for CLI tools |

### 6.2 GUI Framework

| Crate | Paradigm | Verdict |
|-------|----------|---------|
| egui / eframe | Immediate-mode, lightweight | **Selected** — ideal for canvas-based tool UIs |
| iced | Elm-like retained mode | More boilerplate for interactive canvas work |
| slint | Declarative / compiled UI | Polished but heavier ceremony for this use case |
| tauri + web frontend | Rust backend + HTML/JS frontend | Fallback option — see section 8 |

### 6.3 PDF Writing

| Crate | Approach | Verdict |
|-------|----------|---------|
| lopdf | Low-level PDF object manipulation | **Selected** — best for surgical page modification |
| printpdf | High-level PDF creation API | Better for creating new PDFs than modifying existing ones |
| pdf-rs | PDF parsing / reading | Read-only; not suitable for writing overlays |

## 7. Complexity Assessment

| Tier | Scope | Effort Estimate |
|------|-------|-----------------|
| Easy | Text overlays (signatures as text, dates, annotations) on well-formed PDFs | 1 weekend |
| Medium | Image overlays (scanned signatures, stamps) with XObject embedding | 1–2 weeks additional |
| Hard | Encrypted PDFs, compressed content streams, Unicode/Hebrew font subsetting, AcroForms | Significant — out of scope for MVP |

The GUI integration (egui canvas with zoom/pan, overlay placement, coordinate mapping) is estimated at 2–3 weeks of evening and weekend work. This is the primary schedule driver for the MVP.

## 8. Alternative: Tauri + Web Frontend

If the pure-Rust GUI path encounters significant friction (egui canvas limitations, texture management complexity, or input handling edge cases), a fallback architecture is available:

- **Frontend:** React or vanilla HTML/JS with PDF.js for rendering. PDF.js provides mature pan/zoom/page navigation out of the box, and overlay placement becomes standard DOM event handling.
- **Backend:** Tauri (Rust) handles file I/O, PDF writing via lopdf, and system integration.
- **Trade-off:** Loses the single-binary pure-Rust distribution model but roughly halves the development time for the GUI layer.

This path is recommended only if the egui approach proves unworkable. The pure-Rust stack is preferred for distribution simplicity and performance.

## 9. MVP Feature Set

- Open a PDF file via file dialog
- Render all pages in a scrollable, zoomable canvas
- Click to place a text overlay at the cursor position
- Edit overlay text inline (signature name, date, freeform)
- Move overlays by dragging
- Delete overlays
- Save modified PDF to a new file (non-destructive)
- Basic font size and color selection

## 10. Future Enhancements

- Image overlay support (PNG/JPEG signatures and stamps)
- Drawing / freehand annotation mode
- Overlay templates (save and reuse overlay layouts across documents)
- Batch processing (apply the same overlay set to multiple PDFs)
- Undo/redo stack
- Dark mode / theme support
- Keyboard shortcuts for common operations
- PDF page rotation support
