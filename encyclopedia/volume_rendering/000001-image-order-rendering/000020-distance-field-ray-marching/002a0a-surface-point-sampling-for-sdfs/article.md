# Surface Point Sampling for SDFs

Also called: Forplane Rendering Procedure, Sphere Tracing Surface Sampling

## Overview
Surface Point Sampling for SDFs generates training data or rendering samples by querying a Signed Distance Field (SDF) to locate points near the implicit surface, rather than sampling uniformly throughout the volume. It solves the problem of inefficient volumetric coverage, where most uniform samples fall in empty space or deep inside geometry, providing critical gradient information for the signed distance field values off the zero-level set [S1, S2]. It differs from uniform volume sampling by focusing density on the surface transition region, and from pure surface sampling by providing gradient information for the signed distance field (SDF) values off the zero-level set [S1].

## How it works
The method encompasses several strategies for generating points near the SDF surface, primarily categorized by their application in training (Neural Geometric Level of Detail) or rendering (Efficient Deformable Tissue Reconstruction).

### Inputs and Outputs
*   **Inputs**:
    *   Bounding volume $B$ (typically $[-1, 1]^3$) [S1].
    *   SDF function $f(\mathbf{p})$ [S1, S2].
    *   Ray $r(t) = \mathbf{o} + t\mathbf{d}$ [S2].
    *   Indicator grid $G_o$ (in deformable tissue context) [S2].
    *   Static and dynamic fields $\{V^s_{3d}, V^d_{3d}\}_4$ [S2].
    *   Tiny MLP $\theta$ [S2].
*   **Outputs**:
    *   Point set $P$ distributed around the surface [S1, S2].
    *   Expected color $\hat{C}$ and depth $\hat{D}$ [S2].

### Data Structures
*   **Point**: Coordinates $(x, y, z)$ and optionally time $\tau$ [S2].
*   **Feature Vector**: $v(x, y, z, \tau)$ initialized to 1, updated via interpolation [S2].
*   **Encoded Features**: Concatenation of coordinate-time encoding $b(x, y, z, \tau)$ and interpolated features $v(x, y, z, \tau)$ [S2].

### Sphere Tracing Surface Sampling (Training Data Generation)
This variant, used in Neural Geometric Level of Detail (NGLOD), generates surface points for training implicit shapes [S1].

1.  **Uniform Initialization**: Sample points uniformly distributed in the bounding volume $B = [-1, 1]^3$ [S1].
2.  **Ray Casting**: Choose random points on a sphere to form a ray direction $\mathbf{d}$ [S1]. Define ray $r(t)$.
3.  **Sphere Tracing**: Test if the ray hits the surface using sphere tracing (as defined in [Distance-field ray marching](21-distance-field-ray-marching.md)) [S1].
    *   Evaluate $d = f(\mathbf{p})$ at current position.
    *   Advance $t \leftarrow t + d$.
    *   Terminate if $d < \epsilon$ (hit) or $t > t_{max}$ (miss) [S1].
4.  **Collection**: Continue sampling rays until enough rays hit the surface [S1]. The hit points constitute the surface samples.
5.  **Near-Surface Perturbation (Optional)**: To improve training convergence, perturb surface samples with Gaussian noise $\sigma = 0.01$ to generate near-surface points [S1].

### Indicator Grid Sampling (Deformable Tissue Rendering)
This variant, used in Efficient Deformable Tissue Reconstruction, samples points along a ray filtered by an indicator grid to focus on the surface region [S2].

1.  **Uniform Ray Sampling**: Uniformly sample points $P_u$ along the ray $r(t)$ [S2].
2.  **Indicator Filtering**: Pass $P_u$ through the indicator grid $G_o$ to obtain sample points $P$ distributed around the surface [S2].
3.  **Feature Interpolation**: For each point $P(x, y, z, \tau)$ in $P$:
    *   Initialize feature vector $v(x, y, z, \tau) = 1$ [S2].
    *   Iterate over 4 levels of static and dynamic fields $\{V^s_{3d}, V^d_{3d}\}_4$ and dimension pairs $(xy, xz, x\tau, yz, y\tau, z\tau)$ [S2].
    *   For each grid $G$ and dimension pair $ij$, perform bi-linear interpolation $B(G, i, j)$ [S2].
    *   Update $v$ via element-wise multiplication: $v \leftarrow v \odot B(G, i, j)$ [S2].
4.  **Encoding and Prediction**:
    *   Concatenate coordinate-time encoding $b(x, y, z, \tau)$ with features $v(x, y, z, \tau)$ [S2].
    *   Pass through tiny MLP $\theta$ to predict color $c_i$ and density $\sigma_i$ [S2].
5.  **Volume Rendering**:
    *   Render expected color $\hat{C}$ and depth $\hat{D}$ by integrating radiance and density along the ray [S2]:
        $$ \hat{C} = \int_{t_n}^{t_f} w(t) \cdot c(r(t), d) \, dt $$
        $$ \hat{D} = \int_{t_n}^{t_f} w(t) \cdot t \, dt $$
        $$ w(t) = \exp\left(-\int_{t_n}^{t} \sigma(r(s)) \, ds\right) \sigma(r(t)) $$
    *   Return $\hat{C}, \hat{D}$ [S2].

### Explicit Sampling for Point Cloud Extraction
This variant extracts a point cloud from a Probabilistic Directed Distance Field (PDDF) by projecting random points onto the shape surface $S$ [S5]. It does not guarantee uniform sampling over $S$ [S5].

1.  **Uniform Sampling**: Sample a point $p$ uniformly from the bounding box $B$: $p \sim U[B]$ [S5].
2.  **Direction Sampling**: For each $p$, sample $n_v$ directions $v_i$ uniformly from the unit sphere $S^2$: $V(p) = \{v_i(p) \sim U[S^2]\}_{i=1}^{n_v}$ [S5].
3.  **Visibility Avoidance**: Compose these directions to estimate the best direction $v_b^*(p)$ to avoid visibility $\xi(p, v) = 0$ [S5].
4.  **Projection**: Compute the surface point $q(p, v_b^*(p))$ using the PDDF depth and direction. The condition for $q$ being on $S$ is $\xi(p, v) = 1$ and $q(p, v) = p + d(p, v)v$ [S5].
5.  **Iterative Refinement**: Repeat the process $N_H$ times, starting from the newly found point $q$ (i.e., $p \leftarrow q$), to improve accuracy for depths far from $S$ [S5].

### Pseudocode: Sphere Tracing Surface Sampling
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sample_surface_points_sdf(sdf_function, bounding_box_size=2.0, num_rays=100000, epsilon=0.001, max_steps=100):
    """
    Generates surface points by casting rays and sphere tracing.
    Returns a list of 3D points on the surface.
    """
    surface_points = []
    half_box = bounding_box_box_size / 2.0
    
    for _ in range(num_rays):
        # 1. Choose random point on sphere for ray origin/direction
        # Using spherical coordinates for uniform distribution on sphere
        theta = random.uniform(0, 2 * math.pi)
        phi = math.acos(random.uniform(-1, 1))
        
        # Ray direction
        d = [
            math.sin(phi) * math.cos(theta),
            math.sin(phi) * math.sin(theta),
            math.cos(phi)
        ]
        
        # Ray origin: random point in bounding box
        o = [
            random.uniform(-half_box, half_box),
            random.uniform(-half_box, half_box),
            random.uniform(-half_box, half_box)
        ]
        
        # 2. Sphere Tracing
        t = 0.0
        hit = False
        for _ in range(max_steps):
            p = [o[i] + t * d[i] for i in range(3)]
            dist = sdf_function(p)
            
            if dist < epsilon:
                hit = True
                surface_points.append(p)
                break
            
            if t > bounding_box_size * 2: # Escape condition
                break
                
            t += dist
            
    return surface_points
```

### Pseudocode: Explicit PDDF Sampling
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sample_pddf_surface(pddf_function, bounding_box_size=2.0, nv=128, nh=3):
    """
    Extracts points from PDDF by projecting random points.
    nv: Number of directions sampled per point.
    nh: Number of iterative refinements.
    """
    points = []
    half_box = bounding_box_size / 2.0
    
    for _ in range(10000): # Fixed number of initial samples
        # 1. Uniform sample in bounding box
        p = [
            random.uniform(-half_box, half_box),
            random.uniform(-half_box, half_box),
            random.uniform(-half_box, half_box)
        ]
        
        current_p = p
        for _ in range(nh):
            # 2. Sample nv directions
            best_v = None
            best_dist = float('inf')
            
            for _ in range(nv):
                # Uniform direction on sphere
                theta = random.uniform(0, 2 * math.pi)
                phi = math.acos(random.uniform(-1, 1))
                v = [
                    math.sin(phi) * math.cos(theta),
                    math.sin(phi) * math.sin(theta),
                    math.cos(phi)
                ]
                
                # 3. Estimate best direction to avoid visibility issues
                # (Simplified: assume any direction works if PDDF is robust, 
                # or use heuristic to pick direction with non-zero depth)
                # Note: The source mentions composing directions to estimate vb* 
                # without optimization. Here we pick the first valid projection.
                
                # 4. Compute surface point
                # q = p + d(p, v) * v
                # This requires PDDF to return depth d(p, v)
                depth = pddf_function.get_depth(current_p, v)
                
                if depth is not None and depth > 0:
                    q = [current_p[i] + depth * v[i] for i in range(3)]
                    # Check if q is on surface (xi=1)
                    # Assuming PDDF handles this internally or via threshold
                    if pddf_function.is_on_surface(q):
                        current_p = q
                        break
                        
        points.append(current_p)
        
    return points
```

## Subtopics

### [Near-Surface Point Sampling](2910-near-surface-point-sampling.md)
Near-Surface Point Sampling generates training data points in the immediate vicinity of a mesh surface by perturbing existing surface samples with Gaussian noise. It solves the problem of insufficient volumetric coverage near the geometry boundary, which is critical for training neural implicit representations to converge on the correct surface location. It differs from uniform volume sampling by focusing density on the surface transition region, and from pure surface sampling by providing gradient information for the signed distance field (SDF) values off the zero-level set.

### [Surfel Parameterization from Mesh Sampling](4501-surfel-parameterization-from-mesh-sampling.md)
This method estimates the full parameter set (position, normal, tangent vectors, and scales) for a surfel representation starting from a triangle mesh. It solves the problem of defining local tangent frames and anisotropic kernel shapes for point clouds sampled from meshes, where positions and normals are known but geometric extent is not. It differs from sibling methods by deriving tangent scales from the local point cloud neighborhood structure rather than mesh connectivity or fixed heuristics, offering two variants: isotropic scaling for performance and anisotropic scaling for quality.

### [SDF-guided Sampling](7324-sdf-guided-sampling.md)
SDF-guided Sampling is a surface-aware sampling strategy that uses a Signed Distance Field (SDF) to estimate ray-surface intersections, followed by fine-grained sampling around the intersection point to accelerate rendering and emphasize the relationship between surface and radiance fields. It differs from standard NeRF sampling by reducing the number of required sampling points through early surface localization, and from pure mesh-based methods by retaining the flexibility of implicit field evaluation without requiring explicit mesh vertex interpolation.

### [Bidirectional Projective Sampling](8824-bidirectional-projective-sampling.md)
Bidirectional Projective Sampling is a boundary sampling method for physics-based differentiable rendering that generates seed rays from both sensors and emitters to initialize a guiding structure, addressing inefficiencies when scenes contain small-area lights at a distance. It improves upon Zhang et al.'s projective sampling by incorporating rays emitted directly from small-area emitters into the seed ray set during the projection operation, ensuring all successfully projected rays contribute to the guiding structure regardless of emitter size. Unlike previous methods that rely solely on sensor-spawned rays, which have a near-zero probability of intersecting small-area emitters, this method guarantees intersection by explicitly sampling from the emitter, thereby maximizing contribution to the boundary integral.

### [Multi-view Feature Consistency via Linear Interpolation](10227-multi-view-feature-consistency-via-linear-interpolation.md)
This method extracts a differentiable surface point from volume rendering samples by linearly interpolating SDF zero-crossings, enabling the enforcement of multi-view feature consistency constraints. It differs from sibling methods by relying on the hierarchical sampling density near the surface to approximate the zero-crossing with a single secant step, rather than recursive root-finding or mesh-based parameterization.

### Projection Point Integration
**Definition**: The incorporation of projection points derived from gradient information into the power diagram computation to accurately capture the underlying surface, particularly in regions with complex topology or fine details [S3].
**Steps**: Projection points are especially valuable for reconstructing thin structures, sharp edges, and regions with high curvature variation that might otherwise be inadequately captured by discrete SDF sampling points alone [S3].
**Comparison**: The baseline method using only sampling points without projections exhibits substantial topological defects and missing features, whereas the projection-enhanced approach preserves topological integrity [S3].

### Surface Interaction Construction
**Definition**: Constructs a `SurfaceInteraction3f` structure from a ray and intersection time, computing normal, frame, and other interaction properties without SDF autodiff support [S4, unofficial].
**Steps**: Set `si.t = t`, `si.p = ray(t)`, `si.sh_frame.n = normalize(gradient(si.p))`, initialize sh frame, set `si.n = si.sh_frame.n`, `si.wi = to_local(-ray.d)`, `si.wavelengths = ray.wavelengths`, `si.dp_du = sh_frame.s`, `si.dp_dv = sh_frame.t` [S4, unofficial].

### Explicit Sampling for Point Cloud Extraction
**Definition**: A method to extract a point cloud (PC) from a Probabilistic Directed Distance Field (PDDF) by projecting random points onto the shape surface S. It does not guarantee uniform sampling over S [S5].
**Steps**: Sample a point p uniformly from the bounding box B: p ~ U[B]. For each p, sample nv directions vi uniformly from the unit sphere S^2: V(p) = {vi(p) ~ U[S^2]}_{i=1}^{nv}. Compose these directions to estimate the best direction vb*(p) (as in §V-A2 and §V-A3, but without optimization) to avoid visibility ξ(p, v) = 0. Compute the surface point q(p, vb*(p)) using the PDDF depth and direction. The condition for q being on S is ξ(p, v) = 1 and q(p, v) = p + d(p, v)v. Repeat the process NH times, starting from the newly found point q (i.e., p <- q), to improve accuracy for depths far from S [S5].
**Parameters**: nv: Number of directions sampled per point. Typical value: 128 [S5].

## Operating characteristics
*   **Time Complexity**: Depends on the number of rays and steps in sphere tracing. Sphere tracing is $O(N \cdot S)$ where $N$ is the number of rays and $S$ is the average number of steps to convergence [S1]. Indicator grid sampling adds overhead for grid lookups and MLP inference [S2].
*   **Memory Complexity**: Low for sphere tracing (only storing points). Indicator grid sampling requires storing the indicator grid and field levels [S2].
*   **Strengths**: Efficiently targets the surface region, reducing wasted samples in empty space. Provides necessary gradients for training implicit fields [S1].
*   **Limitations**: Sphere tracing can be slow for complex SDFs with many steps. Explicit PDDF sampling does not guarantee uniform distribution [S5].
*   **When to Use**: Training neural implicit representations (NGLOD), rendering dynamic deformable tissues, extracting point clouds from PDDFs.
*   **When Not to Use**: When uniform volumetric coverage is required (e.g., standard NeRF without guidance), or when explicit mesh topology is available and preferred.

## Background
Surface Point Sampling for SDFs originated with the Neural Geometric Level of Detail (NGLOD) paper, which introduced sphere tracing for generating surface and near-surface training points [S1]. It was further developed in Efficient Deformable Tissue Reconstruction using indicator grids for dynamic scenes [S2]. Explicit sampling methods were introduced for Probabilistic Directed Distance Fields (PDDF) to extract point clouds [S5]. Power diagram enhancements with projection points were proposed for adaptive isosurface extraction [S3].

## Sources
[S1] Neural Geometric Level of Detail: Real-time Rendering with Implicit 3D Shapes, https://doi.org/10.1109/cvpr46437.2021.01120, fetched 2026-10-26
[S2] Efficient Deformable Tissue Reconstruction via Orthogonal Neural Plane, https://doi.org/10.1109/tmi.2024.3388559, fetched 2026-10-26
[S3] Power Diagram Enhanced Adaptive Isosurface Extraction From Signed Distance Fields., https://doi.org/10.1109/tvcg.2026.3685989, fetched 2026-10-26
[S4] zichenwang01/relaxed-boundary, https://github.com/zichenwang01/relaxed-boundary, fetched 2026-10-27 (unofficial)
[S5] Probabilistic Directed Distance Fields for Ray-Based Shape Representations, https://doi.org/10.1109/tpami.2025.3594225, fetched 2026-10-29
