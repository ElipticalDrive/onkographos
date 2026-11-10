# GPU Isosurface Raycasting of FCC Datasets

Also called: 6-Direction Cubic Box-Spline on FCC Lattice, FCC Box-Spline Raycasting, FCC Raycasting, Six-Direction Box-Spline Raycasting on FCC, Kim 2012 method

## Overview
This method renders isosurfaces from volume data sampled on a Face-Centered Cubic (FCC) lattice using a six-direction cubic box-spline [S1]. It solves the problem of efficient, high-quality isosurface extraction on non-Cartesian lattices by providing a branch-free, hardware-friendly evaluation of the spline field and its gradient [S1]. Unlike Cartesian raycasting which uses tri-linear or tri-cubic interpolation, this method leverages the isotropic structure of the FCC lattice to achieve better sampling efficiency [S1]. Compared to Body-Centered Cubic (BCC) raycasters, it offers superior performance on FCC datasets by using a symmetric indexing scheme that fits GPU memory hierarchies better than the asymmetric packing required for BCC [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar volume dataset $V$ sampled on the FCC lattice $\mathbb{Z}_{\text{fcc}}$, stored in a four-channel (RGBA) 3D texture [S1]. An isosurface threshold $\tau$ [G].
*   **Output:** A rendered image with surface shading based on the computed normals [S1].

**Data Structures:**
*   **Volume Texture:** A 3D texture with four channels (RGBA). The FCC data is grouped such that the four points $\{j, j+\xi_{110}, j+\xi_{101}, j+\xi_{011}\}$ are stored in the four channels of a single texel at index $(x \gg 1, y \gg 1, z \gg 1)$ [S1].
*   **Transformation Tables:** Pre-computed integer arrays for translation vectors $T[10]$ and rotation/reflection vectors $R[10]$ [S2, unofficial].
*   **Stencil:** An array of 19 integer offsets `stencil[19]` defining the relative positions of the data samples required for evaluation [S2, unofficial].

**Mathematical Background:**
The FCC lattice is defined as:
$$ \mathbb{Z}_{\text{fcc}} := \{ j \in \mathbb{Z}^3 : j(1) + j(2) + j(3) \text{ is even} \} $$
It can be decomposed into four shifted Cartesian lattices scaled by 2:
$$ \mathbb{Z}_{\text{fcc}} = (2\mathbb{Z}^3) \cup (2\mathbb{Z}^3 + \xi_{110}) \cup (2\mathbb{Z}^3 + \xi_{101}) \cup (2\mathbb{Z}^3 + \xi_{011}) $$
where $\xi_{110}=(1,1,0)$, $\xi_{101}=(1,0,1)$, and $\xi_{011}=(0,1,1)$ are the columns of the generator matrix $G_{\text{fcc}}$ [S1].

The reconstruction filter is the six-direction cubic box-spline $M_{\text{fcc}}$, defined by the direction matrix:
$$ \Xi_{\text{fcc}} := \begin{bmatrix} 1 & -1 & 1 & 1 & 0 & 0 \\ 0 & 1 & 0 & 0 & 1 & -1 \\ 0 & 0 & 1 & -1 & 1 & 1 \end{bmatrix} $$
The spline is centered and scaled as $M_{\text{fcc}}(\mathbf{x}) = |\det G_{\text{fcc}}| M_{\Xi_{\text{fcc}}}(\mathbf{x} + \boldsymbol{\xi}_c)$, where $|\det G_{\text{fcc}}| = 2$ and $\boldsymbol{\xi}_c = (1,1,1)^T$ [S1]. The polynomial degree is 3 [S1].

**Algorithm Steps:**
1.  **Parity Determination:** For an input point $\mathbf{p}_{in}$, compute the integer floor $\mathbf{o} = \lfloor \mathbf{p}_{in} \rfloor$. Determine the parity bit $\pi = (o_x + o_y + o_z) \& 1$. This distinguishes between even parity cubes ($\mathbf{o} \in \mathbb{Z}_{\text{fcc}}$) and odd parity cubes ($\mathbf{o} \notin \mathbb{Z}_{\text{fcc}}$) [S2, unofficial].
2.  **Cube Normalization:** Compute the fractional part $\mathbf{p}_{cube} = \mathbf{p}_{in} - \mathbf{o}$. If $\pi = 1$, reflect the fractional part to map the odd parity cube to an even parity equivalent: $\mathbf{p}_{cube} \leftarrow (1,1,1) - \mathbf{p}_{cube}$ [S2, unofficial].
3.  **Tetrahedron Indexing:** Determine the tetrahedron index $i \in \{0, \dots, 4\}$ within the normalized cube by comparing dot products of $\mathbf{p}_{cube}$ with specific vectors [S2, unofficial]:
    *   If $(-1, -1, -1) \cdot \mathbf{p}_{cube} > -1$, then $i=0$.
    *   Else if $(1, 1, -1) \cdot \mathbf{p}_{cube} > 1$, then $i=1$.
    *   Else if $(1, -1, 1) \cdot \mathbf{p}_{cube} > 1$, then $i=2$.
    *   Else if $(-1, 1, 1) \cdot \mathbf{p}_{cube} > 1$, then $i=3$.
    *   Else $i=4$.
4.  **Type and Global Index:** Determine the tetrahedron type $\tau$: if $i=4$ then $\tau=1$ (blue tetrahedron), else $\tau=0$ (red tetrahedron). If $\pi=1$, add 5 to the index: $i \leftarrow i + 5$. The final index $i$ ranges from 0 to 9 [S2, unofficial].
5.  **Transformation:** Apply the pre-computed transformation for index $i$. The local coordinate is transformed as $\mathbf{p}_{local} = \mathbf{p}_{local} \times R[i] + T[i]$, and the origin is updated: $\mathbf{o} \leftarrow \mathbf{o} + T[i]$. Here $R[i]$ is a rotation/reflection vector and $T[i]$ is a translation vector [S2, unofficial].
6.  **Data Fetching:** Fetch 19 samples from the volume texture to avoid branching. For each $k \in \{0, \dots, 18\}$, compute the texture coordinate $\mathbf{fcc} = \mathbf{o} + R[i] \times \text{stencil}[k]$ and fetch the value [S2, unofficial].
7.  **Spline Evaluation:** Compute the spline value $s(\mathbf{p}_{in})$ using the fetched values and the basis function $M_{\text{fcc}}$ evaluated at the transformed local coordinates [S1].
8.  **Normal Computation:** Compute the gradient $\nabla s$ using the analytic property that the directional derivative of a box-spline is a backward difference of a lower-degree box-spline. This requires evaluating three additional quadratic box-splines, reusing the fetched data structure [S2, unofficial].
9.  **Isosurface Detection:** Check if $s(\mathbf{p}_{in}) \ge \tau$ (or sign change relative to previous sample) to detect the surface [G]. Refine the intersection if necessary [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pre-defined constants from sources
T = [
    (0,0,0), (1,1,0), (1,0,1), (0,1,1), (0,0,0),
    (1,1,1), (0,0,1), (0,1,0), (1,0,0), (1,1,1)
]
R = [
    (1,1,1), (-1,-1,1), (-1,1,-1), (1,-1,-1), (1,1,1),
    (-1,-1,-1), (1,1,-1), (1,-1,1), (-1,1,1), (-1,-1,-1)
]
stencil = [
    (0,0,0), (2,0,0), (0,2,0), (0,0,2), (2,1,1), (0,1,1),
    (0,1,-1), (0,-1,-1), (0,-1,1), (1,2,1), (1,0,1), (-1,0,1),
    (-1,0,-1), (1,0,-1), (1,1,2), (1,1,0), (-1,1,0), (-1,-1,0), (1,-1,0)
]

def evaluate_fcc_spline(p_in, volume_tex):
    # 1. Floor and Parity
    o = floor(p_in)
    parity = (o.x + o.y + o.z) & 1
    
    # 2. Fractional part and Normalization
    p_cube = p_in - vec3(o)
    p_local = p_cube
    
    if parity == 1:
        p_cube = vec3(1,1,1) - p_cube
        
    # 3. Tetrahedron Indexing
    if dot(p_cube, vec3(-1,-1,-1)) > -1:
        itet = 0
    elif dot(p_cube, vec3(1,1,-1)) > 1:
        itet = 1
    elif dot(p_cube, vec3(1,-1,1)) > 1:
        itet = 2
    elif dot(p_cube, vec3(-1,1,1)) > 1:
        itet = 3
    else:
        itet = 4
        
    # 4. Type and Global Index
    if itet == 4:
        type = 1
    else:
        type = 0
        
    if parity == 1:
        itet = itet + 5
        
    # 5. Transformation
    # R[itet] and T[itet] are used to transform local coords and origin
    # Note: In GLSL, R is treated as a diagonal matrix for multiplication
    p_local = p_local * vec3(R[itet]) + vec3(T[itet])
    o = o + ivec3(T[itet])
    
    # 6. Data Fetching
    values = []
    for k in range(19):
        fcc = o + ivec3(R[itet]) * stencil[k]
        # Fetch from 4-channel texture
        # The channel selection logic depends on the specific storage mapping
        # The source notes imply a generic FETCH(i) which retrieves the weighted value
        val = fetch_volume_value(fcc, volume_tex, itet, k)
        values.append(val)
        
    # 7. Spline Evaluation
    # The actual polynomial evaluation uses the 19 fetched values
    # and the basis function M_fcc evaluated at p_local
    s = evaluate_box_spline_basis(p_local, values)
    
    return s, type, itet

def fetch_volume_value(fcc_coord, volume_tex, itet, k):
    # The source notes indicate that storing FCC in RGBA requires channel selection
    # i = 2*(x&1) + (y&1) is mentioned as a slow path, but the optimized
    # method uses the transformed stencil directly.
    # The exact channel mapping for the 19 samples is implicit in the stencil
    # and the transformation R[itet].
    # For implementation, one must map the transformed integer coordinate
    # to the correct RGBA channel of the texel at (fcc.x>>1, fcc.y>>1, fcc.z>>1).
    # This is a detail not fully explicit in the text but required for implementation.
    # [G] Assuming standard grouping: channel index is derived from the
    # relative offset of the sample within the 2x2x2 block.
    texel = texelFetch(volume_tex, ivec3(fcc_coord.x >> 1, fcc_coord.y >> 1, fcc_coord.z >> 1), 0)
    # Determine channel based on the specific stencil index and transformation
    # This part requires deriving the channel index from the stencil definition
    # relative to the base lattice point.
    return texel.r # Placeholder for actual channel selection logic
```

## Subtopics

### Quartic Box-Spline on BCC Lattice
A raycasting method using a 7-direction quartic box-spline on the Body-Centered Cubic (BCC) lattice [S2, unofficial]. It computes parity vectors and rotation vectors to transform local coordinates, determining tetrahedron indices via component comparisons. It fetches samples using axis-aligned updates to the origin based on permutation vectors. Compared to the BCC method by Finkbeiner et al., it is 1.2x faster, but 0.4x slower than the Cartesian method by Csébfalvi and Hadwiger [S2, unofficial].

## Operating characteristics
*   **Performance:** Achieves interactive speeds of 20–70 fps for isosurface generation on datasets like 128x128x128 [S1].
*   **Speedup:** Pre-processing (quasi-interpolation and min/max cell construction) is 20 to 30-fold faster using OpenCL kernels [S1].
*   **Comparison:** On FCC datasets, it is 2.4x faster than the BCC raycaster by Finkbeiner et al., but 0.4x slower (i.e., slower) than the Cartesian lattice method by Sigg & Hadwiger [S2, unofficial].
*   **Memory:** Storing FCC data in a four-channel 3D texture is cache-friendly compared to splitting into four Cartesian lattices, but requires channel selection which can slow down performance if not optimized [S2, unofficial].
*   **Strengths:** Fast, accurate, and stable evaluation; efficient empty-space skipping; high-quality isosurfaces due to C1 continuity of the box-spline [S1].
*   **Limitations:** Slower than Cartesian methods; channel selection overhead in GPU memory access [S2, unofficial].

## Background
The method was proposed by Minho Kim in 2012 [S1]. It builds on the theory of box-splines on non-Cartesian lattices, specifically the six-direction cubic box-spline on the FCC lattice first proposed by Entezari and later investigated by Kim et al. [S1]. The FCC lattice is optimal for low-rate sampling, offering better sampling efficiency than the Cartesian lattice [S1]. The method addresses the lack of GPU raycasters for FCC datasets at the time, providing a practical reconstruction scheme that balances quality and performance [S1].

## Sources
[S1] GPU Isosurface Raycasting of FCC Datasets, https://minho-kim-uos.github.io/papers/kim12gpu.pdf, fetched 2026-10-29
[S2] GPU Isosurface Raycasting of Volume Datasets Based On Box-Splines, https://minho-kim-uos.github.io/papers/birs13.pdf, fetched 2026-10-29 (unofficial)
