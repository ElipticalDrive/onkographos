# OpenVDB to Yocto Volume Conversion

Also called: yovdbload, OpenVDB dump to Yocto .vol

## Overview
This method converts sparse OpenVDB volume grids (typically density and temperature) into a dense, contiguous float array with `LayoutXYZ` ordering, packaged as a Yocto Volume (`.vol`) file for rendering in Yocto/Quasar. It differs from the parent [Sparse-to-Dense Grid Conversion](12676-sparse-to-dense-grid-conversion.md) by specifically targeting the Yocto ecosystem's `.vol` format and utilizing the `yovdbload` toolchain, whereas the parent exports to legacy VTK `STRUCTURED_POINTS`. The method handles coordinate translation to ensure the volume bounding box starts at `(0,0,0)` in the dense grid space.

## How it works
**Inputs:**
- One or more OpenVDB `.vdb` files containing volume grids (e.g., density, temperature).
- The `openvdb_print` utility and a custom dump function.
- The `yovdbload` application (part of the Yocto extension tools).

**Outputs:**
- A `.vol` file containing a `yocto::image::volume<float>` object.

**Data Structures:**
1.  **Intermediate Binary Dump (`.bin`):** A binary stream of structs for each voxel.
    -   Layout: `int32_t cx`, `int32_t cy`, `int32_t cz`, `float cdensity` (repeated for each active voxel).
2.  **Dense Volume (`yocto::image::volume<float>`):**
    -   `extent`: `vec3f` containing dimensions $(mx, my, mz)$.
    -   `voxels`: `std::vector<float>` storing values in `LayoutXYZ` order.
    -   Indexing: `operator[](vec3i)` maps 3D coordinates to the linear vector.

**Steps:**

1.  **Inspect OpenVDB Grids:**
    Use `openvdb_print` to identify the names and types of density and optional temperature grids within the `.vdb` file [S1, unofficial].

2.  **Dump to Binary:**
    Use a custom function to iterate over the OpenVDB grids and write a `.bin` file for each property. The file contains the active voxel coordinates and their float values in the struct layout defined above [S1, unofficial].

3.  **Load and Sort Voxels:**
    Read the `.bin` file using `read_ovdbdump`. The loaded vector of voxels is sorted using `sort_ovdb_vector` with the key order: $z$, then $y$, then $x$ [S1, unofficial].

4.  **Calculate Bounding Box:**
    Find the minimum and maximum coordinates of the sparse voxels:
    -   Let $min_x, min_y, min_z$ be the smallest coordinate values found.
    -   Let $max_x, max_y, max_z$ be the largest coordinate values found.
    -   Calculate the translation offset (low bounds):
        $$ low_x = -min_x $$
        $$ low_y = -min_y $$
        $$ low_z = -min_z $$
    -   Calculate the dense grid dimensions:
        $$ mx = max_x + low_x + 1 $$
        $$ my = max_y + low_y + 1 $$
        $$ mz = max_z + low_z + 1 $$
    This ensures the volume bounding box is translated so the lower bound is at $(0, 0, 0)$ and the upper bound is at $(mx-1, my-1, mz-1)$ [S1, unofficial].

5.  **Initialize Dense Array:**
    Create a `std::vector<float>` of size $mx \times my \times mz$, initialized to $0.0f$ [S1, unofficial].

6.  **Map Voxels to Dense Grid:**
    For each voxel $(x, y, z)$ with value $v$ from the sorted list, compute the linear index in `LayoutXYZ` order:
    $$ index = (z + low_z) \times my \times mx + (y + low_y) \times mx + (x + low_x) $$
    Store $v$ at `voxels[index]` [S1, unofficial].

7.  **Construct Yocto Volume:**
    Create an `img::volume<float>` object with dimensions `vec3i{mx, my, mz}` and the populated `voxels` vector [S1, unofficial].

8.  **Save:**
    Use `img::save_volume` to write the volume to a `.vol` file [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def convert_openvdb_to_yocto_vdb(vdb_path: str, output_vol_path: str):
    # Step 1: Inspect and Dump
    # Use openvdb_print to identify grids.
    # Use custom dump function to generate .bin files.
    # .bin format: [int32_t x, int32_t y, int32_t z, float value] repeated.
    bin_files = dump_openvdb_grids(vdb_path)

    for bin_file in bin_files:
        # Step 3: Load and Sort
        voxels = read_ovdb_dump(bin_file)
        # Sort by z, then y, then x
        voxels.sort(key=lambda v: (v.z, v.y, v.x))

        # Step 4: Calculate Bounds
        if not voxels:
            continue

        min_x = min(v.x for v in voxels)
        min_y = min(v.y for v in voxels)
        min_z = min(v.z for v in voxels)
        max_x = max(v.x for v in voxels)
        max_y = max(v.y for v in voxels)
        max_z = max(v.z for v in voxels)

        low_x = -min_x
        low_y = -min_y
        low_z = -min_z

        mx = max_x + low_x + 1
        my = max_y + low_y + 1
        mz = max_z + low_z + 1

        # Step 5: Initialize Dense Array
        dense_voxels = [0.0f] * (mx * my * mz)

        # Step 6: Map Voxels
        for v in voxels:
            idx = (v.z + low_z) * my * mx + (v.y + low_y) * mx + (v.x + low_x)
            dense_voxels[idx] = v.value

        # Step 7: Construct Volume Object
        volume = VolumeFloat()
        volume.extent = vec3f(mx, my, mz)
        volume.voxels = dense_voxels

        # Step 8: Save
        save_volume(output_vol_path, volume)
```

## Subtopics

### Yocto Volume Format (.vol)
A Yocto-compatible volume file format obtained from OpenVDB `.vdb` files using the `yovdbload` application. It stores the volume as a dense grid with `LayoutXYZ` ordering. The format supports indexing via `operator[](vec3i)` and stores the volume extent as `vec3f`. It is chosen when rendering within the Yocto/Quasar ecosystem is required, as it is natively supported by `img::load_volume` and `img::save_volume` [S1, unofficial].

## Operating characteristics
-   **Time Complexity:** Dominated by the sorting step $O(N \log N)$ where $N$ is the number of active voxels, and the mapping step $O(N)$. The initialization of the dense array is $O(M)$ where $M = mx \times my \times mz$.
-   **Memory Complexity:** $O(M)$ for the dense array. This can be significantly larger than the sparse OpenVDB representation if the volume is sparse, leading to high memory usage for large bounding boxes with few active voxels.
-   **Strengths:** Produces a format directly usable by Yocto renderers without further conversion. Simple `LayoutXYZ` indexing is cache-friendly for dense traversal.
-   **Limitations:** Inefficient for highly sparse volumes due to the dense allocation. The coordinate translation assumes the volume fits within memory limits.
-   **When to use:** When the target renderer is Yocto/Quasar and the volume density is sufficient to justify dense storage.
-   **When not to use:** For very sparse volumes where OpenVDB's native sparse traversal is preferred, or when interoperability with other engines (like VTK) is required.

## Background
This method originates from the `yocto-gl-vpt` project, an unofficial extension to Yocto/Quasar for volume path tracing. It was developed to bridge the gap between industry-standard sparse volume formats (OpenVDB) and Yocto's native volume handling. The `yovdbload` tool and associated dump functions were created to facilitate this conversion pipeline [S1, unofficial].

## Sources
[S1] RickyMexx/yocto-gl-vpt, https://github.com/RickyMexx/yocto-gl-vpt, fetched 2026-10-26 (unofficial)
