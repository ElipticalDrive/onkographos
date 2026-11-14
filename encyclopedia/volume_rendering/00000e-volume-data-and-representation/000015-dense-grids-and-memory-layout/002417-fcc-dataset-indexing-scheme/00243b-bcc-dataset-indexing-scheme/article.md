# BCC Dataset Indexing Scheme

Also called: BCC to 2-channel 3D texture mapping

## Overview
The BCC Dataset Indexing Scheme is a method for storing a Body-Centered Cubic (BCC) volume dataset into a standard two-channel 3D texture, enabling efficient hardware-accelerated raycasting without custom memory layouts [S1]. It distinguishes itself from the FCC scheme by utilizing a 2-channel texture instead of 4-channel, reducing memory bandwidth requirements for datasets with lower per-voxel data density or when only scalar values are stored [S1].

## How it works
**Inputs:** A BCC dataset defined on a grid with coordinates $(j_x, j_y, j_z)$, and a 3D texture `texid` with dimensions $(W, H, D)$ and two channels (e.g., RGBA where only RG or RA are used, or a custom 2-channel format).
**Outputs:** The value of the BCC voxel at index $(j_x, j_y, j_z)$ retrieved from the texture.

**Data Structures:**
The BCC lattice points are mapped to a Cartesian grid. The BCC lattice can be viewed as a subset of a Cartesian grid where points satisfy specific parity constraints. The mapping compresses the BCC points into a 3D texture of half the resolution in each dimension (roughly), packed into 2 channels.

**Access Pattern:**
The core operation is a bitwise mapping from the BCC coordinate $(j_x, j_y, j_z)$ to the texture coordinate $(u, v, w)$ and channel index $c$.

1.  Compute the texture coordinates by right-shifting the BCC coordinates by 1:
    $$ u = j_x \gg 1 $$
    $$ v = j_y \gg 1 $$
    $$ w = j_z \gg 1 $$
2.  Determine the channel index $c$ based on the least significant bits of the coordinates. The source provides a specific formula for FCC, but for BCC, it states:
    $$ c = j_x \& 1 $$
    *Note: The source text for BCC access is explicitly given as `texelFetch(texid, j.x >> 1, j.y >> 1, j.z >> 1)[x & 1]` [S1]. Here `x` refers to `j.x`.*

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_bcc_value(texid, jx, jy, jz):
    """
    Retrieves the value of a BCC voxel at integer coordinates (jx, jy, jz).
    texid: The 3D texture object.
    jx, jy, jz: Integer coordinates of the BCC lattice point.
    Returns: The scalar value stored in the texture.
    """
    # Map BCC coordinates to texture coordinates
    # Right shift by 1 is equivalent to integer division by 2 for positive integers
    u = jx >> 1
    v = jy >> 1
    w = jz >> 1
    
    # Determine channel index based on the LSB of jx
    # The source specifies: [x & 1] where x is j.x
    channel_index = jx & 1
    
    # Fetch the texel
    # Assuming texelFetch returns a vector-like object with accessible channels
    texel = texelFetch(texid, u, v, w)
    
    # Return the value from the selected channel
    # In GLSL, this would be texel.r or texel.g depending on channel_index
    if channel_index == 0:
        return texel.r
    else:
        return texel.g
```

**Parameters:**
- Texture channels: 2 (e.g., RG or RA).
- Bitwise operations: Supported in GLSL 1.30 and later [S1].

**Edge Cases and Pitfalls:**
- **Negative Coordinates:** The bitwise right shift (`>>`) and AND (`&`) operators behave differently for negative integers in some languages/hardware. The source assumes positive indices or two's complement behavior consistent with GLSL. Implementers must ensure coordinate systems align with the hardware's bitwise semantics [G].
- **Boundary Conditions:** The texture dimensions must be sufficient to hold the mapped coordinates. If the BCC grid size is $N \times N \times N$, the texture size should be $\lceil N/2 \rceil \times \lceil N/2 \rceil \times \lceil N/2 \rceil$.

## Subtopics

### BCC Grid Extension for 4D
An extension of BCC grids for 4D volume rendering by Neophytou and Mueller [S2, unofficial]. It uses a 3D hyperslicer to extract 3D volumes from the 4D dataset for shaded and semi-transparent visualization, followed by occlusion ordering on the extracted volumes [S2, unofficial]. This allows rendering of time-varying or multi-dimensional data using standard 3D BCC indexing techniques on slices.

## Operating characteristics
**Time and Memory Complexity:**
- Memory: Uses 2 channels per voxel in the texture, compared to 4 for FCC. This reduces memory usage by approximately 50% relative to the FCC scheme for equivalent data density, assuming similar packing efficiency [S1].
- Time: Access is $O(1)$ with bitwise operations, which are very fast on modern GPUs [S1].

**Strengths and Limitations:**
- **Strengths:** Efficient hardware support via standard 3D textures; low memory footprint; fast bitwise indexing [S1].
- **Limitations:** Only suitable for BCC lattices; requires careful handling of coordinate mapping; limited to 2 channels of data per voxel in this specific scheme [S1].

**When to Use:**
Use when rendering BCC datasets and memory bandwidth is a bottleneck, or when only scalar data is stored per voxel. Use FCC if higher data density per texture channel is needed or if the dataset is naturally FCC [S1].

## Background
The scheme is described in the context of GPU isosurface raycasting of FCC and BCC datasets [S1]. It leverages the efficiency of bitwise operations available in GLSL 1.30+ to map non-Cartesian grids to Cartesian textures. The BCC extension for 4D rendering was introduced by Neophytou and Mueller [S2, unofficial].

## Sources
[S1] GPU Isosurface Raycasting of FCC Datasets, https://minho-kim-uos.github.io/papers/kim12gpu.pdf, fetched 2026-10-29
[S2] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
