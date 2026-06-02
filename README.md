**MicroUI Protocol & Toolkit – Design Document (Draft v0.2)**  
*Status: Second Draft – Incorporated semantic color roles*  
*Author: collaborative design session*

---

## 1. Executive Summary

MicroUI is a **resolution-independent, remote-display GUI protocol** designed to run interactive applications across a continuum of devices: from 8-bit microcomputers (1 MHz, 64 KB RAM) to modern desktops with GPU-accelerated toolkits. The application logic (the **Server**) is completely decoupled from the renderer (the **Client**). Both communicate over a lightweight binary protocol.

The protocol is **retained-mode**, **cell-addressable**, and **optimistic**. Clients maintain a replica of the widget tree and handle all transient interaction (text entry, checkbox toggles, cursor movement) locally. The server sees only committed user intentions and may issue **vetoes** to correct state.

---

## 2. Goals & Non-Goals

| Goals                                            | Non-Goals                                               |
| :-:                                              | :-:                                                     |
| Sub-100-byte parser on 6502-class CPUs           | Pixel-perfect WYSIWYG across all clients                |
| Unified codebase for GUI and TUI versions        | Built-in rich text, video, or 3D                        |
| Interactive latency \< 16 ms over local serial   | Multi-user collaborative editing (CRDTs)                |
| Deploy on ESP32, C64, POSIX TTY, and FLTK/X11    | Browser/WebGL as a primary target (future work)         |
| Bandwidth-efficient enough for 2400 baud         | Arbitrary overlapping windows with alpha (desktop-only) |


## 3. Architecture
```
┌─────────────────────────────────────────────┐
│            Application Server               │
│   (C++, Python, Rust — any language)        │
│   Owns: canonical scene tree, layout,       │
│          business logic, validation         │
│   Does not know: pixels, fonts, OS, SDL     │
└──────────────┬──────────────────────────────┘
               │ TCP / Unix Socket / RS-232
               │ MicroUI Protocol (binary)
┌──────────────┴──────────────────────────────┐
│              Client Renderer                │
│   Owns: local replica, input state,         │
│          frame buffer, interaction feedback │
│   Does not know: application logic          │
└─────────────────────────────────────────────┘
```
**Key principle:** The server describes *what* is on screen. The client decides *how* to realize it.

---

## 4. The Scene Model

### 4.1 Cell-Based Coordinates
All geometry is measured in **character cells**, not pixels. A node with `w=10, h=1` occupies a logical region ten cells wide and one tall.

*   **FLTK client:** scales cells by its chosen font metrics (e.g., `1 cell = 10×20 px`).
*   **Terminal client:** maps 1:1 to columns/rows.
*   **C64 client:** maps 1:1 to screen RAM cells (40×25).

**Rationale:** Cells are the greatest common denominator between a raster display and a text terminal. Using cells eliminates the need for the server to know DPI, font metrics, or terminal size. It also guarantees that a layout that fits on a C64 fits identically on a desktop GUI, modulo color fidelity.

### 4.2 Node Types (v1)
| ID | Type | Semantics |
|----|------|-----------|
| `0x01` | `WINDOW` | Top-level container; client may add OS chrome or border |
| `0x02` | `CONTAINER` | Invisible layout box; children arranged by `LAYOUT` |
| `0x03` | `LABEL` | Static, non-interactive text |
| `0x04` | `BUTTON` | Action trigger; may show pressed state transiently |
| `0x05` | `INPUT` | Single-line text field; **client-authoritative editing** |
| `0x06` | `CHECKBOX` | Boolean toggle |
| `0x07` | `RADIO` | Mutually exclusive toggle (grouped by `GROUP` ref) |
| `0x08` | `SLIDER` | 0–255 range; client drags locally, commits on release |
| `0x09` | `PROGRESS` | 0–255 read-only indicator |
| `0x0A` | `SEPARATOR` | Visual divider (line or empty space) |

### 4.3 Property System
Properties are identified by a single byte key. Values are typed by the message opcode (see §5), not by an inline type tag.

| Key | Name | Setter | Description |
|-----|------|--------|-------------|
| `0x01` | `GEOMETRY` | `SET_RECT` | Position/size in cells |
| `0x02` | `VISIBLE` | `SET_U8` | 0 = hidden, 1 = visible |
| `0x03` | `ENABLED` | `SET_U8` | 0 = grayed / inactive |
| `0x04` | `FG_ROLE` | `SET_U8` | Foreground semantic color role (0–15) |
| `0x05` | `BG_ROLE` | `SET_U8` | Background semantic color role (0–15) |
| `0x06` | `BORDER` | `SET_U8` | `0=NONE, 1=SINGLE, 2=DOUBLE, 3=THICK` |
| `0x07` | `TEXT` | `SET_STR` | String ID for label/content |
| `0x08` | `VALUE` | `SET_U8` | Scalar: slider/progress position (0–255) |
| `0x09` | `STATE` | `SET_U8` | Bitfield: `b0=checked, b1=pressed, b2=focused` |
| `0x0A` | `LAYOUT` | `SET_U8` | `0=NONE, 1=HBOX, 2=VBOX` |
| `0x0B` | `WEIGHT` | `SET_U8` | Flex grow factor within container |
| `0x0C` | `STYLE` | `SET_U8` | `b0=bold, b1=dim, b2=reverse, b3=underline` |
| `0x0D` | `Z_INDEX` | `SET_U8` | Drawing order override (higher = later) |
| `0x0E` | `GROUP` | `SET_NODE_REF` | For radio mutual exclusion |

---

## 5. Wire Protocol

### 5.1 Framing
Every message is a length-prefixed frame. This allows a parser to skip unknown messages and enables both stream (TCP) and packet (UDP/serial) transports.


Byte 0: payload_length (u8) -- does NOT include the length byte itself Byte 1: message_type (u8) Byte 2..N: payload (N = payload_length bytes)


**Maximum packet size:** 255 payload bytes + 2 header bytes = **257 bytes total**.

**Rationale:** A single byte length is trivial to parse on a 6502 (`LDY $0, LDA ($buf),Y`) and guarantees the frame fits in one 6502 memory page. It also keeps clients from needing dynamic allocation: they can pre-allocate a single 257-byte ring buffer.

### 5.2 Message Reference

#### Control Messages (`0x00`–`0x0F`)
| Type | Dir | Payload | Description |
|------|-----|---------|-------------|
| `0x00` | — | *none* | **NOP** – no-op, useful for keepalive |
| `0x01` | C→S | `version(u8), cols(u8), rows(u8), flags(u8), max_nodes(u8)` | **HELLO** |
| `0x02` | ↔ | *none* | **PING** |
| `0x03` | ↔ | *none* | **PONG** |
| `0x04` | S→C | *none* | **RESET** – client must drop all state |

`HELLO` flags bitfield:
*   `b0` – `COLOR_1BIT`
*   `b1` – `COLOR_4BIT` (client will down-convert)
*   `b2` – `COLOR_RGB565`
*   `b3` – `COLOR_RGB888` (native client palette depth, not wire format)
*   `b4` – `SUPPORTS_IMAGES` (reserved for v2)
*   `b5` – `SUPPORTS_MOUSE`
*   `b6` – `SUPPORTS_AUDIO` (reserved)
*   `b7` – reserved

#### Scene Management (`0x10`–`0x1F`)
| Type | Dir | Payload | Description |
|------|-----|---------|-------------|
| `0x10` | S→C | `node_id(u8), parent_id(u8), type(u8)` | **CREATE** |
| `0x11` | S→C | `node_id(u8)` | **DELETE** |

#### Property Updates (`0x20`–`0x2F`)
| Type | Dir | Payload | Description |
|------|-----|---------|-------------|
| `0x20` | S→C | `node_id(u8), prop(u8), val(u8)` | **SET_U8** |
| `0x21` | S→C | `node_id(u8), prop(u8), val(u16be)` | **SET_U16** |
| `0x22` | S→C | `node_id(u8), x(u8), y(u8), w(u8), h(u8)` | **SET_RECT** |
| `0x23` | S→C | `node_id(u8), prop(u8), str_id(u8)` | **SET_STR** |
| `0x24` | S→C | `node_id(u8), prop(u8), target_id(u8)` | **SET_NODE_REF** |

#### Assets (`0x30`–`0x3F`)
| Type | Dir | Payload | Description |
|------|-----|---------|-------------|
| `0x30` | S→C | `str_id(u8), len(u8), data[...]` | **DEF_STR** – define string in global table |

String table semantics: Strings are referenced by `str_id` throughout the session. The client must retain them until overwritten or until the client is reset. Max string length per `DEF_STR` is 253 bytes.

#### Presentation (`0x40`–`0x4F`)
| Type | Dir | Payload | Description |
|------|-----|---------|-------------|
| `0x40` | S→C | *none* | **FRAME** – End of atomic update batch. Client should present now. |

**FRAME semantics:** The server batches all `CREATE`, `UPDATE`, and `DELETE` messages between two `FRAME` markers. The client must buffer these in a scratch area and apply them atomically upon `FRAME`. This prevents tearing and allows the client to perform a single diff/composite pass.

#### Client Input (`0x70`–`0x7F`)
| Type | Dir | Payload | Description |
|------|-----|---------|-------------|
| `0x70` | C→S | `node_id(u8), key(u8), mod(u8)` | **EVT_KEY** – `node_id=0` means unassigned/global |
| `0x71` | C→S | `node_id(u8), action(u8), cell_x(u8), cell_y(u8)` | **EVT_POINT** – `action`: `0=MOVE, 1=DOWN, 2=UP` |
| `0x72` | C→S | `node_id(u8), state(u8)` | **EVT_TOGGLE** – checkbox/radio changed to `state` |
| `0x73` | C→S | `node_id(u8), val(u8)` | **EVT_COMMIT_IDX** – list/radio/slider committed index/value |
| `0x74` | C→S | `node_id(u8), len(u8), text[...]` | **EVT_COMMIT_STR** – input/text committed |

### 5.3 Color & Palette Negotiation
The canonical wire format for color is **not RGB**. Colors are specified via **semantic roles** (`FG_ROLE`, `BG_ROLE`), each a single byte index from the 16-entry palette defined below.

The client down-converts roles to physical colors according to its own local theme, palette, and capability flags declared in `HELLO`.

#### The 16-Color Semantic Palette (v1)

| Index | Role | Typical Meaning |
|-------|------|---------------|
| `0x0` | `DEFAULT` | Inherit from parent or client theme default |
| `0x1` | `FG` | Primary text / icons |
| `0x2` | `BG` | Primary background / window chrome |
| `0x3` | `ACCENT` | Interactive elements: buttons, focus rings, selected tab |
| `0x4` | `MUTED` | Secondary / placeholder / disabled subtle text |
| `0x5` | `SUCCESS` | Positive feedback, "OK", completion |
| `0x6` | `ERROR` | Danger, invalid input, alerts, "Delete" |
| `0x7` | `WARNING` | Caution, non-fatal issues |
| `0x8` | `INFO` | Neutral emphasis, hyperlinks, hints |
| `0x9` | `BORDER` | Dividers, outlines, inactive scrollbars |
| `0xA` | `HIGHLIGHT` | Selection background, current list item |
| `0xB` | `INVERTED_FG` | Text drawn on top of `ACCENT` or `HIGHLIGHT` |
| `0xC` | `SHADOW` | Drop shadow / depth (rich clients may ignore) |
| `0xD` | `CURSOR` | Caret, block cursor, insertion bar |
| `0xE` | `MARKER` | Badges, notifications, "new" dots |
| `0xF` | `RESERVED` | Future use; clients should map to `ACCENT` |

**Rationale:** A C64 cannot parse or dither 24-bit RGB in real time. A programmer should not need to test contrast on four different physical displays. By sending semantic roles, the server remains agnostic to dark mode, monochrome terminals, and green-phosphor nostalgia. The client owns the final aesthetic.

#### Platform Mappings (Informative)

**FLTK Desktop Client**
Maps roles to configurable `Fl_Color` indices. A theme file (or hardcoded table) defines the RGB for each role. If the user wants a "dark mode," the FLTK client simply loads a different local mapping of the same 16 roles. The server does not know dark mode exists.

**Terminal / ncurses / ANSI Client**
Maps directly to the 16 ANSI colors (or the 256-color cube if available, but the protocol only guarantees 16).

**Commodore 64 / 8-bit**
A single 16-byte lookup table in ROM/RAM maps roles to C64 color RAM nibbles (0–15). A C64 user who prefers a blue-background theme changes *one byte* in the client's `BG` mapping without the server ever knowing.

**ESP32 / LCD Clients**
Map roles to an RGB565 or RGB332 lookup table. The LCD blitter reads the role index, loads the RGB565 word, and fills pixels.

### 5.4 Images (v2 / Out of Scope)
Images are not defined in v1. When needed, they will be sent as opaque binary blobs referenced by hash/ID, using the same length-prefixed framing, with a negotiated capability flag. Clients without image support render a placeholder `[image]`.

---

## 6. Interaction Model: Optimistic UI & Server Veto

### 6.1 State Ownership
To eliminate per-keystroke latency, the protocol distinguishes two classes of state:

| Class | Owner | Sync Direction | Examples |
|-------|-------|----------------|----------|
| **Presentation** | Server | S→C only | Geometry, colors, labels, visibility |
| **Interaction** | Client | C→S notify, S→C veto | Checked, cursor position, edit buffer, hover |

### 6.2 The Veto Pattern
1. The user interacts (clicks, types, toggles).
2. The client **immediately** updates its local replica and redraws.
3. The client sends a notification (`EVT_TOGGLE`, `EVT_COMMIT_STR`, etc.) to the server.
4. The server validates. If the action is illegal, the server sends a corrective `UPDATE` (e.g., `SET_U8` clearing `STATE.b0`).
5. The client applies the correction silently and does **not** generate another event in response.

**Rationale:** On a 2400 baud link, a full round-trip for every keystroke or click would make the UI unusable. By making the client authoritative for transient state, the system feels local. The server veto pattern keeps business logic centralized without requiring complex conflict resolution (there is only one user per client).

### 6.3 Widget-Specific Behavior

| Widget | Client Action | Network Traffic |
|--------|--------------|-----------------|
| **INPUT** | All editing is local (cursor movement, insertion, deletion). Hardware/software cursor managed entirely by client. | `EVT_COMMIT_STR` on Enter/Tab/Blur only. |
| **CHECKBOX** | Toggle `STATE.b0` immediately on click. Re-draw checkmark. | `EVT_TOGGLE` with new state. |
| **RADIO** | Uncheck previous selection in same `GROUP` locally; check new one. | `EVT_TOGGLE` for the clicked node. Server can veto by resetting both. |
| **BUTTON** | Draw pressed state on `DOWN`, released on `UP`. | `EVT_POINT action=UP` if inside button bounds. |
| **SLIDER** | Move thumb with drag. Update local `VALUE`. | `EVT_COMMIT_IDX` on release (throttled to ≤10 Hz during drag if continuous mode desired). |

### 6.4 Focus
Focus is primarily client-local. The client moves focus based on Tab/Shift-Tab or point clicks. The server may hint focus via `STATE.b2`, but the client is not required to obey it while the user is actively editing a field. Focus changes generate **no** network messages unless the focus change causes a commit (e.g., Tab out of an input field).

---

## 7. Client Lifecycle

### 7.1 Startup
1. Client connects transport.
2. Client sends `HELLO` with its capabilities.
3. Server sends the initial scene tree as a batch terminated by `FRAME`.
4. Client enters main loop: wait for transport data or local input.

### 7.2 Memory Model (Static Allocation)
The protocol is designed so that clients never need `malloc`.

*   **Node Table:** Fixed array of `max_nodes` entries (size declared in `HELLO`). On a C64 this might be 16; on an ESP32, 128.
*   **String Table:** Fixed array of `N` string slots of `M` bytes each. On a C64: 32 slots × 32 bytes = 1 KB.
*   **Receive Buffer:** 257 bytes.
*   **Cell Framebuffer:** `cols × rows` cells. On C64 this is literally the hardware screen RAM (`$0400`).

**Rationale:** 8-bit systems have fragmented memory maps and no virtual memory. Dynamic allocation leads to leaks and fragmentation. Fixed tables allow the entire client to reside in a known memory footprint.

### 7.3 Rendering Pass
Upon `FRAME`:
1. **Apply deltas** from the receive buffer into the node/string tables.
2. **Walk the visible tree** (depth-first, respecting `Z_INDEX`).
3. **Composite** into a cell backbuffer (on TUI/8-bit) or issue draw calls (on FLTK).
4. **Diff and present**: The terminal client diffs the new cell grid against the previous one and emits only changed ANSI sequences. The FLTK client calls `damage(FL_DAMAGE_ALL)` or tracks dirty regions.
5. **Clear delta buffer** and await the next `FRAME`.

---

## 8. Platform-Specific Rendering Notes

### 8.1 FLTK Client (C++)
*   Maps 1 cell to a monospace font metric (e.g., 10×20 px).
*   Each node is a lightweight `Fl_Widget` subclass that reads from the replica table in its `draw()`.
*   Uses `fl_rectf()`, `fl_loop()`, `fl_draw()` for all primitives. No native `Fl_Button` required; custom drawing ensures consistency with the terminal aesthetic if desired.
*   Maps the 16 semantic color roles to `Fl_Color` indices via a local theme table.
*   Mouse events divided by cell size to yield cell coordinates for `EVT_POINT`.

### 8.2 Terminal Client (Python / C / ncurses)
*   Maintains a 2D array of `Cell {fg_role, bg_role, char, attr}`.
*   Renders borders with Unicode box-drawing characters (`┌─┐│└┘`) or ASCII fallbacks.
*   Maps semantic roles to ANSI 16-color / 256-color / truecolor escapes based on local terminal capability.
*   Input fields use local terminal line editing or a custom `getch()` loop; the server is unaware until commit.
*   Mouse support via X10/X11 tracking mode (`\e[?1006h`).

### 8.3 ESP32 / Embedded LCD
*   Typically 320×240 TFT. Choose a cell size of 8×16 or 6×8 pixels.
*   Maintain a cell framebuffer in SRAM. Render by blitting glyph tiles from a compiled-in bitmap font.
*   Maps roles to an RGB565 or RGB332 lookup table.
*   Touch coordinates mapped to cells via integer division.
*   WiFi/TCP using lwIP, or serial bridge for lower power.

### 8.4 Commodore 64 / 8-bit
*   **Transport:** RS-232 to a Pi/ESP32 gateway, or an Ethernet cart with raw UDP.
*   **Parser:** 6502 assembly routine reads the ring buffer, dispatches via jump table, writes into a node table at `$0800`.
*   **Rendering:** Direct pokes to Screen RAM (`$0400`) and Color RAM (`$D800`). A `BUTTON` becomes PETSCII box-drawing characters + centered text. A `PROGRESS` becomes a row of filled bars.
*   **Color:** A 16-byte lookup table maps roles to VIC-II color indices (0–15).
*   **Text Input:** Uses the Kernal screen editor or a 40-column inline editor. The network stack is uninvolved until Return is pressed.

---

## 9. Transport Considerations

The protocol is **transport-agnostic**, but v1 assumes an **ordered, reliable byte stream** (TCP, Unix socket, or reliable serial).

*   **No built-in acknowledgements** are required for v1 because `FRAME` acts as an implicit boundary. If the transport is lossy (UDP), a future version will add sequence numbers and `ACK` frames.
*   **Multiple clients:** The server may support multiple simultaneous client connections (e.g., one FLTK, one TTY), but each client receives the same scene stream. There is no per-client customization in v1.
*   **Latency tolerance:** Because clients render locally from a replica, brief network stalls freeze only external state updates; local scrolling, cursor movement, and button presses remain responsive.

---

## 10. Security Considerations

*   **Threat model:** The server is trusted. The client is an output device. However, anyone who can open a connection to the server can inject `EVT_*` messages.
*   **Mitigations:**
    *   Bind to `localhost` or Unix sockets by default.
    *   Place TLS or NaCl `box` encryption outside the protocol (e.g., `stunnel` or WireGuard) if crossing an untrusted network.
    *   The server must enforce rate limits and node caps; a malicious client requesting `max_nodes=255` should not crash the server.
*   **Client safety:** A malicious server can send infinite `CREATE` loops. The client must hard-cap at its `HELLO` `max_nodes` and ignore surplus messages. `DEF_STR` lengths must be checked against the local table bounds.

---

## 11. Performance Budgets

| Target | Code ROM | Data RAM | Typical Latency | Bandwidth |
|--------|----------|----------|-----------------|-----------|
| **C64** | ~3 KB | ~2.5 KB (tables + buffer) | N/A (local) | 2400 baud |
| **ESP32** | ~32 KB | ~16 KB | < 20 ms over WiFi | TCP/IP |
| **POSIX TUI** | ~64 KB binary | ~256 KB | < 1 ms local | TCP |
| **FLTK Desktop** | ~256 KB binary | ~1 MB | < 1 ms local | TCP |

A typical screen update (moving focus between two form fields) is < 20 bytes. A full screen rebuild for a complex dialog is < 2 KB.

---

## 12. Versioning & Extensibility

*   **Protocol version:** `HELLO` carries a single `version` byte (`0x02` for this draft).
*   **Unknown messages:** Clients and servers **must** skip unknown messages using the length prefix. This allows a v3 server to send optional v3 messages to a v2 client without breaking interoperability.
*   **Node types:** Unknown node types should be rendered as a generic rectangle or placeholder by v2 clients.
*   **Property keys:** Unknown property keys on known node types are ignored.
*   **Palette:** New roles may be added to the 16-entry space in future versions; clients map unknown roles to `DEFAULT` or `ACCENT`.

---

## 13. Rationale Appendix (Decisions for Future Agents)

| Decision | Rationale | What not to change without discussion |
|----------|-----------|-------------------------------------|
| **Cells, not pixels** | Only common denominator between TUI, C64 screen RAM, and GUI | Do not add fractional pixel coordinates; this breaks 8-bit clients. |
| **Binary length-prefixed frames** | Parseable in 20 instructions on a 6502; no heap, no JSON scanner | Do not switch to JSON, XML, or Protobuf for the core protocol. |
| **Retained-mode scene graph** | Bandwidth efficiency; single `UPDATE` changes a label without redraw commands | Do not switch to immediate-mode drawing primitives (Cairo/X11 style); this would multiply bandwidth 100×. |
| **Server veto instead of server echo** | Eliminates round-trip latency for typing and clicking | Do not make the server authoritative for cursor position or checkbox state; this reintroduces lag. |
| **Static string table (`DEF_STR`)** | Deduplicates repeated labels; avoids re-transmitting "Cancel" every frame | Do not inline all strings into `SET_STR`; the C64 string table would explode in size. |
| **U8 node IDs** | 256 widgets is generous for an 8-bit UI and saves one byte per message | If upgrading to U16, add an escape in protocol v3 and keep U8 the default. |
| **Single `STATE` bitfield** | Atomic updates; a 6502 can `ORA`/`AND` one byte efficiently | Do not split every state bit into its own message type; this wastes bandwidth. |
| **Semantic color roles, not RGB** | A C64 cannot parse 24-bit RGB in real time; programmers should not need to test contrast on four different displays. Keeps dark mode / theming client-local. | Do not push raw RGB or per-client palette management to the server. |
| **16-entry role palette** | Matches C64 color RAM (4 bits), ANSI 16-color mode, and the smallest useful FLTK colormap. A 6502 lookup table is exactly 16 bytes. | If expanding, reserve an escape code (`0xFF` role) for extended palette IDs in a future message, but keep the base 16 universal. |
| **No images in v1** | Keeps C64/ESP32 clients implementable in a weekend | Add images via capability-negotiated opaque blobs in v3, not by embedding base64 in strings. |

---

## 14. Open Questions

1.  **Textarea / multiline input:** Should v2 include a `TEXTAREA` node, or is a vertical stack of `INPUT` nodes sufficient?
2.  **Scrolling containers:** How does a client with no pixel scroll capability (C64) handle a `CONTAINER` with children outside the parent bounds? Options: clip, paginate, or treat as overflow error.
3.  **UDP vs. TCP for ESP32:** Should we define a lightweight `ACK`/`NACK` scheme so UDP becomes viable for multicast discovery?
4.  **Audio feedback:** Should there be a `BEEP` or `CLICK` message for clients with speakers, or is this client-local only?
5.  **Font metrics:** Should `HELLO` advertise a preferred cell aspect ratio so the server can avoid slender layouts that look poor on square-pixel LCDs?
6.  **Extended palette:** If a v3 server wants to offer true-color theming to capable clients, should it send a `PALETTE_REDEFINE` message (role + RGB), or should the client keep theming entirely offline?

---

**End of Document.**  
*Future iterations should update the wire protocol section with exact packed-struct layouts for each target language, add a state-machine diagram for the client parser, and define the `CONTAINER` layout algorithm precisely.*
