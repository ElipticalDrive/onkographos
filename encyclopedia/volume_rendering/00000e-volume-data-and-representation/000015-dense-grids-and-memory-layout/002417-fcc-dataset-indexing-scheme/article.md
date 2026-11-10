# FCC Dataset Indexing Scheme
Also called: Four-channel 3D Texture Storage for FCC

## Overview
The FCC Dataset Indexing Scheme is a method for storing Four-Cubic Closest (FCC) lattice volume data in standard hardware-supported 3D textures [S1]. It solves the problem of rendering non-Cartesian (FCC) datasets on graphics hardware that lacks native support for non-Cartesian sampling, avoiding the overhead of shifting layers or loading separate textures for each lattice subset [S1]. It sets itself apart from sibling methods like [3D Texture Representation (CPU/GPU Indexing)](2393-3d-texture-representation-cpu-gpu-indexing.md) by explicitly mapping a 3D non-Cartesian lattice into a 4-channel Cartesian texture structure, trading memory efficiency for potential hardware sampling penalties [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** An FCC dataset embedded in a Cartesian lattice of dimensions $N_x \times N_y \times N_z$.
*   **Output:** A four-channel (RGBA) 3D texture of dimensions $\lceil N_x/2 \rceil \times \lceil N_y/2 \rceil \times \lceil N_z/2 \rceil \times 4$ [S1].

**Data Structures:**
The data is stored in a 3D texture object. Each texel at coordinates $(u, v, w)$ contains four scalar values (R, G, B, A), corresponding to the four FCC lattice points mapped to that Cartesian coordinate.

**Mapping Logic:**
The FCC lattice $\mathbb{Z}_{\text{fcc}}$ is grouped such that each group consists of a base point $j \in 2\mathbb{Z}$ and three offset points:
$$ \mathbb{Z}_{\text{fcc}} = \{j, j + \xi_{110}, j + \xi_{101}, j + \xi_{011} : j \in 2\mathbb{Z}\} $$
where $\xi_{110}, \xi_{101}, \xi_{011}$ represent the specific FCC offsets [S1].

For any FCC index $j$, the parity vector $\mathbf{p}_j$ is defined as:
$$ \mathbf{p}_j := (j(1) \bmod 2, j(2) \bmod 2, j(3) \bmod 2) $$
Since $j \in \mathbb{Z}_{\text{fcc}}$, the sum $j(1)+j(2)+j(3)$ is always even. Consequently, $\mathbf{p}_j$ is restricted to the set $\{\mathbf{0}, \xi_{110}, \xi_{101}, \xi_{011}\}$ [S1].

The channel index $c$ is calculated as:
$$ c = 2p_j(1) + p_j(2) $$
This ensures $c \in \{0, 1, 2, 3\}$ [S1].

The mapping from an FCC index $j$ to the 3D texture coordinates $(u, v, w, c)$ is:
$$ j \mapsto (\lfloor j(1)/2 \rfloor, \lfloor j(2)/2 \rfloor, \lfloor j(3)/2 \rfloor, 2p_j(1) + p_j(2)) $$
This converts the FCC dataset into a four-channel Cartesian dataset of size $\lceil N_x/2 \rceil \times \lceil N_y/2 \rceil \times \lceil N_z/2 \rceil \times 4$ [S1].

**Pseudocode:**
The mapping is implemented using bitwise operations for efficiency [S1].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Assuming j is a tuple (x, y, z) representing the FCC lattice coordinates
# texid is the texture object handle
# texelFetch is a standard GPU texture fetch operation returning a vec4

def fetch_fcc_value(texid, j_x, j_y, j_z):
    # Calculate base texture coordinates
    u = j_x >> 1
    v = j_y >> 1
    w = j_z >> 1
    
    # Calculate channel index using parity bits
    # p_j(1) is j_x & 1
    # p_j(2) is j_y & 1
    channel_index = ((j_x & 1) << 1) | (j_y & 1)
    
    # Fetch the texel
    texel = texelFetch(texid, u, v, w)
    
    # Return the specific channel
    return texel[channel_index]
```

**Edge Cases and Pitfalls:**
1.  **Hardware Channel Fetching:** Current graphics hardware does not allow selective fetching of a single texture channel without fetching all four [S1]. To retrieve a specific channel, one must fetch all four and use a dot product with a selector vector: `dot(texelFetch(tex, x, y, z), vec4(idx==0, idx==1, idx==2, idx==3))` [S1]. This results in massive performance degradation compared to a direct single-channel fetch [S1].
2.  **Boundary Conditions:** The dimensions of the output texture are determined by $\lceil N/2 \rceil$. Ensure that the input lattice dimensions are handled correctly if they are odd, as the ceiling function accounts for the last partial group [S1].

## Subtopics

### BCC Dataset Indexing Scheme
A scheme to convert a Body-Centered Cubic (BCC) dataset to a two-channel 3D texture, accessed similarly to the FCC scheme [S1]. The access pattern is `texelFetch(texid, j.x >> 1, j.y >> 1, j.z >> 1)[x & 1]` [S1]. It is chosen when the underlying data follows a BCC lattice structure rather than FCC [S1].

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** The four-channel scheme requires half the memory footprint of a one-channel scheme (which would store the same data in a single-channel texture of double the resolution) [S1].
*   **Time:** The index mapping involves bitwise operations, which are computationally cheap [S1]. However, the inability to selectively fetch channels forces a full 4-channel fetch and a dot product operation, introducing significant computational overhead per sample [S1].

**Measured Performance:**
Performance comparisons on an MRI-Head dataset show the 4-channel scheme achieving 33.6 fps, while a 1-channel scheme achieved 65.2 fps [S1]. The speed-up varies by dataset size, with larger datasets showing more proportional gains from cache efficiency, but the 4-channel scheme consistently suffers from the channel-fetching penalty [S1].

**Strengths and Limitations:**
*   **Strengths:** Compact storage (half the memory of 1-channel equivalent); reduces cache misses due to compactness; avoids overhead of multiple textures or layer shifting [S1].
*   **Limitations:** Severe performance degradation on current hardware due to non-selective channel fetching; requires bitwise logic for indexing [S1].

**When to Use:**
Use when memory bandwidth is a critical constraint and the hardware supports efficient bitwise operations, but be aware of the rendering performance hit from channel selection. Avoid if maximum rendering speed is the primary goal and memory is not a bottleneck, in which case a 1-channel scheme or other indexing method might be preferable [S1].

## Background
The FCC Dataset Indexing Scheme was proposed in the context of GPU isosurface raycasting for non-Cartesian datasets [S1]. It addresses the limitations of previous approaches, such as loading FCC layers into separate textures (high overhead) or using rotated 2D Cartesian lattices (inefficient) [S1]. The method leverages the structure of the FCC lattice to map it efficiently into standard 3D texture hardware [S1].

## Sources
[S1] GPU Isosurface Raycasting of FCC Datasets, https://minho-kim-uos.github.io/papers/kim12gpu.pdf, fetched 2026-10-29
