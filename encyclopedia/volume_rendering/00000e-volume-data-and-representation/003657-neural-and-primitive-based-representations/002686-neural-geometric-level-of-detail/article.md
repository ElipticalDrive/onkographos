# Neural Geometric Level of Detail

Also called: Neural Geometric LOD, Neural LOD, SVO-based Neural SDF with LOD

## Overview
Neural Geometric Level of Detail (NGLD) is a neural representation for implicit 3D shapes that uses a Sparse Voxel Octree (SVO) to store learned feature vectors, enabling adaptive multi-level detail (LOD) and real-time rendering via sparse sphere tracing [S1]. It solves the prohibitive cost of rendering large neural Signed Distance Functions (SDFs) by decoupling geometric complexity from the neural network size; instead of a single large MLP, it uses small, shallow MLPs (decoders) that decode features aggregated from the SVO [S1]. Unlike [NeuS](6060-neus.md) which uses volumetric rendering, or [MLP Neural Fields](3854-mlp-neural-fields.md) which rely on dense sampling, NGLD enables real-time frame rates by querying only necessary LODs during traversal and blending SDF values for continuous LOD transitions [S1].

## How it works
**Inputs:**
*   Query point $\mathbf{x} \in \mathbb{R}^3$.
*   Target continuous LOD level $\tilde{L} \ge 1$.
*   Trained parameters: MLP weights $\Theta = \{\theta_1, \dots, \theta_{L_{max}}\}$ and SVO feature vectors $\mathcal{Z}$.

**Outputs:**
*   Signed distance value $\hat{d}_{\bar{L}}$ at the blended LOD.

**Data Structures:**
1.  **Sparse Voxel Octree (SVO):**
    *   Spans bounding volume $B = [-1, 1]^3$.
    *   Nodes are allocated only if they contain surface geometry.
    *   Each voxel $V$ at level $l$ holds 8 learnable feature vectors $\mathbf{z}_V^{(j)} \in \mathbb{R}^m$ at its corners ($j=0..7$).
    *   Features are shared between neighboring voxels.
    *   Memory impact: $M = (m+1)|V_{1:L_{max}}|$ bytes, where $m+1$ accounts for the feature dimension and Z-order curve indexing [S1].
2.  **MLP Decoders ($f_{\theta_L}$):**
    *   One shallow MLP per LOD level $L \in \{1, \dots, L_{max}\}$.
    *   Architecture: Single hidden layer with dimension $h=128$, ReLU activation, and no activation on the output layer [S1].
    *   Input dimension: $3 + m$ (concatenation of $\mathbf{x}$ and aggregated feature $\mathbf{z}$).
    *   Output dimension: 1 (scalar distance).
    *   Inference cost is fixed at 4737 floats across all resolutions [S1].

**Parameters:**
*   $m = 32$: Feature vector dimension [S1].
*   $h = 128$: MLP hidden layer dimension [S1].
*   $L_{max}$: Maximum tree depth (typically 5 or 6) [S1].
*   $r_0 = 4$: Initial resolution of the SVO [S1].
*   $\sigma = 0.01$: Standard deviation for Gaussian prior initialization of voxel features [S1].
*   $B = [-1, 1]^3$: Bounding volume [S1].

**Algorithm Steps:**

1.  **Feature Aggregation:**
    Given query $\mathbf{x}$ and discrete LOD level $L$:
    1.  Traverse the SVO up to level $L$ to find all voxels $V_{1:L} = \{V_1, \dots, V_L\}$ containing $\mathbf{x}$ [S1].
    2.  For each level $l \in \{1, \dots, L\}$, compute the per-voxel shape vector $\boldsymbol{\psi}(\mathbf{x}; l, \mathcal{Z})$ by trilinearly interpolating the corner features $\mathbf{z}_V^{(j)}$ of the voxel $V_l$ at $\mathbf{x}$ [S1].
    3.  Sum the features across all levels to obtain the aggregated feature vector:
        $$ \mathbf{z}(\mathbf{x}; L, \mathcal{Z}) = \sum_{l=1}^{L} \boldsymbol{\psi}(\mathbf{x}; l, \mathcal{Z}) $$
    4.  Concatenate $\mathbf{x}$ and $\mathbf{z}$, and pass to the MLP for level $L$:
        $$ \hat{d}_L = f_{\theta_L}([\mathbf{x}, \mathbf{z}(\mathbf{x}; L, \mathcal{Z})]) $$
        where $[\cdot, \cdot]$ denotes concatenation [S1].

2.  **Continuous LOD Blending:**
    For a continuous target LOD $\tilde{L} \ge 1$:
    1.  Calculate integer part $L^* = \lfloor \tilde{L} \rfloor$ and fractional part $\alpha = \tilde{L} - \lfloor \tilde{L} \rfloor$ [S1].
    2.  Compute blended distance:
        $$ \hat{d}_{\bar{L}} = (1 - \alpha) \hat{d}_{L^*} + \alpha \hat{d}_{L^*+1} $$
    *Note: This blending works specifically for SDFs and is ill-defined for density/occupancy or meshes [S1].*

3.  **Interactive Rendering (Sparse Sphere Tracing):**
    To render a frame with ray $\mathbf{r}(t) = \mathbf{x}_0 + t\mathbf{d}$:
    1.  **Pre-frame Ray-SVO Intersection:**
        Perform sparse ray-octree intersection to retrieve every voxel $V$ at each resolution $l$ intersecting the ray. Store ray index, voxel position, parent voxel, and pointers to the 8 corner feature vectors in $V_l(\mathbf{r})$ [S1]. Pointers are precalculated by iterating all voxels [S1].
    2.  **Adaptive Ray Stepping:**
        1.  Perform ray-AABB intersection against voxels in the target LOD level $L$ to find the first voxel $V^*_L \in V_L(\mathbf{r})$ that hits [S1].
        2.  If current point $\mathbf{x}_k$ is not in $V^*_L$, advance $\mathbf{x}_k$ to the ray-AABB intersection point [S1].
        3.  If $\mathbf{x}_k \in V^*_L$, recursively retrieve parent voxels $V^*_l$ for coarser levels $l \in \{1, \dots, L-1\}$ [S1].
        4.  Sum trilinearly interpolated features at each node to compute $\mathbf{z}(\mathbf{x}_k; L, \mathcal{Z})$ [S1].
        5.  Pass $\mathbf{x}_k$ and $\mathbf{z}$ to $f_{\theta_L}$ to get conservative distance $d_b^L$ [S1].
        6.  Step: $\mathbf{x}_{k+1} = \mathbf{x}_k + d_b^L \mathbf{d}$ [S1].
        7.  If $\mathbf{x}_{k+1}$ is in empty space, skip to the next voxel in $V_L(\mathbf{r})$. If inside a voxel, repeat sphere trace step [S1].

4.  **Sparse Ray-Octree Intersection (Parallel Traversal):**
    Algorithm 1 in [S1] describes an iterative, parallel, breadth-first traversal:
    1.  Initialize $N^{(0)}_i \leftarrow \{i, 0\}$ for each ray $i$ (ray index, node index).
    2.  For level $l$ from 0 to $L$:
        a.  $D \leftarrow \text{DECIDE}(\mathbf{R}, N^{(l)}, l)$ returns binary list $D_j=1$ if ray intersects voxel $j$.
        b.  $S \leftarrow \text{EXCLUSIVESUM}(D)$.
        c.  If $l=L$, $N^{(l)} \leftarrow \text{COMPACTIFY}(N^{(l)}, D, S)$.
        d.  Else, $N^{(l+1)} \leftarrow \text{SUBDIVIDE}(N^{(l)}, D, S)$.
    3.  Result is a compact, depth-ordered list of ray-voxel intersections [S1].

5.  **LOD Selection:**
    Choose continuous LOD $\tilde{L}$ using a depth heuristic that transitions linearly with user-defined thresholds based on distance to the object [S1].

6.  **Training:**
    1.  Compute individual losses at each level $L$.
    2.  Sum losses across levels:
        $$ J(\theta, \mathcal{Z}) = \mathbb{E}_{\mathbf{x}, d} \sum_{L=1}^{\max} \| f_{\theta_L}([\mathbf{x}, \mathbf{z}(\mathbf{x}; L, \mathcal{Z})]) - d \|^2 $$
    3.  Stochastically optimize loss w.r.t. $\theta_{1:L_{max}}$ and $\mathcal{Z}$ [S1].
    4.  Estimate expectation with importance sampling using a mixture of: uniform samples in $B$, surface samples, and perturbed surface samples [S1].
    5.  Dataset preparation: Normalize mesh, remove internal triangles, and sign distances with ray stabbing [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def trilinear_interpolate(features, x, voxel_origin, voxel_size):
    """
    Trilinearly interpolate features at point x within a voxel.
    features: list of 8 vectors (corners 0..7)
    x: query point (3,)
    voxel_origin: origin of the voxel (3,)
    voxel_size: size of the voxel (scalar)
    """
    # Normalize x to [0, 1] within the voxel
    u = (x[0] - voxel_origin[0]) / voxel_size
    v = (x[1] - voxel_origin[1]) / voxel_size
    w = (x[2] - voxel_origin[2]) / voxel_size
    
    # Clamp to [0, 1] to handle boundary cases
    u = max(0.0, min(1.0, u))
    v = max(0.0, min(1.0, v))
    w = max(0.0, min(1.0, w))
    
    # Interpolate along x-axis
    f00 = features[0] * (1-u) + features[1] * u
    f10 = features[2] * (1-u) + features[3] * u
    f01 = features[4] * (1-u) + features[5] * u
    f11 = features[6] * (1-u) + features[7] * u
    
    # Interpolate along y-axis
    f0 = f00 * (1-v) + f10 * v
    f1 = f01 * (1-v) + f11 * v
    
    # Interpolate along z-axis
    result = f0 * (1-w) + f1 * w
    return result

def aggregate_features(x, lod_level, svo):
    """
    Aggregate features from SVO up to lod_level.
    svo: Dictionary or structure mapping (level, voxel_id) -> voxel_data
    voxel_data contains: origin, size, features (list of 8 vectors)
    """
    aggregated_z = None
    
    # Traverse SVO to find voxels containing x at each level
    # Note: In practice, this involves ray-voxel or point-voxel lookup
    # Here we assume a function get_voxel_at_level(level, x) returns the voxel
    for l in range(1, lod_level + 1):
        voxel = get_voxel_at_level(l, x, svo)
        if voxel is None:
            continue
        
        # Trilinear interpolation
        psi = trilinear_interpolate(voxel.features, x, voxel.origin, voxel.size)
        
        if aggregated_z is None:
            aggregated_z = psi
        else:
            aggregated_z = aggregated_z + psi
            
    return aggregated_z

def decode_distance(x, aggregated_z, lod_level, mlps):
    """
    Decode distance using the MLP for the specific LOD.
    mlps: Dictionary mapping lod_level -> MLP_weights
    """
    # MLP: Single hidden layer, h=128, ReLU, output dim 1
    # Input: concat(x, aggregated_z)
    # Weights for level L
    W1 = mlps[lod_level]['W1'] # shape (128, 3 + m)
    b1 = mlps[lod_level]['b1'] # shape (128,)
    W2 = mlps[lod_level]['W2'] # shape (1, 128)
    b2 = mlps[lod_level]['b2'] # shape (1,)
    
    input_vec = concat(x, aggregated_z)
    hidden = relu(input_vec @ W1.T + b1)
    distance = hidden @ W2.T + b2
    return distance[0]

def sphere_trace(ray_origin, ray_dir, svo, mlps, lod_selection_func):
    """
    Render a single ray using sparse sphere tracing.
    """
    t = 0.0
    max_steps = 100
    
    for _ in range(max_steps):
        # 1. LOD Selection
        current_dist = ray_origin + t * ray_dir
        lod_tilde = lod_selection_func(current_dist, svo)
        lod_int = int(math.floor(lod_tilde))
        alpha = lod_tilde - lod_int
        
        # 2. Feature Aggregation and Decoding for LODs
        z_lo = aggregate_features(current_dist, lod_int, svo)
        d_lo = decode_distance(current_dist, z_lo, lod_int, mlps)
        
        z_hi = aggregate_features(current_dist, lod_int + 1, svo)
        d_hi = decode_distance(current_dist, z_hi, lod_int + 1, mlps)
        
        # 3. Blending
        d = (1 - alpha) * d_lo + alpha * d_hi
        
        # 4. Step
        if d < 1e-4: # Surface hit
            return t, current_dist
        
        t += d
        
        # 5. Sparse Optimization: Skip empty space
        # Check if current point is in empty space using SVO intersection
        # If in empty space, jump to next voxel boundary
        voxel = get_voxel_containing_point(current_dist, svo)
        if voxel is None:
            # Jump to next intersecting voxel along ray
            # This requires ray-AABB intersection with SVO structure
            next_t = ray_aabb_intersection(ray_origin, ray_dir, svo, t)
            if next_t is not None:
                t = next_t
            else:
                break # Ray escaped
        
    return None, None # No hit
```

## Operating characteristics
*   **Time Complexity:** Rendering speed is dominated by MLP inference and memory I/O, not ray-octree intersection [S1]. Inference costs are fixed at 4737 floats across all resolutions [S1].
*   **Memory Complexity:** $M = (m+1)|V_{1:L_{max}}|$ bytes [S1]. Storage for TurboSquid V Mech at LOD 1 is 96 KB (including 90 KB decoder) [S1].
*   **Performance:** On TurboSquid, speeds are 50x faster than FFN and comparable to Neural Implicits (NI) [S1]. Sparse frametimes are >100x faster than DeepSDF [S1]. Frametimes are >500x faster than NeRF [S1].
*   **Strengths:** Real-time rendering of high-fidelity neural SDFs; state-of-the-art geometry reconstruction; smooth LOD blending; compressed format [S1].
*   **Limitations:** Scaling to extremely large scenes or thin, volume-less geometry is difficult due to dependence on training point samples [S1]; animating/deforming geometry is not easily supported [S1]; non-metric SDFs (discontinuous) are challenging [S1]; lowest LOD struggles with high-level features [S1].

## Background
NGLD was introduced by Towaki Takikawa, Joey Litalien, Kangxue Yin, Karsten Kreis, Charles Loop, Derek Nowrouzezahrai, Alec Jacobson, Morgan McGuire, and Sanja Fidler from NVIDIA, University of Toronto, McGill University, and Vector Institute [S1]. It addresses the limitations of standard neural SDFs (e.g., DeepSDF [S1]) which are too slow for real-time graphics due to large MLPs and many distance queries per ray [S1]. It builds on [Neural and primitive-based representations](10-neural-and-primitive-based-representations.md) by using an SVO to store features instead of a single latent vector, enabling adaptive LOD [S1].

## Sources
[S1] Neural Geometric Level of Detail: Real-time Rendering with Implicit 3D Shapes, https://doi.org/10.1109/cvpr46437.2021.01120, fetched 2026-10-26
