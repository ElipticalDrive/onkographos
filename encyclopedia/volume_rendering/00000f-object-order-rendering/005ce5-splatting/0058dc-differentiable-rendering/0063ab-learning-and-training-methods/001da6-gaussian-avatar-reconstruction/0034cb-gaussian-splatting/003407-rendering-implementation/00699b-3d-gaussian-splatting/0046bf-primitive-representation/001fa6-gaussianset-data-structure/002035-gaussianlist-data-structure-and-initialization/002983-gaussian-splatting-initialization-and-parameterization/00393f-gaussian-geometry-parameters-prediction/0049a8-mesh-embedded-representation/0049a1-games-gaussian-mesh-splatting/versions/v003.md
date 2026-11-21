# GaMeS (Gaussian Mesh Splatting)

Also called: Gaussian Mesh Splatting, Mesh-based adapting and modification of Gaussian Splatting

## Overview
GaMeS is a hybrid representation that integrates 3D Gaussian Splatting with mesh topology, allowing Gaussian components to be parameterized by the vertices of mesh faces [S1]. It solves the problem of conditioning and editing large numbers of independent Gaussians by tying their mean, rotation, and scale directly to the geometry of a canonical triangle mesh [S1]. Unlike sibling methods such as [Mesh-Embedded Gaussian Splatting](9553-mesh-embedded-gaussian-splatting.md) which may rely on expensive pre-processing or SDF regularization, GaMeS directly constructs Gaussians on mesh faces or a "Triangle Soup" derived from flat splats, enabling automatic adaptation of Gaussian parameters when the underlying mesh is deformed [S1].

## How it works
GaMeS operates in two primary input scenarios: mesh-initialized input (where a mesh is provided) and mesh-free input (where a "Triangle Soup" is estimated). The core mechanism involves parameterizing each Gaussian component using the vertices of a specific triangle face, ensuring that geometric transformations of the mesh automatically update the Gaussian's position, orientation, and scale.

### Inputs and Outputs
*   **Inputs**: A canonical triangle mesh $\mathcal{M}$ with $n$ faces, or a set of initial Gaussian splats for the mesh-free case.
*   **Outputs**: A set of parameterized Gaussians $\mathcal{G}_M$ defined by means, covariances, opacities, and spherical harmonics colors, which can be rendered in real-time.

### Data Structures
For a mesh with $n$ faces, GaMeS defines a fixed number $k \in \mathbb{N}$ of Gaussian components per face. The total number of Gaussians is $k \cdot n$ [S1].

For each face $V_i$ and each Gaussian $j \in \{1, \dots, k\}$ on that face, the trainable parameters are:
*   $\alpha_{1j}, \alpha_{2j}, \alpha_{3j}$: Barycentric coordinates such that $\alpha_{1j} + \alpha_{2j} + \alpha_{3j} = 1$.
*   $\rho_j \in \mathbb{R}^+$: A trainable global scale multiplier for the covariance.
*   $\sigma_j$: Opacity.
*   $c_j$: Spherical Harmonics coefficients for color.

### Initialization: Mesh-Initialized Input
When a mesh is provided, all Gaussian components are placed on the mesh surface. For a single triangle face with vertices $V = \{v_1, v_2, v_3\} \subset \mathbb{R}^3$, the Gaussian parameters are derived as follows:

1.  **Calculate Face Normal ($r_1$)**:
    Compute the normal vector using the cross product of edge vectors:
    $$
    r_1 = \frac{(v_2 - v_1) \times (v_3 - v_1)}{\|(v_2 - v_1) \times (v_3 - v_1)\|}
    $$
    [S1]

2.  **Calculate First Tangent ($r_1$ in some contexts, but here defined as normal, let's follow source notation carefully)**:
    The source defines the first orthonormal vector $r_1$ as the normal. However, in the rotation matrix construction $R_V = [r_1, r_2, r_3]$, $r_1$ is the normal. To find $r_2$, the source uses the vector from the face center $m$ to vertex $v_1$:
    $$
    m = \text{mean}(v_1, v_2, v_3)
    $$
    $$
    r_1^{\text{tangent}} = \frac{v_1 - m}{\|v_1 - m\|}
    $$
    *Note: The source text uses $r_1$ for the normal in the cross product formula but then uses $r_1$ for the tangent in the Gram-Schmidt step description. Let's align with the explicit matrix definition $R_V = [r_1, r_2, r_3]$ where $r_1$ is the normal.*
    
    Correcting based on "The first vector is defined by the normal vector... The second one is defined by...":
    Let $n$ be the normal.
    $$
    n = \frac{(v_2 - v_1) \times (v_3 - v_1)}{\|(v_2 - v_1) \times (v_3 - v_1)\|}
    $$
    The source states: "The first vector is defined by the normal vector... The second one is defined by... $r_2 = \frac{v_1 - m}{\|v_1 - m\|}$".
    Then $r_3$ is obtained via Gram-Schmidt.
    
    Let's re-read carefully: "Where the first vector is defined by the normal vector... The second one is defined by... $r_2 = \frac{v_1 - m}{\|v_1 - m\|}$... To obtain $r_3$, we use the vector from the center to the second vertex... $r_3 = \frac{\text{orth}(v_2 - m; r_1, r_2)}{\|\text{orth}(v_2 - m; r_1, r_2)\|}$."
    
    There is a contradiction in the source's variable naming between the normal calculation and the tangent calculation.
    Quote 1: "The first vector is defined by the normal vector... $n = \dots$".
    Quote 2: "The second one is defined by... $r_2 = \frac{v_1 - m}{\|v_1 - m\|}$".
    Quote 3: "To obtain $r_3$, we use the vector from the center to the second vertex... $r_3 = \dots$".
    
    However, earlier it says: "Calculate the first orthonormal vector $r_1$ by normalizing the vector from the face center $m$ to vertex $v_1$". This contradicts the later statement that $r_1$ is the normal.
    
    Let's look at the Rotation Matrix definition: $R_V = [r_1, r_2, r_3]$.
    And the Covariance: $\Sigma_V = R_V S_V S_V^T R_V^T$.
    And Scaling: $s_1 = \epsilon$ (normal vector scaling).
    This implies $r_1$ corresponds to the axis with scale $s_1$, which is the normal.
    Therefore, $r_1$ MUST be the normal.
    
    The statement "Calculate the first orthonormal vector $r_1$ by normalizing the vector from the face center $m$ to vertex $v_1$" is likely a typo in the source notes or refers to a different vector index. The explicit formula for $n$ and the scaling $s_1=\epsilon$ (normal) confirms $r_1$ is the normal.
    
    **Revised Step 1 (Normal):**
    $$
    r_1 = \frac{(v_2 - v_1) \times (v_3 - v_1)}{\|(v_2 - v_1) \times (v_3 - v_1)\|}
    $$
    [S1]

    **Revised Step 2 (Tangent 1):**
    Calculate the centroid $m = \frac{v_1 + v_2 + v_3}{3}$.
    Define the first tangent vector $r_2$ using the vector from the center to vertex $v_1$:
    $$
    r_2 = \frac{v_1 - m}{\|v_1 - m\|}
    $$
    [S1]

    **Revised Step 3 (Tangent 2):**
    Define the second tangent vector $r_3$ using the Gram-Schmidt process on the vector from the center to vertex $v_2$ ($v_2 - m$), orthogonalizing against $r_1$ and $r_2$:
    $$
    \text{orth}(x; r_1, r_2) = x - \text{proj}(x, r_1) - \text{proj}(x, r_2)
    $$
    $$
    \text{proj}(v, u) = \frac{\langle v, u \rangle}{\langle u, u \rangle} u
    $$
    $$
    r_3 = \frac{\text{orth}(v_2 - m; r_1, r_2)}{\|\text{orth}(v_2 - m; r_1, r_2)\|}
    $$
    [S1]

    **Rotation Matrix:**
    $$
    R_V = [r_1, r_2, r_3]
    $$
    [S1]

3.  **Calculate Scaling Matrix ($S_V$)**:
    The scaling matrix is diagonal $S_V = \text{diag}(s_1, s_2, s_3)$.
    *   $s_1 = \epsilon$: A small constant to avoid numerical problems, corresponding to the normal direction (since Gaussians are flat on the surface).
    *   $s_2 = \|m - v_1\|$: The distance from the centroid to vertex $v_1$.
    *   $s_3 = \langle v_2, r_3 \rangle$: The projection of vertex $v_2$ onto the second tangent. *Note: The source says $s_3 = \langle v_2, r_3 \rangle$. This seems dimensionally inconsistent if $v_2$ is a position vector not centered at origin. However, if we assume the local coordinate system or relative vectors, it might be $\langle v_2 - m, r_3 \rangle$ or similar. The source explicitly writes $s_3 = \langle v_2, r_3 \rangle$. We will stick to the source text.*
    [S1]

4.  **Calculate Covariance ($\Sigma_V$)**:
    $$
    \Sigma_V = R_V S_V S_V^T R_V^T
    $$
    [S1]

5.  **Define Gaussian Mean ($m_V$)**:
    The mean is a convex combination of the face vertices:
    $$
    m_V(\alpha_{1j}, \alpha_{2j}, \alpha_{3j}) = \alpha_{1j} v_1 + \alpha_{2j} v_2 + \alpha_{3j} v_3
    $$
    where $\alpha_{1j} + \alpha_{2j} + \alpha_{3j} = 1$.
    [S1]

6.  **Construct Gaussian Component ($G_{Vj}$)**:
    For each Gaussian $j$ on face $V$:
    $$
    G_{Vj} = \mathcal{N}(m_V(\alpha_{1j}, \alpha_{2j}, \alpha_{3j}), \rho_j \Sigma_V)
    $$
    where $\rho_j \in \mathbb{R}^+$ is a trainable scale parameter.
    [S1]

### Initialization: Mesh-Free Input (Triangle Soup)
If no mesh is provided, GaMeS first trains standard Gaussian Splatting with flat splats ($s_1 = \epsilon$). Then, for each Gaussian, it estimates a triangle face to create a "Triangle Soup".

1.  **Define Triangle Vertices from Gaussian**:
    Given a Gaussian with mean $\hat{m}$, rotation $R = [r_1, r_2, r_3]$, and scaling $S = \text{diag}(\epsilon, s_2, s_3)$:
    *   $v_1 = \hat{m}$
    *   $v_2 = \hat{m} + s_2 r_2$
    *   $v_3 = \hat{m} + s_3 r_3$
    [S1]

2.  **Freeze Vertices and Re-parameterize**:
    The vertices $V = [v_1, v_2, v_3]$ are frozen. The Gaussian is re-parameterized with new parameters $\hat{m}, \hat{R}, \hat{S}$.
    *   $\hat{m} = v_1$
    *   $\hat{r}_1 = \frac{(v_2 - v_1) \times (v_3 - v_1)}{\|(v_2 - v_1) \times (v_3 - v_1)\|}$
    *   $\hat{r}_3 = \text{orth}(v_3 - v_1; r_1, r_2)$ *Note: The source uses $r_1, r_2$ from the original Gaussian's rotation here, or potentially the local frame. Given $v_1, v_2, v_3$ are defined by $r_2, r_3$, this step reconstructs the frame.*
    *   $\hat{r}_2$ is implicitly determined by orthogonality or the Gram-Schmidt process on $v_2-v_1$. The source lists $\hat{r}_1$ and $\hat{r}_3$ explicitly.
    *   $\hat{s}_1 = \epsilon$
    *   $\hat{s}_2 = \|v_2 - v_1\|$
    *   $\hat{s}_3 = \langle v_3 - v_1, \hat{r}_3 \rangle$
    [S1]

3.  **Compute New Covariance**:
    $$
    \hat{\Sigma}_V = \hat{R}_V \hat{S}_V \hat{S}_V^T \hat{R}_V^T
    $$
    [S1]

4.  **Final Gaussian**:
    $$
    \hat{G}_V = \mathcal{N}(\hat{m}_V, \hat{\Sigma}_V)
    $$
    [S1]

### Deformation and Editing
When the mesh vertices are modified (e.g., during animation or editing), the Gaussian parameters are automatically updated:
*   **Mean**: Recalculated using the new vertex positions and the fixed barycentric coordinates $\alpha$.
*   **Rotation**: Recalculated from the new face geometry (normals and tangents).
*   **Scale**: Recalculated from the new edge lengths and projections.
This allows the Gaussians to seamlessly conform to mesh deformations, such as facial expressions when using the FLAME framework [S1].

### Edge Cases and Pitfalls
*   **Variable Face Sizes**: If mesh faces vary significantly in size, using a fixed number of Gaussians $k$ per face leads to poor representation. The solution is to subdivide large faces into smaller parts before parameterization [S1].
*   **Numerical Stability**: The scaling parameter $s_1$ (normal direction) is set to a small constant $\epsilon$ rather than zero to avoid numerical issues in the covariance matrix inversion [S1].
*   **Cross-Triangle Updates**: As noted in the parent article, if optimization moves an embedding point across a triangle boundary, the embedding must be re-expressed in the neighboring triangle to maintain differentiability. Failure to handle this causes Gaussians to stick to boundaries [S2, unofficial].

## Operating characteristics
*   **Complexity**: The number of Gaussians is fixed at $k \cdot n$, where $n$ is the number of mesh faces and $k$ is the number of Gaussians per face. This provides a predictable memory footprint compared to unstructured GS.
*   **Performance**: GaMeS enables real-time rendering and editing. The parameterization allows for efficient updates of Gaussian properties via mesh transformations without re-optimizing individual Gaussian parameters from scratch [S1].
*   **Strengths**:
    *   Direct integration with mesh editing tools.
    *   Automatic adaptation of rotation and scale during deformation.
    *   No need for expensive pre-processing stages or SDF regularization required by methods like SuGaR [S1].
*   **Limitations**:
    *   Requires a mesh input or the estimation of a Triangle Soup.
    *   Fixed $k$ per face may not adapt well to highly non-uniform mesh densities without subdivision [S1].

## Background
GaMeS was introduced to address the lack of a well-defined approach for conditioning and editing the hundreds of thousands of Gaussian components in standard 3D Gaussian Splatting [S1]. It builds on the idea of constructing Gaussians directly on mesh surfaces, a concept explored in methods like SuGaR, but distinguishes itself by using a one-stage training approach that accommodates both mesh-initialized and mesh-free inputs [S1]. The method has been demonstrated on datasets like Mip-NeRF360 and synthetic datasets, and is particularly effective for articulated objects when initialized with frameworks like FLAME for human facial expression modification [S1].

## Sources
[S1] GaMeS: Mesh-based adapting and modification of Gaussian Splatting, https://doi.org/10.1016/j.patcog.2026.114561, fetched 2026-10-29
