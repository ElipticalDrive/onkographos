# FCC Dataset Indexing Scheme
Also called: Four-channel 3D Texture Storage for FCC

## Overview
The FCC Dataset Indexing Scheme is a method for storing Face-Centered Cubic (FCC) volume datasets as four-channel (RGBA) 3D textures on standard graphics hardware [S1]. It solves the problem that most GPU texture units lack native support for non-Cartesian (e.g., FCC or BCC) lattice sampling, which would otherwise require complex coordinate transformations or multiple texture lookups [S1]. By mapping the FCC lattice points into a compact Cartesian RGBA texture, the scheme reduces memory footprint by half compared to a naive one-channel storage approach, at the cost of increased computational overhead during sampling due to hardware limitations on selective channel fetching [S1].

## How it works
**Inputs:**
*   An FCC dataset defined on a Cartesian lattice of size $N_x \times N_y \times N_z$ [S1].
*   A 3D texture object (`texid`) allocated with internal format `RGBA32F` (or equivalent 4-channel format) and dimensions $W \times H \times D$ [G].

**Outputs:**
*   A 3D texture containing the FCC data, accessible via standard `texelFetch` operations [S1].

**Data Structures:**
*   **Texture Dimensions:** The texture dimensions are derived from the input lattice size:
    $$ W = \lfloor (N_x + 1) / 2 \rfloor, \quad H = \lfloor (N_y + 1) / 2 \rfloor, \quad D = \lfloor (N_z + 1) / 2 \rfloor $$ [S1].
*   **Channel Mapping:** Each texel $(x, y, z)$ in the texture stores four FCC lattice points [S1]. The channel index $c \in \{0, 1, 2, 3\}$ corresponds to specific parity offsets within the FCC lattice [S1].

**Algorithm:**
1.  **Lattice Grouping:** The FCC lattice $\mathbb{Z}_{\text{fcc}}$ is grouped into sets of four points based on their parity [S1]. For any base point $j \in 2\mathbb{Z}$ (where coordinates are even integers), the group is defined as:
    $$ \{j, j + \boldsymbol{\xi}_{110}, j + \boldsymbol{\xi}_{101}, j + \boldsymbol{\xi}_{011}\} $$ [S1].
2.  **Parity Calculation:** For a given FCC index $j$, calculate the parity vector $\mathbf{p}_j$:
    $$ \mathbf{p}_j = (j(1) \bmod 2, j(2) \bmod 2, j(3) \bmod 2) $$ [S1].
    Since $j \in \mathbb{Z}_{\text{fcc}}$, the sum $j(1)+j(2)+j(3)$ is always even [S1]. Consequently, $\mathbf{p}_j$ is restricted to $\{\mathbf{0}, \boldsymbol{\xi}_{110}, \boldsymbol{\xi}_{101}, \boldsymbol{\xi}_{011}\}$ [S1].
3.  **Channel Index Derivation:** Compute the channel index $c$ using the first two components of the parity vector:
    $$ c = 2p_j(1) + p_j(2) $$ [S1].
    This maps the four possible parity states to integers $\{0, 1, 2, 3\}$ [S1].
4.  **Texture Coordinate Mapping:** Map the FCC index $j$ to the texture coordinate $(x, y, z)$ and channel $c$:
    $$ x = \lfloor j(1) / 2 \rfloor, \quad y = \lfloor j(2) / 2 \rfloor, \quad z = \lfloor j(3) / 2 \rfloor $$ [S1].
    $$ c = 2(j(1) \bmod 2) + (j(2) \bmod 2) $$ [S1].
5.  **Bitwise Implementation:** In GLSL (version 1.30+), this mapping is implemented efficiently using bitwise shifts and masks [S1]. The following Python-like pseudocode implements the lookup:
    ```python
    def fetch_fcc(texid, j):
        # j is a tuple or array of three integers (x, y, z)
        tex_coord = (j[0] >> 1, j[1] >> 1, j[2] >> 1)
        channel = ((j[0] & 1) << 1) | (j[1] & 1)
        texel = texel_fetch(texid, tex_coord)  # Returns vec4 (RGBA)
        return texel[channel]
    ```

**Parameters:**
*   Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Hardware Channel Fetching:** Current graphics hardware does not support selective fetching of a single texture channel without loading all four [S1]. To fetch only channel $c$, the shader must fetch all four channels and perform a dot product with a selection vector:
    ```python
    def fetch_fcc_safe(texid, j, target_channel):
        tex_coord = (j[0] >> 1, j[1] >> 1, j[2] >> 1)
        texel = texel_fetch(texid, tex_coord)
        selection = [1.0 if i == target_channel else 0.0 for i in range(4)]
        return dot(texel, selection)
    ```
    This results in massive performance degradation compared to a native single-channel fetch [S1].
*   **Boundary Conditions:** The texture dimensions $W, H, D$ must be calculated carefully to ensure all FCC points fit within the allocated texture space [S1]. Points with odd coordinates in the original lattice are mapped to the last valid index in the respective dimension [S1].

## Subtopics

### [BCC Dataset Indexing Scheme](3232-bcc-dataset-indexing-scheme.md)
A scheme to convert a BCC dataset to a two-channel 3D texture, accessed similarly to the FCC scheme.

## Operating characteristics
**Time Complexity:**
*   Index mapping is $O(1)$ using bitwise operations [S1].
*   Texture sampling is $O(1)$ but involves fetching 4 channels and a dot product, increasing the constant factor significantly compared to standard 1-channel textures [S1].

**Memory Complexity:**
*   The scheme requires $W \times H \times D \times 4$ texels [S1].
*   Compared to a naive 1-channel storage of the full FCC lattice (which would require storing empty space or duplicating data), this scheme reduces the memory footprint by half [S1].

**Performance:**
*   Benchmarks on MRI-Head dataset show the 4-channel scheme achieves 33.6 fps, while a 1-channel scheme achieves 65.2 fps [S1].
*   The performance penalty is attributed to the inability to selectively fetch channels on current hardware, forcing the dot-product workaround [S1].
*   The scheme reduces cache misses due to compact storage but incurs computational overhead from the index mapping and multi-channel fetch [S1].

**Strengths:**
*   Compact memory usage (half of 1-channel scheme) [S1].
*   Compatible with standard 3D texture hardware [S1].
*   Efficient bitwise index calculation [S1].

**Limitations:**
*   Significant performance degradation due to hardware limitations on channel selection [S1].
*   Not suitable for real-time applications where the 1-channel scheme's memory overhead is acceptable but performance is critical [S1].

## Background
The FCC Dataset Indexing Scheme was proposed by Kim et al. in 2012 as part of their work on GPU isosurface raycasting of FCC datasets [S1]. The work addresses the limitations of existing hardware in handling non-Cartesian volume data directly [S1]. Prior approaches, such as loading separate textures for each layer or using complex coordinate shifts, were deemed inefficient due to overhead [S1]. The scheme builds on the mathematical properties of the FCC lattice, specifically its grouping into subsets of four points with specific parity relationships [S1].

## Sources
[S1] GPU Isosurface Raycasting of FCC Datasets, https://minho-kim-uos.github.io/papers/kim12gpu.pdf, fetched 2026-10-29
