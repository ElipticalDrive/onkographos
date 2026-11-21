# Near-Surface Point Sampling

Also called: DeepSDF sampling, Non-Uniform Surface Sampling

## Overview
Near-Surface Point Sampling generates a non-uniform distribution of query points for training implicit surface representations, specifically Signed Distance Fields (SDFs). It solves the problem of insufficient training data density near the object boundary, which is critical for accurate surface reconstruction. It sets itself apart from uniform volume sampling by concentrating samples in the immediate vicinity of the mesh, and from pure surface sampling by including a controlled noise perturbation to regularize the gradient field.

## How it works
**Inputs:**
- A watertight mesh $M$ with correct face normals.
- Number of points $N$ (default 500,000) [S2, unofficial].
- Standard deviation for noise $\sigma$ (default 0.01) [S1][S2, unofficial].
- Bounding radius $R$ (default 1.0, unit sphere) [S2, unofficial].

**Outputs:**
- A set of $N$ 3D points $P = \{p_1, \dots, p_N\}$.
- Corresponding SDF values $D = \{d_1, \dots, d_N\}$ and signs $S = \{s_1, \dots, s_N\}$.

**Data Structures:**
- Surface Point Cloud: A set of points on the mesh surface, typically stored in a kd-tree for nearest-neighbor queries.
- Training Batch: A list of tuples $(p_i, d_i, s_i)$.

**Steps:**
1. **Mesh Normalization:** Transform the mesh $M$ so that it fits inside a unit sphere centered at the origin. This ensures consistent scale for the noise perturbation and bounding volume [S2, unofficial].
2. **Surface Sampling:** Generate a set of points $P_{surf}$ uniformly distributed on the surface of $M$. The number of surface points is determined by the desired split ratio. According to the DeepSDF paper, a 2:2:1 split is used for surface, near, and uniform samples, meaning $N_{surf} = N_{near} = \frac{2}{5}N$ and $N_{uniform} = \frac{1}{5}N$ [S1].
3. **Near-Surface Perturbation:** For each point $p \in P_{surf}$, generate a perturbed point $p'$ by adding Gaussian noise:
   $$ p' = p + \epsilon, \quad \epsilon \sim \mathcal{N}(0, \sigma^2 I) $$
   where $\sigma = 0.01$ [S1][S2, unofficial]. These points form the set $P_{near}$.
4. **Uniform Volume Sampling:** Generate $N_{uniform}$ points uniformly distributed within the bounding sphere of radius $R$ [S1]. These points form the set $P_{uniform}$.
5. **SDF Calculation:** For all points in $P_{near} \cup P_{uniform}$, calculate the SDF value and sign:
   - **Value:** Find the closest point $q$ on the surface point cloud using a kd-tree. The distance is $d = \|p' - q\|$ [S2, unofficial].
   - **Sign:** Determine if the point is inside or outside the surface.
     - *Normal Method:* Compute the dot product of the vector $(p' - q)$ and the normal $n_q$ at $q$. If $(p' - q) \cdot n_q < 0$, the point is inside (negative sign) [S2, unofficial].
     - *Depth Method:* Project the point into the camera frame of each scan. If the point's depth is greater than the depth buffer value for any camera, it is outside (positive sign) [S2, unofficial].
6. **Validation:** Check the fraction of points inside the shape (negative SDF). If this fraction is less than `min_size` (default 0), throw an exception [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def near_surface_point_sampling(mesh, N=500000, sigma=0.01, R=1.0, min_size=0.0):
    # 1. Normalize mesh to unit sphere
    mesh = normalize_to_unit_sphere(mesh)
    
    # 2. Determine counts based on 2:2:1 split
    N_surf = int(2/5 * N)
    N_near = int(2/5 * N)
    N_uniform = N - N_surf - N_near
    
    # 3. Generate surface points
    P_surf = sample_mesh_surface(mesh, N_surf)
    
    # 4. Generate near-surface points via perturbation
    noise = np.random.normal(0, sigma, (N_near, 3))
    P_near = P_surf[:N_near] + noise
    
    # 5. Generate uniform volume points
    P_uniform = sample_uniform_in_sphere(R, N_uniform)
    
    # 6. Combine all points
    P_all = np.vstack([P_near, P_uniform])
    
    # 7. Calculate SDF values and signs
    # Assuming get_surface_point_cloud returns a kd-tree and normals
    point_cloud_data = get_surface_point_cloud(mesh, surface_point_method='sample')
    kd_tree = point_cloud_data.kd_tree
    normals = point_cloud_data.normals
    
    D = np.zeros(N)
    S = np.zeros(N)
    
    for i, p in enumerate(P_all):
        # Find closest point on surface
        dist, idx = kd_tree.query(p)
        q = kd_tree.data[idx]
        n = normals[idx]
        
        D[i] = dist
        
        # Determine sign using normal method
        vec = p - q
        if np.dot(vec, n) < 0:
            S[i] = -1.0 # Inside
        else:
            S[i] = 1.0  # Outside
            
    # 8. Validate min_size
    fraction_inside = np.sum(S == -1.0) / N
    if fraction_inside < min_size:
        raise Exception(f"Too few points inside shape: {fraction_inside} < {min_size}")
        
    return P_all, D, S
```

**Edge Cases and Pitfalls:**
- **Non-watertight meshes:** The sampling method requires watertight meshes with correct face normals. Non-watertight meshes may produce incorrect surface point clouds [S2, unofficial].
- **Sign Ambiguity:** The normal method can produce "bubble" artifacts in concave regions or near holes. The depth method is more robust but less accurate [S2, unofficial].
- **Precision:** The noise $\sigma$ is small relative to the unit sphere scale. Ensure floating-point precision is sufficient for the perturbation [S1].

## Subtopics

### [Distance Field Foot Point Generation](6946-distance-field-foot-point-generation.md)
Projects grid voxels onto the isosurface using the gradient. It differs by using a single-step projection based on unit-length gradients, whereas Near-Surface Point Sampling uses random Gaussian perturbation of existing surface points.

### [Primary Edge Sampling](7168-primary-edge-sampling.md)
Samples visibility discontinuities (silhouettes). It differs by targeting the geometric condition where the surface normal is perpendicular to the view vector, rather than perturbing surface points.

### [Surface and Volume Training Position Sampling](9996-surface-and-volume-training-position-sampling.md)
Mixes uniform volume, uniform surface, and perturbed surface samples. It differs by using area-weighted triangle selection and explicit mixing strategies, whereas Near-Surface Point Sampling is a specific perturbation technique often used within such frameworks.

### [Spot Noise (Motion Blurred Particles near Contour Surfaces)](13068-spot-noise-motion-blurred-particles-near-contour-surfaces.md)
Visualizes velocity fields near contour surfaces. It builds on Near-Surface Point Sampling by using distance-field estimation to restrict particle placement to a band around the isosurface.

### [Surface Sampling for NeRF Rendering](14054-surface-sampling-for-nerf-rendering.md)
Generates points on mesh triangles for NeRF training. It differs by focusing on uniform spatial distribution over mesh geometry for training data generation, rather than foot-point projection or edge-based sampling.

## Operating characteristics
- **Time Complexity:** Dominated by the kd-tree query for SDF calculation, which is $O(N \log M)$ where $M$ is the number of surface points in the cloud [S2, unofficial].
- **Memory Complexity:** $O(M)$ to store the surface point cloud and kd-tree [S2, unofficial].
- **Performance:** Training is performed on NVIDIA Tesla V100 GPUs using PyTorch with a batch size of 512 [S1].
- **Strengths:** Provides high-density samples near the surface, which improves the accuracy of the learned SDF [S1].
- **Limitations:** Requires a watertight mesh and correct normals for the sampling method [S2, unofficial]. The sign determination can be ambiguous in complex geometries [S2, unofficial].

## Background
The method was proposed in the DeepSDF paper [S2, unofficial]. It is used in neural implicit surface reconstruction to provide training data that emphasizes the surface boundary. The implementation is available in the `mesh_to_sdf` repository [S2, unofficial].

## Sources
[S1] Neural Geometric Level of Detail: Real-time Rendering with Implicit 3D Shapes, https://doi.org/10.1109/cvpr46437.2021.01120, fetched 2026-10-26
[S2] mesh_to_sdf, https://github.com/marian42/mesh_to_sdf, fetched 2026-10-28 (unofficial)
