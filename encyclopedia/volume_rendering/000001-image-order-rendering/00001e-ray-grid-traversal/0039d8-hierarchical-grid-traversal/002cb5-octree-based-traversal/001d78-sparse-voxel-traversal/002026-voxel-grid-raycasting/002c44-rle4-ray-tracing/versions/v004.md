# RLE4 Ray Tracing

Also called: Map4 Traversal

## Overview
RLE4 Ray Tracing is a ray-marching algorithm designed to traverse volume data compressed using Run-Length Encoding (RLE) in the Y-dimension, specifically structured as a "Map4" layout. It solves the problem of efficiently locating the first occupied voxel along a ray or coloring a voxel based on its position within sparse volumetric data. It distinguishes itself from sibling methods by avoiding full decompression of the volume; instead, it uses a texture-based lookup system to determine if a sample point lies in an uncompressed surface region, an "inside" region, or an "outside" region, returning special markers for the latter two [S1, unofficial]. This approach is particularly suited for scenarios where the volume geometry is sparse or has large uniform interior/exterior regions, allowing the renderer to skip volumetric integration in those areas [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A ray defined by origin and direction (implicitly handled by the caller providing `pos`), volume dimensions `vx, vy, vz`, and a pre-compressed RLE4 (Map4) structure containing slab pointers and Y-range data.
*   **Output:** A color value (RGB) packed into a texture array if the voxel is found, or a skip marker if not.

**Data Structures:**
The Map4 structure consists of:
*   `map`: An array of `ushort` pointers (or indices) indexed by `x + z * sx`, where `sx` is the X-dimension size. This acts as a slab pointer for each XZ coordinate.
*   `slabs`: A contiguous array of `ushort` values storing the RLE-encoded Y-ranges and color data.
*   **Slab Entry Format:** Each slab entry is a 16-bit integer (`ushort`).
    *   Bits 0-9 (10 bits): Start Y coordinate (`y1`) relative to the slab start.
    *   Bits 10-15 (6 bits): Length of the Y-range (`len = y2 - y1`). Note: The source code calculates `y2` as `y1 + (slab >> 10)`, implying the upper bits store the end Y or length. Based on `y2=y1+(slab>>10)`, the upper 6 bits represent the length or offset. Let $L = slab \gg 10$. Then $y_2 = y_1 + L$.
    *   *Correction on decoding:* The source says `y1+=slab&1023; y2=y1+(slab>>10);`. This implies `slab & 1023` is the *increment* to `y1`, not the absolute `y1`. However, standard RLE usually stores absolute or delta. Let's stick strictly to the source formula: `y1` is accumulated. `y2` is `y1 + (slab >> 10)`.

**Algorithm Steps:**

1.  **Coordinate Conversion:** Convert the ray's current float position `pos` to integer voxel coordinates `x, y, z` by scaling with volume dimensions `vx, vy, vz`.
    $$ x = \lfloor pos_x \cdot vx \rfloor, \quad y = \lfloor pos_y \cdot vy \rfloor, \quad z = \lfloor pos_z \cdot vz \rfloor $$
2.  **Bounds Check:** Verify `x` and `z` are within bounds.
    ```python
    if x < 0 or x >= vx or z < 0 or z >= vz:
        return # Out of bounds
    ```
3.  **Slab Access:** Access the slab pointer for the current XZ coordinate.
    ```python
    p = map4.slabs + map4.map[x + z * map4.sx]
    ```
4.  **Read Segment Lengths:** Read the lengths of the first and second slab segments to skip header data.
    ```python
    len1 = p[0]
    p += 1
    len2 = p[0]
    p += 1
    ```
5.  **Initialize State:**
    *   Set `y1 = 0`, `y2 = 0`.
    *   Set texture pointer `pt` to `p + len1`.
    *   Set `texture_offset = 0`.
6.  **Iterate Slabs:** Loop `s` from `0` to `len1 - 1`:
    a.  **Read Slab Entry:**
        ```python
        slab = p[0]
        p += 1
        ```
    b.  **Decode Y-Range:**
        ```python
        y1 += slab & 1023
        y2 = y1 + (slab >> 10)
        ```
    c.  **Check Intersection:** Check if the target Y coordinate `y` falls within the current slab range `[y1, y2)`.
        ```python
        if y >= y1 and y < y2:
            # Voxel found
            col = read_color_from_texture(pt, texture_offset, y - y1) # Implementation detail [G]
            # Quantize color to 5 bits per channel (0-31 range)
            r_ = clamp(float(col.x * 15 + 16), 0, 31)
            g_ = clamp(float(col.y * 15 + 16), 0, 31)
            b_ = clamp(float(col.z * 15 + 16), 0, 31)
            # Store packed RGB
            packed = uint(r_ + (g_ << 5) + (b_ << 10))
            pt[texture_offset + y - y1] = packed
            return
        ```
    d.  **Advance:** If not found, advance the texture pointer and Y1.
        ```python
        texture_offset += y2 - y1
        y1 = y2
        ```
7.  **Return:** If the loop completes without finding `y`, the voxel is in an empty region (skip).

**Parameters:**
*   Color quantization: 5 bits per channel (0-31 range).
*   Y-range decoding: 10 bits for Y-increment, 6 bits for Y-length.

**Edge Cases and Pitfalls:**
*   **Bounds:** The algorithm assumes the target Y is within the volume bounds checked at the start. If the ray passes through multiple slabs, it iterates until the Y range is found.
*   **Precision:** The color quantization uses 5 bits per channel. This may introduce visible banding in high-dynamic-range scenes.
*   **Empty Regions:** If the Y coordinate is not found in any slab, the algorithm returns without writing, effectively skipping the voxel. This relies on the caller handling the "skip" state appropriately.

## Subtopics

### [RL-compressed GPU Ray Casting with Texture-Based Lookup](10793-rl-compressed-gpu-ray-casting-with-texture-based-lookup.md)
This method performs GPU-based ray casting on volume data compressed using run-length encoding (RLE), storing the compressed structure in 2D textures to facilitate efficient random access. It distinguishes itself from sibling methods by avoiding full decompression; instead, it uses a texture-based lookup system to determine if a sample point lies in an uncompressed surface region, an "inside" region, or an "outside" region, returning special markers for the latter two. This approach is particularly suited for scenarios where the volume geometry is sparse or has large uniform interior/exterior regions, allowing the renderer to skip volumetric integration in those areas.

## Operating characteristics
*   **Time Complexity:** $O(k)$ where $k$ is the number of Y-slabs traversed along the ray. In sparse volumes, $k$ is significantly smaller than the total number of voxels.
*   **Memory Complexity:** $O(N)$ for the compressed RLE4 structure, where $N$ is the number of occupied Y-ranges. This is much smaller than the full voxel grid for sparse data.
*   **Performance:** Efficient for sparse volumes with large uniform regions. The texture-based lookup avoids branching penalties associated with full decomposition.
*   **Strengths:** Low memory footprint for sparse data; fast traversal of empty space; GPU-friendly texture lookups.
*   **Limitations:** Color quantization limits visual fidelity; requires pre-compression of volume data; not suitable for dense volumes where compression ratio is low.

## Background
RLE4 Ray Tracing, also known as Map4 Traversal, originates from the OpenGL-Voxel-Splatting project by sp4cerat [S1, unofficial]. It is an implementation technique for efficient ray casting in sparse volumetric data, leveraging RLE compression to skip empty space. The method is particularly relevant in real-time rendering contexts where memory bandwidth and cache efficiency are critical.

## Sources
[S1] sp4cerat/OpenGL-Voxel-Splatting, https://github.com/sp4cerat/OpenGL-Voxel-Splatting, fetched 2026-10-27 (unofficial)
