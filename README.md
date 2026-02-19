# BeidPrep Pro — OpenRadioss Pre-Processor

> **Bridging the gap between commercial FEA solvers and OpenRadioss.**

BeidPrep is a, standalone pre-processing application designed to author, edit, and validate `.rad` input decks for the [OpenRadioss](https://github.com/OpenRadioss/OpenRadioss) explicit solver. It provides an integrated 3D environment, an AI-driven card generation engine, and a structured template library — enabling engineers to move from LS-DYNA workflows to OpenRadioss without sacrificing productivity.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [AI-Powered Card System](#2-ai-powered-card-system)
3. [Selection Manager](#3-selection-manager)
4. [Supported Card Library — Technical Reference](#4-supported-card-library--technical-reference)
5. [GUI & 3D Viewer](#5-gui--3d-viewer)
6. [Installation](#6-installation)
7. [License](#7-license)

---

## 1. Project Overview

BeidPrep targets the creation of a solver decks to be run using the OpenRadioss solver of high-energy, highly non-linear physics simulations including:

- **Structural Impact & Crash** — automotive, aerospace, and defense applications
- **Smoothed-Particle Hydrodynamics (SPH)** — fluid-structure interaction, projectile penetration, fragmentation
- **Explosive / Detonation Simulations** — blast loading, shaped charges, mine-blast events
- **Composite Material Modelling** — progressive failure, fabric materials, orthotropic shells
- **Hyper-elastic and Visco-elastic Bodies** — rubber, foam, biological tissues

The application reads and writes the Radioss Starter deck format (`.rad`), strictly enforcing the **10-character column layout at 100 characters per line** required by the solver. Every field is formatted with column precision — integers, floats with scientific notation, and character flags — eliminating hand-formatting errors that can silently corrupt a run.

BeidPrep also includes an **LS-DYNA mesh importer** (`.k` format), converting keyword-based LS-DYNA decks into equivalent OpenRadioss mesh cards and giving teams a direct migration path from commercial to open-source solvers. 

---

## 2. AI-Powered Card System

### Default Template Library

BeidPrep ships with a curated template library covering the most common Radioss cards across all analysis domains (see [Section 4](#4-supported-card-library--technical-reference) for the full list). These templates drive the UI: field editors, column positions, data types, conditional visibility, and repeat-row logic are all read from the JSON schema — making the editor self-describing and immediately correct.

### Not Limited to Default Cards

The built-in library is a starting point, not a ceiling. BeidPrep includes a **Gemini AI-powered Card Architect** that can generate a complete, ready-to-use template for *any* Radioss card — including obscure, specialised, or newly released keywords — directly from documentation.

#### How It Works

1. Navigate to **Templates → Add Card from URL**.
2. Paste the URL of any Altair Radioss or OpenRadioss documentation page.
3. BeidPrep's scraper fetches the page, extracts tables, field definitions, and hierarchy information, and compiles a structured prompt.
4. **Google Gemini 2.0 Flash** processes the prompt under a 13-rule engineering schema and returns a validated JSON template capturing:
   - Keyword name and all known aliases
   - Header format string with field widths and column positions
   - Fixed and repeating row definitions
   - Field types (`int`, `float`, `string`, `blank`, `flag`)
   - Conditional visibility rules (show field only if another field has a specific value)
   - Dynamic repeat-count expressions
   - Category classification (Materials, Constraints, Load Cases, etc.)
5. The new card is written into your local `templates_library.json` and immediately available in the editor — no restart required.

#### API Key Setup

Navigate to **Templates → Set Google API Key…** and enter your Gemini API key. The key is stored for the session. Alternatively, set the environment variable `BEIDPREP_GEMINI_API_KEY` before launching the application.

#### Template File Management

Your `templates_library.json` lives **next to the executable** and can be freely shared with colleagues:

| Action | Menu Item | Effect |
|---|---|---|
| Load a colleague's template pack | Templates → Load Template… | Replaces your active library with the selected JSON |
| Share your custom cards | Templates → Save Template As… | Exports a copy of your library to any location |
| Recover the factory defaults | Templates → Reset to Default Template | Restores the original bundled library |

---

## 3. Selection Manager

The Selection Manager is the interactive bridge between the 3D viewport and the solver card data. It implements a **Filter → Buffer → Action** pipeline that makes visual-to-numerical data transfer precise and repeatable.

### Selection Targets (`What`)

| Target | Description |
|---|---|
| **Node** | Individual mesh nodes — used for boundary conditions, constraints, initial conditions |
| **Element** | Individual elements (shells, bricks, beams, etc.) — used for part assignment and groups |
| **Segment (Face)** | External faces of solid or shell elements — used to build `/SURF/SEG` contact surfaces |
| **Part** | All elements belonging to a named part — bulk selection |
| **Set / Group** | Members of an existing group card |
| **Config (by attribute)** | Elements sharing the same material or property ID |

### Selection Methods (`How`)

| Method | Description |
|---|---|
| **Rectangle** | Click-drag to draw a rectangular pick window |
| **Circle** | Click-drag from centre to radius for circular windowing |
| **Lasso** | Free-form polygon selection |
| **Pick by Face** | Single-click individual surface faces; angle tolerance controls adjacency growth |

### Selection Modes

| Mode | Mouse Action | Behaviour |
|---|---|---|
| **ADD** | Left-click / Left-drag | Adds picked items to the current selection buffer |
| **SUB** | Right-click / Right-drag | Removes picked items from the current selection buffer |
| **REPLACE** | Configurable | Clears the buffer and sets only the new picks |

### Grow & Invert

- **Grow**: Expands the current element/face selection to all geometrically adjacent items within the configured **angle tolerance** (0.0° – 89.0°). Use this to flood-fill a surface region from a single seed pick.
- **Invert**: Selects everything *except* the current buffer — useful for "select all but this component" workflows.

### Filter → Buffer → Action Pipeline

```
Mouse interaction in 3D viewport
        │
        ▼
[FILTER] VTK ray-cast picker
  - Resolves surface cell IDs to volume element IDs
  - Maps face hits back to element faces (segment records)
  - Applies angle tolerance for face adjacency filtering
        │
        ▼
[BUFFER] SelectionBuffer (in-memory)
  - nodes:  Set[int]
  - elems:  Set[int]
  - parts:  Set[int]
  - faces:  Dict[FaceKey → SegmentRecord]
  - Mode applied: ADD / SUB / REPLACE
  - Signal emitted → HUD count updated
        │
        ▼
[ACTION] User clicks target card button
  - Export nodes → /GRNOD/NODE
  - Export faces → /SURF/SEG  (segment IDs auto-generated)
  - Create /BCS from node set
  - Create /INIVEL, /IMPDISP, /MPC, /RLINK, etc.
```

### What You Can Create from a Selection

**Node Sets & Surface Segments**

- `/GRNOD/NODE` — node groups for load application and contact definitions
- `/SURF/SEG` — surface segment sets for contact surfaces; BeidPrep auto-generates sequential segment IDs from picked faces

**Boundary Conditions**

- `/BCS` — displacement constraints; configure free/fixed DOF interactively
- `/INIVEL` — assign initial velocity vectors to the selected nodes
- `/IMPDISP` — imposed displacement time history on selected nodes
- `/IMPVEL` — imposed velocity time history on selected nodes

**Contacts & Interfaces**

- `/INTER/TYPE2` — penalty-method contact (slave surface from selection)
- `/INTER/TYPE24` — penalty-method interface with edge treatment

**Constraints**

- `/RBE2` — rigid body element; pick master node, then slave set
- `/RBE3` — interpolation/distributed coupling element
- `/MPC` — multi-point constraint (linear DOF coupling)
- `/RLINK` — rigid link between two nodes
- `/CYL_JOINT` — cylindrical kinematic joint
- `/GJOINT` — generalised kinematic joint

**Geometry Transformations** (applied to selected nodes) (Debugging is ongoing)

- Translate (ΔX, ΔY, ΔZ)
- Rotate (axis + angle)
- Scale (uniform factor)
- Mirror / symmetric copy

---

## 4. Supported Card Library — Technical Reference - The Default Library 

### Element Types

| Keyword | Description | Nodes |
|---|---|---|
| `/SHELL` | 2D shell element (Belytschko-Tsay, BATOZ, etc.) | 3 or 4 |
| `/SH3N` | 3-node degenerate triangular shell | 3 |
| `/BRICK` | 8-node hexahedral solid element | 8 |
| `/BEAM` | 1D beam element (Euler-Bernoulli / Timoshenko) | 2 |
| `/TRUSS` | 1D truss element (axial force only) | 2 |
| `/SPRING` | 1D spring / dashpot element | 2 |
| `/SPHCEL` | Spherical particle cell (SPH method) | 1 |
| `/XELEM` | Extended / extra element definition | — |
| `/PENTA` | 5-node pentahedral solid | 5 |
| `/TETRA` | 4-node tetrahedral solid | 4 |

### Material Models

| Keyword | Alias | Description |
|---|---|---|
| `/MAT/LAW1` | `/MAT/ELAST` | Linear elastic (isotropic) |
| `/MAT/LAW2` | `/MAT/PLAS_JOHNS` | Johnson-Cook elasto-plastic |
| `/MAT/LAW19` | `/MAT/FABRI` | Fabric / textile material |
| `/MAT/LAW36` | `/MAT/PLAS_TAB` | Tabulated elasto-plastic (curve-driven) |
| `/MAT/LAW42` | `/MAT/OGDEN` | Ogden hyper-elastic |
| `/MAT/LAW62` | `/MAT/VISC_HYP` | Visco-hyper-elastic |
| `/MAT/LAW70` | `/MAT/FOAM_TAB` | Tabulated foam (crushing) |
| `/MAT/LAW79` | `/MAT/JOHN_HOLM` | Johnson-Holmquist (ceramics, concrete) |

### Property Types

| Keyword | Alias | Description |
|---|---|---|
| `/PROP/TYPE1` | `/PROP/SHELL` | Shell section (thickness, integration, hourglass) |
| `/PROP/TYPE9` | `/PROP/SH_ORTH` | Orthotropic shell |
| `/PROP/TYPE14` | `/PROP/SOLID` | Solid element property |
| `/PROP/TYPE19` | `/PROP/PLY` | Composite ply stack |

### Failure Models

| Keyword | Description |
|---|---|
| `/FAIL/TENSSTRAIN` | Tensile strain failure criterion |
| `/FAIL/JOHNSON` | Johnson-Cook failure model |

### Constraints & Connections

`/RBE2`, `/RBE3`, `/MPC`, `/RLINK`, `/CYL_JOINT`, `/GJOINT`

### Interfaces (Contact)

`/INTER/TYPE2`, `/INTER/TYPE24`

### Groups & Sets

`/GRNOD/NODE`, `/SURF/SEG`, `/PART`

### Load Cases & Boundary Conditions

`/BCS`, `/PLOAD`, `/GRAV`, `/INIVEL`, `/IMPDISP`, `/IMPVEL`, `/CLOAD`

### Control & Output

`/RUN`, `/DT/NODA`, `/ANIM/DT`, `/TFILE`, `/H3D`, `/PRINT`, `/TH`, `/IOFLAG`, `/UNIT`, `/BEGIN`, `/END`, `/TITLE`

### Tools

`/FUNCT` (curve definition with interactive plot editor), `/TABLE`

---

## 5. GUI & 3D Viewer

### Application Layout

```
┌─────────────────────────────────────────────────────────────────┐
│  Menu Bar: File | Templates | View | Info                       │
├───────────────┬─────────────────────────────┬───────────────────┤
│               │                             │                   │
│  Component    │    3D Viewport (PyVista)    │   Card Editor     │
│  Tree         │                             │   Panel           │
│  (left)       │   [Selection HUD overlay]   │   (right)         │
│               │                             │                   │
├───────────────┴─────────────────────────────┴───────────────────┤
│  Status Bar                                                      │
└─────────────────────────────────────────────────────────────────┘
```

### Menu Bar

**File**
- `Open .rad File` (`Ctrl+O`) — Load a Radioss Starter deck
- `Save .rad File` (`Ctrl+S`) — Write the current deck back to disk
- `Import LS-DYNA (.k)` — Convert an LS-DYNA keyword file to Radioss (2D and 3D elements only)

**Templates**
- `Add Card from URL` — Launch the AI Card Architect (Gemini-powered, Gemini API key is needed)
- `Load Template…` — Replace the active template library with a custom JSON file
- `Save Template As…` — Export the current template library to a chosen path
- `Reset to Default Template` — Restore the factory-bundled template library
- `Set Google API Key…` — Configure the Gemini API key for AI card generation

**View**
- `Background…` — Set the 3D viewport background colour (RGB / Hex picker)
- `Copy 3D View to Clipboard` — Screenshot the viewport to clipboard
- `Copy Window to Clipboard` — Screenshot the full application window
- `Theme → Dark` / `Theme → Light` — Switch between Obsidian dark and clean light themes
- `Show Edges` (toggle) — Overlay mesh wireframe on solid elements

**Info**
- `About` — Application version, credits, legal notice
- `Model Info` — Statistics dialog: node count, element count by type, card count by category

---

### Component Tree (Left Panel)

The Component Tree provides a hierarchical view of every card in the loaded deck, organised by Radioss category:

> **Nodes · Elements · Materials · Properties · Interfaces · Constraints · Groups (Sets) · Load Cases · General Controls · Tools / Curves · Component and Parts · Output Database · Etc..**

**Per-card controls** (inline icons):
- **Eye** — Toggle visibility of this card's geometry in the 3D viewport
- **Colour swatch** — Change the display colour assigned to this card
- **Mesh icon** — Toggle mesh / solid rendering mode for this card
- **Star** — Bookmark / favourite a card for quick access

**Right-click context menu** on any card:
- Edit (opens the Card Editor panel)
- Delete
- Duplicate
- Copy ID
- View in 3D (isolate and fit-to-view)

**Left mouse click** any card to open the Card Editor.

---

### Card Editor Panel (Right Panel)

The Card Editor is template-driven: every field, its type, column position, and label is read from the JSON template schema. There is no hard-coded form layout.

**Field types rendered:**
- Text input for strings and free-format values
- Numeric spinbox for float and integer fields
- Drop-down for flag and enumeration fields
- **Dynamic visibility** — fields appear or hide based on the value of other fields (e.g., showing extra rows only when a flag is set to a specific value)

**Repeat rows:**
- Cards with variable-length data (node lists, element tables, curve points) show an **Add Row / Remove Row** control
- Rows are added and removed interactively; the formatter writes the correct line count into the header on save

**Picker integration:**
- Fields that reference IDs (part ID, material ID, node ID) show a **Pick** button that activates the Selection Manager in the 3D viewport and back-fills the field on selection

**Action buttons (card-dependent):**
- `Add from Selection` — Populate node/face lists from the current selection buffer
- `Clear All` — Reset all data rows
- `Save` — Commit changes to the card and update the 3D view
- `Create` — For element creation dialogs (Beam, Truss, Spring)

**Excel Paste:**
- Paste (`Ctrl+V`) tabular data copied from Excel or any spreadsheet directly into the active field table

**Plot Editor** (for `/FUNCT` cards):
- Interactive matplotlib canvas
- Drag curve points to edit values in real time
- X/Y axis display with undo support

**Plot Digitizer** (for `/FUNCT` cards):
- Load a scanned image of an experimental stress-strain curve
- Click to extract data point coordinates
- Export as a complete `/FUNCT` card

---

### 3D Viewport

The 3D viewport is powered by **PyVista** (VTK back-end) rendered inside a PySide6 . It displays the full mesh geometry with per-card colour coding and supports hardware-accelerated OpenGL rendering.

#### Selection HUD Overlay

A floating, glassmorphism-styled panel sits in the top-left corner of the viewport. It contains all Selection Manager controls and live feedback:

**What** (pick target):
`Node` · `Element` · `Segment (Face)` · `Part` · `Set/Group`

**How** (pick method):
`Rectangle` · `Circle` · `Lasso` · `Pick by Face`

**Mode**: `ADD` · `SUB` · `REPLACE`

**Angle Tolerance**: spinbox (0.0° – 89.0°) controlling face adjacency for Grow and Face-pick

**Buttons**: `Grow` · `Clear`

**Camera shortcuts**: `Top` · `Front` · `Side` · `Isometric` · `Flip`

**Live feedback line**: *"Selected 47 nodes"* / *"Picked 12 faces"* — updates immediately on every pick event.

#### Mouse Interaction Summary

| Action | Result |
|---|---|
| `LMB click` on mesh | Add item to selection buffer (ADD mode) |
| `LMB drag` | Rectangle / Circle / Lasso window selection |
| `RMB click` on mesh | Remove item from selection (SUB mode) |
| `RMB drag` | Subtract-window selection |
| `Shift + LMB drag` | Rotate camera (pick suspended) |
| `Shift + scroll` | Zoom (direction reversed — engineering convention) |
| `Double-click` tree item | Open Card Editor for that card |
| `RMB click` tree item | Context menu (Edit / Delete / Duplicate / Copy ID) |

---

### Themes

| Theme | Primary BG | Accent | Text |
|---|---|---|---|
| **Dark — Obsidian**  | `rgb(18, 18, 18)` | `#00D4FF` electric blue · `#00FF41` electric green | `#FFFFFF` / `#B0B0B0` |
| **Light** (default)| `rgb(240, 242, 245)` | `#0077AA` blue · `#00AA00` green | `#222222` / `#111111` |

Both themes apply to the entire application window, including the HUD overlay and all dialogs. Semi-transparent glassmorphism panels are used for the dark theme; clean flat surfaces for the light theme.

---

## 6. Installation

BeidPrep Pro is distributed as a **self-contained, single-file Windows executable**. No Python installation, no Anaconda environment, no `pip install` — nothing.

### Distribution Package

```
BeidPrepPro2026.exe          ← Application (double-click to launch)
templates_library.json       ← Active template library (editable / replaceable)
```

Both files must be kept in the same folder. The application reads and writes `templates_library.json` from the directory where the `.exe` lives.

**First run behaviour:** If `templates_library.json` is missing (e.g., fresh install without the JSON), the application automatically extracts the factory-bundled default copy and writes it next to the executable.

### System Requirements

| Requirement | Minimum |
|---|---|
| OS | Windows 10 64-bit or later |
| RAM | 4 GB (8 GB recommended for large meshes) |
| GPU | Any OpenGL 3.3-capable GPU (integrated graphics supported) |
| Disk | ~200 MB (executable + dependencies) |

### Build from Source (Developers)

If you have the source code and a Python 3.10+ environment with all dependencies installed:

```bash
pip install pyinstaller
pyinstaller BeidPrepPro2026.spec
```

Output: `dist/BeidPrepPro2026.exe`

Key dependencies: `PySide6`, `pyvista`, `vtk`, `numpy`, `google-generativeai`, `requests`, `qt_material`

---

## 7. License

```
Copyright (c) 2026 Konstantin Arhiptsov / Beider Simulation.
All Rights Reserved.

Free for use, but source code is proprietary.

Redistribution of the compiled executable is permitted for
commercial purposes provided this copyright notice is retained.
Modification, reverse-engineering, or redistribution of the source
code in any form is strictly prohibited without prior written
permission from the copyright holder.

BeidPrep v1.0 is currently in Beta and is provided to the community as free software "as-is".
By downloading and using this software, you acknowledge the following:
No Liability: I assume no responsibility or liability for any errors, model inaccuracies, or engineering failures resulting from the use of this tool.
Engineering Judgment: This software is a productivity aid and does not replace the requirement for professional engineering judgment and thorough verification of simulation inputs and results.
```

Beta Status: As a Beta release, you may encounter bugs. I encourage your feedback to help improve the tool, but I provide no warranties regarding its performance or stability.

---

*BeidPrep Pro — Engineered for solvers that don't compromise.*
