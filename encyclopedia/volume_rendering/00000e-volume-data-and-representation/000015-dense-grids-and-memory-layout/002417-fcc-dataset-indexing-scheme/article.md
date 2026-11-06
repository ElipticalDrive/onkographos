# FCC Dataset Indexing Scheme
Also called: Four-channel 3D Texture Storage for FCC

## Overview
The FCC Dataset Indexing Scheme is a method for storing and accessing Four-Cubic Close-Packed (FCC) volume datasets on graphics hardware that lacks native support for non-Cartesian lattices. It solves the problem of efficiently mapping FCC lattice points to standard 3D textures by grouping points into a four-channel (RGBA) Cartesian structure. It sets itself apart from the [1-channel scheme](3064-3d-texture-storage-and-sampling.md) by halving the memory footprint, though it incurs significant performance penalties due to hardware limitations in selective channel fetching.

## How it works
**Inputs:** An FCC dataset embedded in a Cartesian lattice of size $N_x \times N_y \times N_z$.
**Outputs:** A four-channel 3D texture of size $W \times H \times D \times 4$, where $W = \lceil N_x/2 \rceil$, $H = \lceil N_y/2 \rceil$, and $D = \lceil N_z/2 \rceil$.

**Data Structures:**
The texture stores values in a 4-channel format (e.g., `vec4` in GLSL). Each texel at coordinates $(u, v, w)$ and channel $c \in \{0, 1, 2, 3\}$ corresponds to a specific FCC lattice point.

**Mapping Logic:**
The FCC lattice $\mathbb{Z}_{\text{fcc}}$ is defined by grouping points as:
$$ \mathbb{Z}_{\text{fcc}} = \{j, j + \boldsymbol{\xi}_{110}, j + \boldsymbol{\xi}_{101}, j + \boldsymbol{\xi}_{011} : j \in 2\mathbb{Z}\} $$
where $\boldsymbol{\xi}_{110} = (1,1,0)$, $\boldsymbol{\xi}_{101} = (1,0,1)$, and $\boldsymbol{\xi}_{011} = (0,1,1)$. Note that for any $j \in \mathbb{Z}_{\text{fcc}}$, the sum of coordinates $j(1)+j(2)+j(3)$ is always even.

For an FCC index $j$, the **parity vector** $\mathbf{p}_j$ is defined as:
$$ \mathbf{p}_j := (j(1) \bmod 2, j(2) \bmod 2, j(3) \bmod 2) $$
Because the sum of coordinates is even, $\mathbf{p}_j$ can only take values from the set $\{\mathbf{0}, \boldsymbol{\xi}_{110}, \boldsymbol{\xi}_{101}, \boldsymbol{\xi}_{011}\}$. The channel index $c$ is derived from the parity vector as:
$$ c = 2p_j(1) + p_j(2) $$
This maps the parity to a unique integer in $\{0, 1, 2, 3\}$.

The texture coordinates $(u, v, w)$ are calculated by integer division (floor) of the FCC coordinates by 2:
$$ u = \lfloor j(1)/2 \rfloor, \quad v = \lfloor j(2)/2 \rfloor, \quad w = \lfloor j(3)/2 \rfloor $$

**Implementation Steps:**
1.  Given an FCC lattice coordinate $j = (j_x, j_y, j_z)$, compute the texture coordinates:
    *   $u = j_x \gg 1$ (bitwise right shift by 1)
    *   $v = j_y \gg 1$
    *   $w = j_z \gg 1$
2.  Compute the channel index $c$:
    *   Extract the least significant bits of $j_x$ and $j_y$: $b_x = j_x \& 1$, $b_y = j_y \& 1$.
    *   Calculate $c = (b_x \ll 1) | b_y$.
3.  Fetch the texel from the 3D texture at $(u, v, w)$.
4.  Select the value at channel $c$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fetch_fcc_value(tex, j_x, j_y, j_z):
    # Calculate texture coordinates
    u = j_x >> 1
    v = j_y >> 1
    w = j_z >> 1
    
    # Calculate channel index from parity bits
    # j_x & 1 is the LSB of x, j_y & 1 is the LSB of y
    channel = ((j_x & 1) << 1) | (j_y & 1)
    
    # Fetch all four channels
    texel = texel_fetch(tex, u, v, w)
    
    # Select the specific channel
    # Note: Hardware does not support direct channel indexing efficiently.
    # See Pitfalls for the required workaround.
    return texel[channel]
```

**Pitfalls:**
*   **Selective Fetching:** Current graphics hardware does not allow efficient selective fetching of a single texture channel. Fetching all four channels and using a conditional or dot product to select one results in massive performance degradation.
*   **Workaround:** The access must be rewritten as a dot product with a selector vector:
    ```glsl
    vec4 texel = texelFetch(tex, u, v, w);
    vec4 selector = vec4(channel == 0, channel == 1, channel == 2, channel == 3);
    float value = dot(texel, selector);
    ```
*   **Boundary Conditions:** The mapping assumes the FCC lattice is properly aligned within the Cartesian bounds. The resulting texture dimensions are $\lceil N_x/2 \rceil \times \lceil N_y/2 \rceil \times \lceil N_z/2 \rceil$.

## Subtopics

### BCC Dataset Indexing Scheme
A scheme to convert a Body-Centered Cubic (BCC) dataset to a two-channel 3D texture. It uses a similar bitwise mapping logic but with a different parity structure. The access pattern is `texelFetch(texid, j.x >> 1, j.y >> 1, j.z >> 1)[j.x & 1]`. It requires half the memory footprint of a standard 1-channel Cartesian storage for BCC data, analogous to the FCC 4-channel scheme.

## Operating characteristics
**Time Complexity:** $O(1)$ per sample access. The mapping involves bitwise operations which are very fast. However, the hardware limitation on channel selection forces a full 4-channel fetch and a dot product, increasing the effective latency compared to a direct 1-channel fetch.
**Memory Complexity:** The 4-channel scheme requires half the memory footprint of a 1-channel Cartesian scheme for the same FCC dataset density. The 1-channel scheme would require storing empty channels or padding, effectively doubling the memory usage for the same number of valid samples.
**Performance:**
*   **MRI-Head Dataset:** 4-channel scheme achieved 33.6 fps, while the 1-channel scheme achieved 65.2 fps.
*   **VisMale Dataset:** 4-channel scheme achieved 31.5 fps, while the 1-channel scheme achieved 68.9 fps.
*   **CT-Head Dataset:** 4-channel scheme achieved 25.3 fps, while the 1-channel scheme achieved 54.6 fps.
*   **Carp Dataset:** 4-channel scheme achieved 7.3 fps, while the 1-channel scheme achieved 22.3 fps.
The 4-channel scheme is consistently slower due to the channel selection overhead, despite the memory efficiency. It is suitable when memory bandwidth or capacity is the bottleneck, but not when raw rendering speed is the primary constraint.

## Background
The scheme was proposed by Kim et al. in "GPU Isosurface Raycasting of FCC Datasets" [S1]. It addresses the lack of hardware support for non-Cartesian volume datasets. Previous attempts to load FCC layers into separate textures or shift layers were deemed inefficient due to overhead. The method leverages the mathematical structure of the FCC lattice to pack it into standard 3D textures.

## Sources
[S1] GPU Isosurface Raycasting of FCC Datasets, https://minho-kim-uos.github.io/papers/kim12gpu.pdf, fetched 2026-10-29
