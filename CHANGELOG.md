# Changelog

## [1.0.1] - 2026-02-28

### 🚀 Major New Features
- **Model Merging (Append Mode):** Import multiple .k or .rad models into a single session with automated ID shifting.
- **"Blank Slate" Mode:** Start projects without a mesh to create and save analysis card templates.
- **Splash Screen:** Professional 800×450 px startup screen (Splash.png) that remains visible until the main window is ready.
- **Resizable Tree Panel:** The category tree is now adjustable (160–560 px) with a hover-and-drag handle.
- **"By Part" Selection:** New HUD option for single-click or area-drag selection of entire parts.
- **Pivot Rotation:** Transform elements around a custom X/Y/Z point or snapped "Node ID".

### 🛠️ Critical Bug Fixes
- **Performance & Memory:**
  - **132x Memory Reduction:** Replaced `pv.merge` logic with a shared point helper, reducing memory usage from 220 MB to 1.6 MB on large models.
  - **UTF-8 Encoding:** Fixed garbled characters and BOM issues by implementing `utf-8-sig` encoding and launcher wrappers.
- **Mesh & Logic:**
  - **Recursive Deletion:** Deleting a /PART now correctly triggers a cascaded deletion of all associated elements.
  - **Isolate/Hide Fix:** Rewritten at the element-level to prevent entire parts from being hidden unintentionally.
  - **LS-DYNA Importer:** Fixed translation and connectivity issues in .k files.
- **GUI & Viewport:**
  - **RMB Context Menu:** Right-click now always opens; relevant actions are intelligently greyed out if selection is empty.
  - **Selection Visuals:** New neon-green fill + dark green wireframe for selected elements.
  - **Node Visibility:** Minimum sphere size raised to 8.0 px for clarity at high zoom.
  - **NVIDIA/AMD Hotfix:** Resolved viewport crashes on specific dedicated GPU configurations.

### ⚡ Performance Improvements
- **Optimized Loading:** Significant speed increases for file opening and viewport interaction.
- **Picker Efficiency:** Optimized two-pass node picker (PointPicker + CellPicker fallback).
