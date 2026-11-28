# Device Volume Snapshot

Also called: DeviceVolume, VXL-042 Storage, Volume Immutable Storage, toDevice

## Overview
The Device Volume Snapshot is a compact, immutable representation of volume metadata and texture handles optimized for passing to CUDA kernels [S2, unofficial]. It solves the problem of transferring volume state from a mutable host-side builder to device code without exposing internal memory layouts or risking dangling pointers [S1]. It sets itself apart from sibling methods by enforcing immutability upon construction and using a shared-resource model for lightweight copying, ensuring thread-safe sharing across multiple frames or kernels [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A mutable `Volume::Builder` containing cells, definitions, and grid dimensions [S1].
*   **Output:** An immutable `DeviceVolume` (or `DeviceScene` snapshot) struct containing texture handles and metadata, ready for device-side access [S2, unofficial].

**Data Structures:**
The core data structure is the `DeviceVolume` struct (referred to as `DeviceScene` in the reference implementation [S2, unofficial]). It contains the following fields:
*   `field_tex`: `cudaTextureObject_t` – The primary volume texture handle [S2, unofficial].
*   `grad_tex`: `cudaTextureObject_t` – The gradient texture handle, which can be `0` if gradients are not computed or required [S2, unofficial].
*   `dim`: `int3` – The dimensions of the volume grid $(D_x, D_y, D_z)$ [S2, unofficial].
*   `voxel_size`: `float3` – The physical size of a single voxel $(S_x, S_y, S_z)$ [S2, unofficial].
*   `origin`: `float3` – The world-space origin of the volume grid [S2, unofficial].
*   `value_range`: `float2` – The minimum and maximum scalar values in the volume $(V_{min}, V_{max})$ [S2, unofficial].
*   `density_scale`: `float` – A scalar multiplier for density values [S2, unofficial].

**Internal Storage Layout (Builder Phase):**
Before snapshotting, the `Volume` class uses a specific internal layout defined by the VXL-042 specification [S1]:
*   **Definition Table:** A local table storing unique `Definition` objects [S1]. Multiple cells can share the same `Definition` (and its underlying `Data`) to save memory [S1].
*   **Compact Grid:** A grid of cells that does not store full definitions [S1]. Instead, it stores indices into the Definition Table and orientation data [S1].
*   **Immutability:** Once `build()` is called on the `Builder`, the internal table and grid are frozen [S1]. The `Volume` object shares these resources internally, making copies lightweight [S1].

**Numbered Steps:**
1.  **Build:** The `Volume::Builder` accepts `Cell` objects by value [S1]. It deduplicates `Definition` objects and populates the compact grid with indices and orientations [S1].
2.  **Construct Snapshot:** Upon calling `build()`, the system creates the immutable `Volume` snapshot [S1].
3.  **Initialize Device Struct:** The `DeviceVolume` struct is populated with values derived from the `Volume::Description` and the bound texture handles [S2, unofficial].
4.  **Transfer to Device:** The `Scene` class manages the transfer of the snapshot to device memory via its `commit(cudaStream_t stream)` method, which updates the device pointer `m_ds_dev` [S2, unofficial].

**Pseudocode:**
The following pseudocode illustrates the construction and usage flow based on the reference implementations [S1][S2, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Self-contained implementation matching evidence [S1][S2, unofficial]
class VolumeBuilder:
    def __init__(self, dim):
        self.dim = dim
        self.definitions = {}
        self.grid = []

    def set_cell(self, pos, cell):
        if cell.definition not in self.definitions:
            self.definitions[cell.definition] = len(self.definitions)
        def_idx = self.definitions[cell.definition]
        self.grid.append((def_idx, cell.orientation))

    def build(self):
        return Volume(self.dim, self.definitions, self.grid)

class Volume:
    def __init__(self, dim, definitions, grid):
        self.dim = dim
        self.definitions = definitions
        self.grid = grid

    def cell(self, pos):
        idx = pos[0] + pos[1] * self.dim[0] + pos[2] * self.dim[0] * self.dim[1]
        def_idx, orientation = self.grid[idx]
        return {"definition": list(self.definitions.keys())[def_idx], "orientation": orientation}

class Scene:
    def __init__(self):
        self.m_ds_host = None
        self.m_ds_dev = None

    def commit(self, stream):
        tex_handles = self._get_texture_handles()
        self.m_ds_host = DeviceVolume(
            field_tex=tex_handles.field,
            grad_tex=tex_handles.grad,
            dim=self._volume.dim,
            voxel_size=(1.0, 1.0, 1.0),
            origin=(0.0, 0.0, 0.0),
            value_range=(0.0, 1.0),
            density_scale=1.0
        )
        self.m_ds_dev = self._copy_to_device(self.m_ds_host, stream)

    def _get_texture_handles(self):
        return type('Handles', (), {'field': 1, 'grad': 0})()

    def _copy_to_device(self, data, stream):
        return data
```

**Edge Cases and Pitfalls:**
*   **Dangling Pointers:** The public API must not expose `std::span<const Cell>` or raw pointers to the internal compact storage [S1]. Internal indices or pointers may become invalid if the storage layout changes or if the snapshot is shared across threads [S1]. Use `cell(position)` returning values or views instead [S1].
*   **Gradient Handling:** The `grad_tex` field may be `0` [S2, unofficial].
*   **Immutability:** Once a `Volume` is built, it must not be modified [S1]. Modifications require creating a new `Builder` and a new snapshot [S1].

## Operating characteristics
*   **Time Complexity:** Construction is $O(N)$ where $N$ is the number of cells, due to deduplication and grid population [S1]. Copying the snapshot is $O(1)$ because it shares internal resources [S1].
*   **Memory Complexity:** Storage is compact, storing only indices and orientations in the grid, plus a single copy of each unique `Definition` [S1].
*   **Strengths:** Thread-safe sharing via immutability; lightweight copying; optimized for coalesced texture access on GPU [S1][S2, unofficial].
*   **Limitations:** Not suitable for frequent updates; requires rebuilding the snapshot for changes [S1].
*   **When to use:** When volume data is static for multiple frames or kernels and needs to be passed to CUDA code [S2, unofficial].
*   **When not to use:** When frequent per-frame voxel updates are required, as rebuilding the snapshot is necessary [S1].

## Background
The concept originates from the Sparkle Voxel Engine design, specifically the VXL-042 storage specification, which emphasizes immutable volume representations and compact cell storage [S1]. The implementation details for CUDA integration, including the `DeviceScene` struct and texture handling, are derived from the cuVRender project [S2, unofficial]. The approach aligns with general GPU memory principles of immutability and safety for shared resources [8345-gpu-memory-and-buffer-construction.md].

## Sources
[S1] V0.1.4 — Voxel engine design and implementation specification, https://github.com/EreliaStudio/Sparkle/pull/37, fetched 2026-10-26
[S2] Loveof1ife7/cuVRender-A-CUDA-Accelerated-Volume-Rendering-Engine, https://github.com/Loveof1ife7/cuVRender-A-CUDA-Accelerated-Volume-Rendering-Engine, fetched 2026-10-27 (unofficial)
