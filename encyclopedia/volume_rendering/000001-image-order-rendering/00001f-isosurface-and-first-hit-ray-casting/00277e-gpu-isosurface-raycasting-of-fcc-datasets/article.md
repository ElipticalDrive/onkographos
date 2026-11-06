# GPU Isosurface Raycasting of FCC Datasets

Also called: 6-Direction Cubic Box-Spline on FCC Lattice, FCC Box-Spline Raycasting, FCC Raycasting, Six-Direction Box-Spline Raycasting on FCC, Kim 2012 method

## Overview
This method renders isosurfaces from volume data sampled on a Face-Centered Cubic (FCC) lattice using a six-direction cubic box-spline [S1]. It solves the problem of efficiently evaluating C1 continuous splines and their gradients on non-Cartesian lattices on the GPU, avoiding conditional branches and minimizing texture fetch overhead [S1]. It is distinguished from Cartesian raycasting by its use of the FCC lattice for better sampling efficiency at low rates, and from BCC raycasting by a symmetric indexing scheme that is more cache-friendly along the z-axis [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A scalar field $V$ defined on the FCC lattice $\mathbb{Z}_{\text{fcc}}$, stored in a 3D texture. An isovalue $\tau$.
- Output: The color of the first hit isosurface for each pixel.

**Data Structures:**
- **FCC Lattice Storage:** The FCC dataset is stored in a four-channel (RGBA) 3D texture. The lattice is decomposed into four Cartesian sub-lattices: $\mathbb{Z}_{\text{fcc}} = (2\mathbb{Z}^3) \cup (2\mathbb{Z}^3 + \xi_{110}) \cup (2\mathbb{Z}^3 + \xi_{101}) \cup (2\mathbb{Z}^3 + \xi_{011})$, where $\xi_{110}=(1,1,0)^T$, $\xi_{101}=(1,0,1)^T$, $\xi_{011}=(0,1,1)^T$ [S1].
- **Transformation Tables:** Pre-computed integer arrays `T[10]` and `R[10]` store the translation offsets and rotation/reflection signs for each of the 10 tetrahedral types (0-9) [S2, unofficial].
- **Stencil:** A fixed array `stencil[19]` containing the relative integer offsets of the 19 data points required to evaluate the spline for any tetrahedron type [S2, unofficial].

**Mathematical Background:**
The six-direction box-spline $M_{\text{fcc}}$ is defined by the direction matrix:
$$
\Xi_{\text{fcc}} := \begin{bmatrix} 1 & -1 & 1 & 1 & 0 & 0 \\ 0 & 1 & 0 & 0 & 1 & -1 \\ 0 & 0 & 1 & -1 & 1 & 1 \end{bmatrix}
$$
The center shift is $\xi_c = [1, 1, 1]^T$ [S1]. The spline is given by $M_{\text{fcc}}(\mathbf{x}) = 2 M_{\Xi_{\text{fcc}}}(\mathbf{x} + \xi_c)$ [S1]. The polynomial degree is 3 [S1].

**Algorithm Steps:**
1.  **Ray Marching:** For each pixel, march along the ray. At each step, obtain the current 3D position $\mathbf{p}_{in}$ in the volume domain.
2.  **Cube Identification:** Compute the integer cube corner $\mathbf{f} = \lfloor \mathbf{p}_{in} \rfloor$ and the fractional part $\mathbf{p}_{cube} = \mathbf{p}_{in} - \mathbf{f}$ [S2, unofficial].
3.  **Parity Check:** Determine the parity bit $\pi$ of the cube corner:
    $$ \pi = (\mathbf{f} \cdot [1, 1, 1]^T) \& 1 $$
    If $\pi=1$, the cube is of odd parity (not in $\mathbb{Z}_{\text{fcc}}$) [S1][S2, unofficial].
4.  **Normalization:** If $\pi=1$, reflect the fractional part to map the odd parity cube to an even parity equivalent:
    $$ \mathbf{p}_{cube} \leftarrow [1, 1, 1]^T - \mathbf{p}_{cube} $$
    Store the original fractional part in $\mathbf{p}_{local}$ for later transformation [S2, unofficial].
5.  **Tetrahedron Indexing:** Determine the tetrahedron index $i \in \{0, \dots, 4\}$ by comparing $\mathbf{p}_{cube}$ against four knot planes:
    - If $(-1, -1, -1) \cdot \mathbf{p}_{cube} > -1$, then $i=0$.
    - Else if $(1, 1, -1) \cdot \mathbf{p}_{cube} > 1$, then $i=1$.
    - Else if $(1, -1, 1) \cdot \mathbf{p}_{cube} > 1$, then $i=2$.
    - Else if $(-1, 1, 1) \cdot \mathbf{p}_{cube} > 1$, then $i=3$.
    - Else $i=4$.
6.  **Type and Final Index:** Determine the spline type $\tau$ and the final index $i_{tet}$:
    - If $i=4$, $\tau=1$ (blue tetrahedron $\tau_4$); else $\tau=0$ (red tetrahedron $\tau_0$).
    - If $\pi=1$, $i_{tet} = i + 5$; else $i_{tet} = i$.
    This maps the point to one of the 10 tetrahedral types [S2, unofficial].
7.  **Transformation:** Retrieve the transformation matrix signs `R[i_tet]` and offset `T[i_tet]` from the pre-computed tables. Transform the local position and update the origin:
    $$ \mathbf{p}_{local} \leftarrow \mathbf{p}_{local} \odot \mathbf{R}[i_{tet}] + \mathbf{T}[i_{tet}] $$
    $$ \mathbf{f} \leftarrow \mathbf{f} + \mathbf{T}[i_{tet}] $$
    where $\odot$ denotes component-wise multiplication [S2, unofficial].
8.  **Data Fetching:** Fetch 19 scalar values from the volume texture using the transformed stencil. For each $k \in \{0, \dots, 18\}$:
    $$ \mathbf{fcc}_k = \mathbf{f} + \mathbf{R}[i_{tet}] \odot \text{stencil}[k] $$
    Fetch the value $V_k$ from the texture at $\mathbf{fcc}_k$. Note that because the data is stored in a 4-channel texture representing the 4 sub-lattices, the fetch must select the correct channel based on the parity of the coordinates [S1][S2, unofficial]. Specifically, if the texture stores the decomposition $\mathbb{Z}_{\text{fcc}} = L_0 \cup L_1 \cup L_2 \cup L_3$, the channel is determined by which sub-lattice the coordinate belongs to [S2, unofficial].
9.  **Spline Evaluation:** Compute the spline value $s(\mathbf{p}_{in})$ using the fetched values and the basis function $M_{\text{fcc}}$ evaluated at the transformed local coordinates. The evaluation uses only the two base tetrahedra types $\tau_0$ and $\tau_4$ due to symmetry [S1].
10. **Isosurface Test:** Check if $s(\mathbf{p}_{in}) \ge \tau$ (or sign change). If hit, compute the gradient $\nabla s$ using the property that directional derivatives of box-splines are backward differences of lower-degree box-splines, requiring evaluation of three additional quadratic box-splines [S2, unofficial]. Compute shading and return color.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pre-computed tables from [S2, unofficial]
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

def fetch_fcc(tex, x, y, z):
    # Helper to fetch from 4-channel texture based on sub-lattice index
    # Sub-lattices are determined by (x&1, y&1, z&1) mapped to 0-3
    # This is a placeholder for the specific channel selection logic described in [S2, unofficial]
    # i = 2*(x&1) + (y&1) is mentioned as a simplification, but full 3D parity is needed
    # for the 4 sub-lattices: 2Z^3, 2Z^3+(1,1,0), 2Z^3+(1,0,1), 2Z^3+(0,1,1)
    # The exact mapping depends on the texture layout implementation.
    pass

def evaluate_spline(p_in, tex):
    f = floor(p_in)
    p_cube = p_in - vec3(f)
    
    # Parity
    parity = (f.x + f.y + f.z) & 1
    
    p_local = p_cube
    
    # Normalize for odd parity
    if parity == 1:
        p_cube = vec3(1,1,1) - p_cube
        
    # Determine tetrahedron index i (0-4)
    if dot(p_cube, vec3(-1,-1,-1)) > -1:
        i = 0
    elif dot(p_cube, vec3(1,1,-1)) > 1:
        i = 1
    elif dot(p_cube, vec3(1,-1,1)) > 1:
        i = 2
    elif dot(p_cube, vec3(-1,1,1)) > 1:
        i = 3
    else:
        i = 4
        
    # Determine type and final index
    if i == 4:
        type_val = 1
    else:
        type_val = 0
        
    if parity == 1:
        i_tet = i + 5
    else:
        i_tet = i
        
    # Transform
    r_vec = R[i_tet]
    t_vec = T[i_tet]
    
    p_local = p_local * r_vec + t_vec # Component-wise mult
    f = f + t_vec
    
    # Fetch 19 samples
    values = []
    for k in range(19):
        stencil_vec = stencil[k]
        fcc_coord = f + r_vec * stencil_vec # Component-wise mult
        val = fetch_fcc(tex, fcc_coord.x, fcc_coord.y, fcc_coord.z)
        values.append(val)
        
    # Evaluate spline using values and p_local
    # The actual polynomial evaluation of M_fcc for tau_0 or tau_4
    # is omitted here as it is a standard polynomial evaluation
    # using the 19 coefficients and the transformed local coordinates.
    s = evaluate_polynomial(values, p_local, type_val)
    
    return s
```

**Parameters:**
- Isovalue $\tau$: Scene-dependent.
- Ray step size: Determined by the ray marching strategy (e.g., empty-space skipping).
- Hardware: Tested on NVIDIA GeForce GTX 460 [S1].

**Edge Cases and Pitfalls:**
- **Channel Selection:** Storing FCC data in a 4-channel texture requires careful channel selection during fetches, which can slow down performance if not optimized [S2, unofficial].
- **Precision:** The use of integer arithmetic for transformations (`R`, `T`) avoids precision issues in coordinate mapping [S2, unofficial].
- **Boundary Conditions:** The method assumes the volume domain is properly bounded; ray casting must handle exits from the volume [S1].

## Operating characteristics
- **Time Complexity:** The evaluation is $O(1)$ per sample due to the fixed stencil size (19 fetches) and branchless transformation [S1][S2, unofficial].
- **Memory Complexity:** The data is stored in a 3D texture with 4 channels, increasing memory usage by 4x compared to a single-channel Cartesian grid of the same physical dimensions, but the FCC lattice requires fewer samples for the same reconstruction quality [S1][S2, unofficial].
- **Performance:** Achieves 20–70 fps for interactive isosurface generation on datasets up to $128 \times 128 \times 256$ [S1]. It is 2.4x faster than BCC raycasting [Finkbeiner et al. 2010] and 0.4x the speed of Cartesian raycasting [Sigg & Hadwiger 2005] [S2, unofficial].
- **Strengths:** High-quality C1 continuous isosurfaces; efficient GPU implementation with no conditional branches in the evaluation loop; symmetric indexing improves cache coherency compared to BCC [S1][S2, unofficial].
- **Limitations:** Slower than Cartesian raycasting due to the complexity of the spline evaluation and texture fetch overhead; pre-processing (quasi-interpolation) is required to convert raw data to the spline coefficients [S1][S2, unofficial].

## Background
The method was introduced by Minho Kim in 2012 [S1]. It builds on the theory of box-splines on non-Cartesian lattices, specifically the six-direction cubic box-spline on the FCC lattice proposed by Entezari and Kim et al. [S1]. The FCC lattice is noted for its optimality in low-rate sampling scenarios [S1]. The method improves upon previous BCC raycasters by providing a symmetric and cache-friendly indexing scheme [S1][S2, unofficial].

## Sources
[S1] GPU Isosurface Raycasting of FCC Datasets, https://minho-kim-uos.github.io/papers/kim12gpu.pdf, fetched 2026-10-29
[S2] GPU Isosurface Raycasting of Volume Datasets Based On Box-Splines, https://minho-kim-uos.github.io/papers/birs13.pdf, fetched 2026-10-29 (unofficial)
