# Neural Geometric Level of Detail

Also called: Neural Geometric LOD, Neural LOD, SVO-based Neural SDF with LOD

## Overview
Neural Geometric Level of Detail (NGLD) is a neural representation for implicit 3D shapes that uses a Sparse Voxel Octree (SVO) to store learned feature vectors, enabling adaptive multi-level detail (LOD) and real-time rendering via sparse sphere tracing [S1]. Unlike fixed-size neural networks (e.g., DeepSDF, FFN) or dense voxel grids, NGLD decouples geometric complexity from network size by storing features in a sparse octree and using small, shallow MLPs as decoders [S1]. This allows the representation to scale memory usage with geometric detail while maintaining constant inference cost per query, achieving state-of-the-art geometry reconstruction quality with real-time performance [S1].

## How it works
NGLD builds on [Neural and primitive-based representations](10-neural-and-primitive-based-representations.md) by replacing the single latent vector or dense grid with a sparse octree of features. It represents Signed Distance Functions (SDFs) using an SVO of learned feature vectors and small MLP decoders for each level of detail (LOD) [S1].

### Inputs and Outputs
- **Inputs**: Query point $\mathbf{x} \in \mathbb{R}^3$, target continuous LOD level $\tilde{L} \ge 1$.
- **Outputs**: Signed distance value $\hat{d}_{\bar{L}} \in \mathbb{R}$.

### Data Structures
1.  **Sparse Voxel Octree (SVO)**:
    -   Spans bounding volume $\mathcal{B} = [-1, 1]^3$ [S1].
    -   Levels $L \in \{1, \dots, L_{\max}\}$ define discrete LODs [S1].
    -   Resolution at level $L$ is $r_L = r_0 \cdot 2^L$, with initial resolution $r_0 = 4$ [S1].
    -   Voxels are allocated only if they contain a surface [S1].
    -   Each voxel $V$ at level $L$ holds 8 learnable feature vectors $\mathbf{z}_V^{(j)} \in \mathbb{R}^m$ at its corners, indexed by $j=0..7$ [S1].
    -   Features are shared between neighboring voxels [S1].
    -   Memory impact: $M = (m+1)|V_{1:L_{\max}}|$ bytes, where $m+1$ accounts for feature dimension and Z-order curve indexing, and $|V_{1:L_{\max}}|$ is the octree size [S1].
2.  **Decoders**:
    -   A set of shallow MLPs $f_{\theta_L}$ for each LOD level $L$ [S1].
    -   Parameters $\theta_{1:L_{\max}}$ [S1].
    -   Single hidden layer with dimension $h=128$, ReLU activation, and no activation on the output layer [S1].
    -   Decoder size is fixed at 90 KB [S1].

### Parameters
-   $m = 32$: Feature vector dimension [S1].
-   $h = 128$: MLP hidden layer dimension [S1].
-   $L_{\max}$: Maximum tree depth (typically 5 or 6) [S1].
-   $\sigma = 0.01$: Standard deviation for Gaussian prior initialization of voxel features $\mathbf{z}$ [S1].
-   $r_0 = 4$: Initial resolution [S1].

### Algorithm Steps

1.  **Feature Aggregation**:
    Given query point $\mathbf{x}$ and discrete LOD level $L$:
    1.  Traverse SVO up to level $L$ to find voxels $V_{1:L} = \{V_1, \dots, V_L\}$ containing $\mathbf{x}$ [S1].
    2.  For each level $l \in \{1, \dots, L\}$, compute per-voxel shape vector $\boldsymbol{\psi}(\mathbf{x}; l, \mathcal{Z})$ by trilinearly interpolating the corner features $\mathbf{z}_V^{(j)}$ of the voxel at level $l$ containing $\mathbf{x}$ [S1].
    3.  Sum features across levels: $\mathbf{z}(\mathbf{x}; L, \mathcal{Z}) = \sum_{l=1}^{L} \boldsymbol{\psi}(\mathbf{x}; l, \mathcal{Z})$ [S1].

2.  **SDF Computation**:
    1.  Concatenate $\mathbf{x}$ and aggregated feature $\mathbf{z}(\mathbf{x}; L, \mathcal{Z})$ [S1].
    2.  Pass to MLP $f_{\theta_L}$:
        $$ \hat{d}_L = f_{\theta_L}([\mathbf{x}, \mathbf{z}(\mathbf{x}; L, \mathcal{Z})]) $$
        where $[\cdot, \cdot]$ denotes concatenation [S1].

3.  **Continuous LOD Blending**:
    For a desired continuous LOD $\tilde{L} \ge 1$:
    1.  Calculate $L^* = \lfloor \tilde{L} \rfloor$ and $\alpha = \tilde{L} - \lfloor \tilde{L} \rfloor$ [S1].
    2.  Compute blended distance:
        $$ \hat{d}_{\bar{L}} = (1 - \alpha) \hat{d}_{L^*} + \alpha \hat{d}_{L^*+1} $$
    Note: This blending works only for SDFs [S1].

4.  **Training**:
    Minimize loss $J(\theta, \mathcal{Z})$:
    $$ J(\theta, \mathcal{Z}) = \mathbb{E}_{\mathbf{x}, d} \sum_{L=1}^{\max} \| f_{\theta_L}([\mathbf{x}, \mathbf{z}(\mathbf{x}; L, \mathcal{Z})]) - d \|^2 $$
    -   Optimize $\theta_{1:L_{\max}}$ and $\mathcal{Z}$ stochastically [S1].
    -   Samples $\mathbf{x}$ are drawn from a mixture of: uniform samples in $\mathcal{B}$, surface samples, and perturbed surface samples [S1].
    -   Dataset preparation: Normalize mesh, remove internal triangles, sign distances with ray stabbing [S1].

5.  **Interactive Rendering (Sparse Sphere Tracing)**:
    1.  **Pre-frame Ray-SVO Intersection**:
        -   For each ray $r$, retrieve every voxel $V$ at each resolution $l$ intersecting $r$ [S1].
        -   Store ray index, voxel position, parent voxel, and pointers to 8 corner feature vectors $\mathbf{z}_V^{(j)}$ in $V_l(r)$ [S1].
        -   Pointers are precalculated by iterating all voxels [S1].
    2.  **Sparse Ray-Octree Intersection (Algorithm 1)**:
        ```python
        def RAYTRACEOCTREE(L, R):
            # R is list of rays, N(0) initialized with root node for each ray
            N = [ (ray_index, root_node) for ray_index in range(len(R)) ]
            for l in range(0, L + 1):
                # D: binary list, Dj=1 if ray intersects voxel j at level l
                D = DECIDE(R, N[l], l)
                # S: exclusive sum of D
                S = EXCLUSIVESUM(D)
                if l == L:
                    # Compactify to get final list of intersections
                    N[l] = COMPACTIFY(N[l], D, S)
                else:
                    # Subdivide to get next level proposals
                    N[l+1] = SUBDIVIDE(N[l], D, S)
            return N[L]
        ```
        -   Uses breadth-first traversal and parallel scan kernels [S1].
        -   Child voxels are ordered by distance to ray origin based on octant analysis [S1].
    3.  **Adaptive Ray Stepping**:
        -   Choose continuous LOD $\tilde{L}$ using a depth heuristic based on distance to object [S1].
        -   Let $L = \lfloor \tilde{L} \rfloor$ [S1].
        -   Find first voxel $V^*_L$ in $V_L(r)$ that hits the ray [S1].
        -   Current point $\mathbf{x}_k$:
            -   If $\mathbf{x}_k \notin V^*_L$, advance $\mathbf{x}_k$ to ray-AABB intersection point [S1].
            -   If $\mathbf{x}_k \in V^*_L$, recursively retrieve parent voxels $V^*_l$ for coarser levels $l \in \{1, \dots, L-1\}$ [S1].
            -   Sum trilinearly interpolated features at each node to get $\mathbf{z}(\mathbf{x}_k; L, \mathcal{Z})$ [S1].
            -   Pass to MLP $f_{\theta_L}$ to get conservative distance $d_b^L$ [S1].
            -   Step: $\mathbf{x}_{k+1} = \mathbf{x}_k + d_b^L \cdot \mathbf{d}$ [S1].
            -   If $\mathbf{x}_{k+1}$ is in empty space, skip to next voxel in $V_L(r)$ [S1].
            -   If inside voxel, repeat sphere trace step [S1].

### Edge Cases and Pitfalls
-   **Empty Space**: Standard SDFs are defined on all $\mathbb{R}^3$, but NGLD is sparse. Proper handling of distance queries in empty space is required to avoid overshooting [S1].
-   **Low LOD Quality**: At the lowest LOD, the surface extractor struggles to reconstruct good surfaces because features become increasingly high-level and complex [S1].
-   **Non-Metric SDFs**: Non-metric SDFs (e.g., Oldcar model) do not satisfy the Eikonal equation $|\nabla f| = 1$ and contain discontinuities. NGLD can capture these, whereas standard FFN and SIREN fail [S1].
-   **Scaling**: Scaling to extremely large scenes or very thin, volume-less geometry is difficult due to dependence on training point samples [S1].
-   **Animation**: Traditional methods for animating or deforming geometry are not easily supported [S1].
-   **Level Blending**: Only works for SDFs; ill-defined for meshes, point clouds, or density/occupancy [S1].

## Operating characteristics
-   **Time Complexity**: Inference costs are fixed at 4737 floats across all resolutions [S1]. The bottleneck is MLP inference and memory I/O, not ray-octree intersection [S1].
-   **Memory Complexity**: $M = (m+1)|V_{1:L_{\max}}|$ bytes [S1]. Storage is sparse, with voxels allocated only where surfaces exist [S1].
-   **Performance**:
    -   On TurboSquid, speeds are 50x faster than FFN and comparable to Neural Implicits (NI) [S1].
    -   Sparse frametimes are >100x faster than DeepSDF [S1].
    -   Over 500x faster than NeRF in frametimes [S1].
    -   50x faster than NSVF [S1].
-   **Training**: Converges quickly, achieving better reconstruction with ~45% the training time of DeepSDF and FFN (30 epochs vs 100) [S1].
-   **Strengths**: Real-time rendering, state-of-the-art geometry quality, smooth LOD blending, handles non-metric SDFs.
-   **Limitations**: Difficult to scale to very large scenes or thin geometry; hard to animate/deform; low LOD quality can be poor [S1].

## Background
NGLD was introduced by Towaki Takikawa, Joey Litalien, Kangxue Yin, Karsten Kreis, Charles Loop, Derek Nowrouzezahrai, Alec Jacobson, Morgan McGuire, and Sanja Fidler from NVIDIA, University of Toronto, McGill University, and Vector Institute [S1]. It addresses the prohibitive cost of real-time rendering with large neural SDFs by introducing an octree-based feature volume that adaptively fits shapes with multiple LODs [S1]. Potential applications include scene reconstruction, ultra-precise robotics path planning, and interactive content creation [S1].

## Sources
[S1] Neural Geometric Level of Detail: Real-time Rendering with Implicit 3D Shapes, https://doi.org/10.1109/cvpr46437.2021.01120, fetched 2026-10-26
