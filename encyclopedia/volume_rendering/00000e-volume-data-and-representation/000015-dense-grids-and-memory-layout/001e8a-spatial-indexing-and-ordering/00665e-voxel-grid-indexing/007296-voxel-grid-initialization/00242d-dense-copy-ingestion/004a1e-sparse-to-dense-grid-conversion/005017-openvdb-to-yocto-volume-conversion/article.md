# OpenVDB to Yocto Volume Conversion


## Overview
This method converts sparse OpenVDB volume grids (typically density and temperature) into a dense, contiguous float array with `LayoutXYZ` ordering, packaged as a Yocto Volume (`.vol`) file for rendering in Yocto/Quasar [S1, unofficial]. It differs from the parent [Sparse-to-Dense Grid Conversion](12676-sparse-to-dense-grid-conversion.md) by specifically targeting the Yocto ecosystem's `.vol` format and utilizing the `yovdbload` toolchain, whereas the parent exports to legacy VTK `STRUCTURED_POINTS` [S1, unofficial]. The method handles coordinate translation to ensure the volume bounding box starts at `(0,0,0)` in the dense grid space [S1, unofficial].

## How it works
**Inputs:**
- One or more OpenVDB `.vdb` files containing volume grids (e.g., density, temperature) [S1, unofficial].
- The `openvdb_print` utility and a custom dump function [S1, unofficial].
- The `yovdbload` application (part of the Yocto extension tools) [S1, unofficial].

**Outputs:**
- A `.vol` file containing a `yocto::image::volume<float>` object [S1, unofficial].

**Data Structures:**
1.  **Intermediate Binary Dump (`.bin`):** A binary stream of structs for each voxel [S1, unofficial].
    -   Layout: `int32_t cx`, `int32_t cy`, `int32_t cz`, `float cdensity` (repeated for each active voxel) [S1, unofficial].
2.  **Dense Volume (`yocto::image::volume<float>`):**
    -   `extent`: `vec3f` containing dimensions $(mx, my, mz)$ [S1, unofficial].
    -   `voxels`: `std::vector<float>` storing values in `LayoutXYZ` order [S1, unofficial].
    -   Indexing: `operator[](vec3i)` maps 3D coordinates to the linear vector [S1, unofficial].

**Steps:**

1.  **Inspect OpenVDB Grids:**
    Use `openvdb_print` to identify the names and types of density and optional temperature grids within the `.vdb` file [S1, unofficial].

2.  **Dump to Binary:**
    Use a custom function to iterate over the OpenVDB grids and write a `.bin` file for each property [S1, unofficial]. The file contains the active voxel coordinates and their float values in the struct layout defined above [S1, unofficial].

3.  **Load and Sort Voxels:**
    Read the `.bin` file using `read_ovdbdump` [S1, unofficial]. The loaded vector of voxels is sorted using `sort_ovdb_vector` with the key order: $z$, then $y$, then $x$ [S1, unofficial].

4.  **Calculate Bounding Box:**
    Find the minimum and maximum coordinates of the sparse voxels [S1, unofficial].
    -   Let $min\_coord\_x, min\_coord\_y, min\_coord\_z$ be the smallest coordinate values found [S1, unofficial].
    -   Let $max\_coord\_x, max\_coord\_y, max\_coord\_z$ be the largest coordinate values found [S1, unofficial].
    -   Calculate the translation offset (low bounds):
        $$ low\_x = |min\_coord\_x| $$
        $$ low\_y = |min\_coord\_y| $$
        $$ low\_z = |min\_coord\_z| $$
    -   Calculate the dense grid dimensions:
        $$ mx = max\_coord\_x + |min\_coord\_x| + 1 $$
        $$ my = max\_coord\_y + |min\_coord\_y| + 1 $$
        $$ mz = max\_coord\_z + |min\_coord\_z| + 1 $$
    This ensures the volume bounding box is translated so the lower bound is at $(0, 0, 0)$ and the upper bound coordinates are $(mx-1, my-1, mz-1)$ [S1, unofficial].

5.  **Initialize Dense Array:**
    Create a `std::vector<float>` of size $mx \times my \times mz$, initialized to $0.0f$ [S1, unofficial].

6.  **Map Voxels to Dense Grid:**
    For each voxel $(x, y, z)$ with value $v$ from the sorted list, compute the linear index in `LayoutXYZ` order:
    $$ index = (z + low\_z) \times my \times mx + (y + low\_y) \times mx + (x + low\_x) $$
    Store $v$ at `voxels[index]` [S1, unofficial].

7.  **Construct Yocto Volume:**
    Create an `img::volume<float>` object with dimensions `vec3i{mx, my, mz}` and the populated `voxels` vector [S1, unofficial].

8.  **Save:**
    Use `img::save_volume` (via `yovdbload`) to write the volume to a `.vol` file [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
from typing import List, Tuple

class Voxel:
    def __init__(self, x: int, y: int, z: int, value: float):
        self.x, self.y, self.z, self.value = x, y, z, value

def convert_openvdb_to_yocto_vdb(bin_files: List[str], output_vol_path: str):
    # Step 1 & 2: Inspect OpenVDB grids and dump to .bin files.
    # The .bin file contains xyz coordinates and float values per voxel.
    for bin_file in bin_files:
        # Step 3: Load voxels from .bin and sort by z, then y, then x.
        voxels = load_bin_file(bin_file)
        voxels.sort(key=lambda v: (v.z, v.y, v.x))

        if not voxels:
            continue

        # Step 4: Calculate bounding box bounds.
        min_x = min(v.x for v in voxels)
        min_y = min(v.y for v in voxels)
        min_z = min(v.z for v in voxels)
        max_x = max(v.x for v in voxels)
        max_y = max(v.y for v in voxels)
        max_z = max(v.z for v in voxels)

        low_x = abs(min_x)
        low_y = abs(min_y)
        low_z = abs(min_z)

        mx = max_x + low_x + 1
        my = max_y + low_y + 1
        mz = max_z + low_z + 1

        # Step 5: Initialize dense array.
        dense_voxels = [0.0] * (mx * my * mz)

        # Step 6: Map voxels to dense grid in LayoutXYZ order.
        for v in voxels:
            idx = (v.z + low_z) * my * mx + (v.y + low_y) * mx + (v.x + low_x)
            dense_voxels[idx] = v.value

        # Step 7 & 8: Construct volume object and save.
        volume = {"extent": (mx, my, mz), "voxels": dense_voxels}
        save_volume_to_file(output_vol_path, volume)

def load_bin_file(path: str) -> List[Voxel]:
    # Reads binary stream of (int32_t x, int32_t y, int32_t z, float value)
    # Returns list of Voxel objects.
    pass

def save_volume_to_file(path: str, volume: dict):
    # Writes volume dict to .vol file using img::save_volume logic.
    pass
```

## Subtopics

### Yocto Volume Format (.vol)
A Yocto-compatible volume file format obtained from OpenVDB `.vdb` files using the `yovdbload` application [S1, unofficial]. It stores the volume as a dense grid with `LayoutXYZ` ordering [S1, unofficial]. The format supports indexing via `operator[](vec3i)` and stores the volume extent as `vec3f` [S1, unofficial]. It is chosen when rendering within the Yocto/Quasar ecosystem is required, as it is natively supported by `img::load_volume` and `img::save_volume` [S1, unofficial].

## Operating characteristics
-   **Time Complexity:** Dominated by the sorting step $O(N \log N)$ where $N$ is the number of active voxels, and the mapping step $O(N)$ [G]. The initialization of the dense array is $O(M)$ where $M = mx \times my \times mz$ [G].
-   **Memory Complexity:** $O(M)$ for the dense array [G]. This can be significantly larger than the sparse OpenVDB representation if the volume is sparse, leading to high memory usage for large bounding boxes with few active voxels [G].
-   **Strengths:** Produces a format directly usable by Yocto renderers without further conversion [S1, unofficial]. Simple `LayoutXYZ` indexing is cache-friendly for dense traversal [G].
-   **Limitations:** Inefficient for highly sparse volumes due to the dense allocation [G]. The coordinate translation assumes the volume fits within memory limits [G].
-   **When to use:** When the target renderer is Yocto/Quasar and the volume density is sufficient to justify dense storage [S1, unofficial].
-   **When not to use:** For very sparse volumes where OpenVDB's native sparse traversal is preferred, or when interoperability with other engines (like VTK) is required [G].

## Background
This method originates from the `yocto-gl-vpt` project, an unofficial extension to Yocto/Quasar for volume path tracing [S1, unofficial]. It was developed to bridge the gap between industry-standard sparse volume formats (OpenVDB) and Yocto's native volume handling [S1, unofficial]. The `yovdbload` tool and associated dump functions were created to facilitate this conversion pipeline [S1, unofficial].

## Sources
[S1] RickyMexx/yocto-gl-vpt, https://github.com/RickyMexx/yocto-gl-vpt, fetched 2026-10-26 (unofficial)
