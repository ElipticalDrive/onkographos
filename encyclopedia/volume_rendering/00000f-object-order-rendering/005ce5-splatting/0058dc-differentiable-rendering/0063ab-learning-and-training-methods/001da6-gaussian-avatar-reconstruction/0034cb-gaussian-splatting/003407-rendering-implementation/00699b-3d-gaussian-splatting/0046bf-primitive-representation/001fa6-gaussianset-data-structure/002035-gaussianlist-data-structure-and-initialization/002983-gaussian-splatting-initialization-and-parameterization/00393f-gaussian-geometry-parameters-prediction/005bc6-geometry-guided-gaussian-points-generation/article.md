# Geometry Guided Gaussian Points Generation

Also called: MGadapter, Mesh Sampler, generalized adapter, heuristic function T

## Overview
Geometry Guided Gaussian Points Generation (MGadapter) is a method that generates structured 3D Gaussian point attributes from an intermediate triangular mesh, providing explicit geometric guidance to 3D Gaussian Splatting (3DGS) [S1]. It solves the problem of vanilla 3DGS lacking deterministic geometry boundaries and explicit normal directions, which complicates light transport computation and hinders physically-based inverse rendering [S1]. Unlike sibling methods that rely on learned approximated normals or real-time optimization to maintain shape consistency, MGadapter uses a predefined heuristic function to map mesh triangles directly to Gaussian points, leveraging well-defined mesh normals for improved accuracy and stable training [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A triangular mesh $M$ derived from a scalar field $\zeta$ via differentiable isosurface extraction (e.g., FlexiCubes) [S1]. Each triangle face $F_i$ consists of three vertices $P_i = (p_{i1}, p_{i2}, p_{i3})$ and corresponding vertex normals $N_i = (n_{i1}, n_{i2}, n_{i3})$ [S1].
*   **Output:** A set of $K$ Gaussian point attributes $\{(\mu_i, S_i, R_i, n_i) \mid i=1,\dots,K\}$ for each triangle $P$ [S1].

**Data Structures:**
*   **Triangle Face $F_i$:** Defined by vertex positions $P_i$ and vertex normals $N_i$ [S1].
*   **Per-vertex Attributes $A_i$:** Stored as tuples $A_i = (a_{i1}, a_{i2}, a_{i3})$ corresponding to the three vertices of the triangle [S1].
*   **Gaussian Point Attributes:**
    *   Position $\mu_i \in \mathbb{R}^3$
    *   Normal $n_i \in \mathbb{R}^3$
    *   Scale $S_i$ (represented as $S_{ix}, S_{iy}$)
    *   Rotation $R_i$ (represented as direction vector $R_{ix}$)
    *   Opacity: Fixed to 1 [S1].

**Parameters:**
*   $K$: Number of Gaussian points per triangle. Empirically set to $K=6$ [S1].
*   $u, v$: Scalar parameters defining barycentric coordinates for sampling points. Values are not specified in the sources [S1].
*   $\alpha_{jk}, \beta_{jk}$: Scaling factors for the Gaussian dimensions. Values are not specified in the sources [S1].

**Steps:**
1.  **Mesh Generation:** Obtain the intermediate triangular mesh $M$ from the scalar field $\zeta$ using a differentiable isosurface extraction technique such as FlexiCubes [S1].
2.  **Iteration:** For each triangle face $F_i$ in mesh $M$, generate $K=6$ Gaussian points [S1].
3.  **Barycentric Coordinate Definition:** Define 6 barycentric coordinates $\mathbf{b}_j$ for $j=1,\dots,6$ using parameters $u$ and $v$ [S1]:
    $$
    \begin{aligned}
    \mathbf{b}_1 &= (u, u, 1-2u) \\
    \mathbf{b}_2 &= (u, 1-2u, u) \\
    \mathbf{b}_3 &= (1-2u, u, u) \\
    \mathbf{b}_4 &= (v, v, 1-2v) \\
    \mathbf{b}_5 &= (v, 1-2v, v) \\
    \mathbf{b}_6 &= (1-2v, v, v)
    \end{aligned}
    $$
4.  **Midpoint Calculation:** Compute 6 midpoints $\mathbf{m}_{jk}$ by averaging pairs of barycentric coordinates [S1]:
    $$
    \begin{aligned}
    \mathbf{m}_{12} &= \frac{\mathbf{b}_1 + \mathbf{b}_2}{2} = \left(u, \frac{1-u}{2}, \frac{1-u}{2}\right) \\
    \mathbf{m}_{23} &= \frac{\mathbf{b}_2 + \mathbf{b}_3}{2} = \left(\frac{1-u}{2}, \frac{1-u}{2}, u\right) \\
    \mathbf{m}_{31} &= \frac{\mathbf{b}_3 + \mathbf{b}_1}{2} = \left(\frac{1-u}{2}, u, \frac{1-u}{2}\right) \\
    \mathbf{m}_{45} &= \frac{\mathbf{b}_4 + \mathbf{b}_5}{2} = \left(v, \frac{1-v}{2}, \frac{1-v}{2}\right) \\
    \mathbf{m}_{56} &= \frac{\mathbf{b}_5 + \mathbf{b}_6}{2} = \left(\frac{1-v}{2}, \frac{1-v}{2}, v\right) \\
    \mathbf{m}_{64} &= \frac{\mathbf{b}_6 + \mathbf{b}_4}{2} = \left(\frac{1-v}{2}, v, \frac{1-v}{2}\right)
    \end{aligned}
    $$
5.  **Barycentric Interpolation:** For any attribute $A_i = (a_{i1}, a_{i2}, a_{i3})$ defined at the triangle vertices and a barycentric coordinate $(q_1, q_2, q_3)$, compute the interpolated value as [S1]:
    $$
    (q_1, q_2, q_3) \odot A_i = q_1 a_{i1} + q_2 a_{i2} + q_3 a_{i3}
    $$
6.  **Gaussian Point Attribute Assignment:** For each midpoint $\mathbf{m}_{jk}$ (where $jk \in \{12, 23, 31, 45, 56, 64\}$), assign the Gaussian point attributes [S1]:
    *   **Position:** $\mu = \mathbf{m}_{jk} \odot P_i$
    *   **Normal:** $n = \mathbf{m}_{jk} \odot N_i$
    *   **Scale X:** $S_{ix} = \alpha_{jk} \| \mathbf{b}_k \odot P_i - \mathbf{m}_{jk} \odot P_i \|_2$
    *   **Rotation X:** $R_{ix} = \frac{\mathbf{b}_k \odot P_i - \mathbf{m}_{jk} \odot P_i}{\| \mathbf{b}_k \odot P_i - \mathbf{m}_{jk} \odot P_i \|_2}$
    *   **Scale Y:** $S_{iy} = \frac{\text{Area}(F_i)}{\beta_{jk} \| \dots \|}$ (Formula incomplete in source [S1])
    *   **Opacity:** Set to 1 [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_gaussians_from_triangle(vertices, normals, u, v, alpha, beta):
    """
    Generates 6 Gaussian points from a single triangle face.
    
    Args:
        vertices: List of 3 vertex positions [p1, p2, p3], each (x,y,z).
        normals: List of 3 vertex normals [n1, n2, n3], each (x,y,z).
        u: Scalar parameter for first set of barycentric coordinates.
        v: Scalar parameter for second set of barycentric coordinates.
        alpha: List of 6 scaling factors for S_x [alpha_12, alpha_23, ...].
        beta: List of 6 scaling factors for S_y [beta_12, beta_23, ...].
        
    Returns:
        List of 6 Gaussian point dictionaries with keys:
        'mu' (pos), 'n' (normal), 'Sx', 'Rx', 'Sy', 'Ry', 'opacity'.
    """
    # Define barycentric coordinates
    b = [
        (u, u, 1 - 2*u),
        (u, 1 - 2*u, u),
        (1 - 2*u, u, u),
        (v, v, 1 - 2*v),
        (v, 1 - 2*v, v),
        (1 - 2*v, v, v)
    ]
    
    # Define midpoints indices and their barycentric values
    # m_jk corresponds to midpoint between b_j and b_k
    midpoints = [
        (b[0], b[1]), # m12
        (b[1], b[2]), # m23
        (b[2], b[0]), # m31
        (b[3], b[4]), # m45
        (b[4], b[5]), # m56
        (b[5], b[3])  # m64
    ]
    
    gaussians = []
    
    for idx, (b_j, b_k) in enumerate(midpoints):
        # Calculate midpoint barycentric coords
        m = ((b_j[0] + b_k[0]) / 2, 
             (b_j[1] + b_k[1]) / 2, 
             (b_j[2] + b_k[2]) / 2)
        
        # Barycentric interpolation helper
        def bary_interp(attr_list, coords):
            return (coords[0] * attr_list[0] + 
                    coords[1] * attr_list[1] + 
                    coords[2] * attr_list[2])
        
        # Position and Normal
        mu = bary_interp(vertices, m)
        n = bary_interp(normals, m)
        
        # Calculate vector for Scale X and Rotation X
        # Vector from midpoint to the 'k' vertex in barycentric space mapped to world
        # Note: b_k is the barycentric coordinate of the vertex defining the edge direction
        # The source formula uses b_k \odot P_i - m \odot P_i
        vertex_k_pos = bary_interp(vertices, b_k)
        midpoint_pos = bary_interp(vertices, m)
        
        vec = (vertex_k_pos[0] - midpoint_pos[0],
               vertex_k_pos[1] - midpoint_pos[1],
               vertex_k_pos[2] - midpoint_pos[2])
        
        norm_vec = (vec[0]**2 + vec[1]**2 + vec[2]**2)**0.5
        
        # Avoid division by zero for degenerate triangles
        if norm_vec < 1e-8:
            Sx = 0.0
            Rx = (0.0, 0.0, 0.0)
        else:
            Sx = alpha[idx] * norm_vec
            Rx = (vec[0]/norm_vec, vec[1]/norm_vec, vec[2]/norm_vec)
            
        # Scale Y calculation
        # Area of triangle
        # Area = 0.5 * || (p2-p1) x (p3-p1) ||
        p1, p2, p3 = vertices
        v1 = (p2[0]-p1[0], p2[1]-p1[1], p2[2]-p1[2])
        v2 = (p3[0]-p1[0], p3[1]-p1[1], p3[2]-p1[2])
        cross = (v1[1]*v2[2] - v1[2]*v2[1],
                 v1[2]*v2[0] - v1[0]*v2[2],
                 v1[0]*v2[1] - v1[1]*v2[0])
        area = 0.5 * (cross[0]**2 + cross[1]**2 + cross[2]**2)**0.5
        
        # Sy formula incomplete in source, using placeholder logic based on structure
        # Sy = Area / (beta * norm)
        if norm_vec < 1e-8:
            Sy = 0.0
        else:
            Sy = area / (beta[idx] * norm_vec)
            
        # Rotation Y is not explicitly defined in the provided excerpt for Ry, 
        # but typically orthogonal to Rx. Source cuts off.
        # [G] Assuming Ry is not provided in the specific excerpt, we leave it undefined or zero.
        Ry = (0.0, 0.0, 0.0) 

        gaussians.append({
            'mu': mu,
            'n': n,
            'Sx': Sx,
            'Rx': Rx,
            'Sy': Sy,
            'Ry': Ry,
            'opacity': 1.0
        })
        
    return gaussians
```

**Edge Cases and Pitfalls:**
*   **Degenerate Triangles:** If the triangle area or edge length is zero, division by zero occurs in scale/rotation normalization. A guard (e.g., $10^{-8}$) is required [G].
*   **Missing Parameters:** The specific values for $u, v, \alpha_{jk}, \beta_{jk}$ are not provided in the sources [S1]. The formula for $S_{iy}$ is incomplete in the source text [S1].
*   **Mesh Modification:** Unlike methods that maintain shape consistency via real-time optimization, MGadapter uses a static heuristic. If the mesh is modified, the Gaussian points do not automatically update without re-running the adapter [S1].

## Subtopics

### [GSGEN](8985-gsgen.md)
A sibling method for Gaussian generation.

### [Textured-GS](9436-textured-gs.md)
A sibling method focusing on texture.

### [Deformable 3D Gaussian Splatting](10060-deformable-3d-gaussian-splatting.md)
A sibling method handling deformation.

### [Fourier Splatting Primitive](11094-fourier-splatting-primitive.md)
A sibling method using Fourier primitives.

### [Conditional Slicing Mechanism](11455-conditional-slicing-mechanism.md)
A sibling method using slicing.

### [Structure-Aware 3D Gaussian Splatting (SAGS)](11908-structure-aware-3d-gaussian-splatting-sags.md)
A sibling method aware of structure.

### [Mesh-embedded representation](12559-mesh-embedded-representation.md)
A sibling method embedding meshes.

### [Inverse Gaussian Splattling](13193-inverse-gaussian-splattling.md)
A sibling method for inverse rendering.

### [Occupancy-Opacity Factorization](13440-occupancy-opacity-factorization.md)
A sibling method factorizing occupancy.

### [QuickSplat](15256-quicksplat.md)
A sibling method for speed.

### [CAG-GS Semantic Mapping and Gaussian Generation](15260-cag-gs-semantic-mapping-and-gaussian-generation.md)
A sibling method using semantics.

### [Geometry and orientation extension](15381-geometry-and-orientation-extension.md)
A sibling method extending geometry.

### [Voxel and Grid-Based Prediction](15382-voxel-and-grid-based-prediction.md)
A sibling method using voxels.

### [Attribute-Conditioned SH Enhancement](15410-attribute-conditioned-sh-enhancement.md)
A sibling method enhancing SH.

### [Functional Gaussian Splatting Representation](15572-functional-gaussian-splatting-representation.md)
A sibling method using functionals.

### [InnerGS Volumetric Rendering](15748-innergs-volumetric-rendering.md)
A sibling method for volumetric rendering.

### [Hybrid and Multi-Representation](15831-hybrid-and-multi-representation.md)
A sibling method using hybrid representations.

### [Feed-forward and Generalizable Prediction](15832-feed-forward-and-generalizable-prediction.md)
A sibling method for generalization.

### [Plenoptic Sampling Guided 3DGS Construction](16971-plenoptic-sampling-guided-3dgs-construction.md)
A sibling method using plenoptic sampling.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:**
    *   Provides explicit geometric guidance and well-defined mesh normals, improving normal accuracy [S1].
    *   Avoids the need for approximated normals and normal-depth consistency loss [S1].
    *   Enables effective light transport modeling and self-occlusion evaluation using the mesh as a proxy [S1].
    *   Maintains shape consistency without real-time optimization, avoiding unstable training processes associated with re-executing optimization steps when the mesh is modified [S1].
*   **Limitations:**
    *   Relies on a predefined heuristic function, which may not adapt to complex mesh variations as well as learned methods [S1].
    *   Requires an intermediate mesh $M$ generated from a scalar field $\zeta$ [S1].
*   **When to Use:** When physically-based inverse rendering is required, and explicit geometry boundaries and normals are critical for light transport computation [S1].
*   **When Not to Use:** When the mesh is frequently modified and real-time adaptation of Gaussian points is needed without re-running the adapter [S1].

## Background
*   **Origin:** The method was introduced in "GeoSplatting: Towards Geometry Guided Gaussian Splatting for Physically-Based Inverse Rendering" [S1].
*   **History:** The work addresses limitations of vanilla 3DGS in inverse rendering by integrating mesh geometry. It contrasts with prior works that learn approximated normals [S1].
*   **Usage:** Used in physically-based inverse rendering tasks where accurate normal estimation and light transport modeling are essential [S1].

## Sources
[S1] GeoSplatting: Towards Geometry Guided Gaussian Splatting for Physically-Based Inverse Rendering, https://doi.org/10.1109/ICCV51701.2025.02692, fetched 2026-10-30
