# IRIS Ray-Geometry Intersection

Also called: OptiX-based Gaussian Intersection, Proxy Geometry Intersection

## Overview
IRIS is an intersection-aware ray-based sampling method for implicit editable scenes that determines samples as the maximum response points of Gaussians intersected by each ray [S1]. It solves the problem of efficiently gathering depth samples from 3D Gaussian primitives in a hardware-accelerated ray tracing pipeline [S1]. It sets itself apart from standard BVH traversal methods by replacing explicit Gaussian volume checks with proxy geometry (affine-transformed icosahedra) and leveraging OptiX Any-hit shaders to compute sampling parameters directly during intersection tests [S1].

## How it works
**Inputs:** Ray origin $O$ and direction $V$ in world space; a set of 3D Gaussians defined by mean $\mu$, covariance $\Sigma$, and opacity $\alpha$; target sample quota (hyperparameter).
**Outputs:** A list of samples, where each sample is a 2-tuple $(t_{\text{sample}}, \text{Gaussian index})$.

**Data Structures:**
1.  **Proxy Geometry:** Each Gaussian is bounded by a regular, affine-transformed icosahedron [S1]. The size of the icosahedron matches the confidence ellipsoid of the underlying $\chi^2$ distribution with 3 degrees of freedom [S1].
    *   **Canonical icosahedron:** Triangle vertices and indices are stored once in global memory [S1].
    *   **Instance Buffer:** Populated by a dedicated CUDA kernel. Each entry contains an affine transformation matrix $M$ that maps the reference proxy to the specific Gaussian primitive [S1].
2.  **Hit Buffer:** A fixed-size buffer storing 2-tuples of $(t_{\text{sample}}, \text{Gaussian index})$ [S1].
    *   **Size:** 16 entries [S1].
    *   **Content:** $t_{\text{sample}}$ is the intersection parameter; the index identifies the Gaussian.
3.  **Output Buffer:** Accumulates final samples transferred from the hit buffer [S1].

**Parameters:**
*   **Hit Buffer Capacity:** 16 entries [S1].
*   **Target Sample Quota (IPR - Intersections Per Ray):**
    *   Bounded scenes (NeRF Synthetic): 128 IPR [S1].
    *   Unbounded scenes (Mip-NeRF 360): 256 IPR [S1].
*   **Bounding Proxy Threshold ($\lambda$):** Squared Mahalanobis distance threshold defining the proxy size [S1].
    *   Bounded scenes: $\lambda = 6.25$ (covers ~90% of Gaussian probability mass) [S1].
    *   Unbounded scenes: $\lambda = 11.3449$ (covers 99% of Gaussian probability mass) [S1].

**Steps:**
1.  **Proxy Construction:** For each Gaussian, compute the affine transformation matrix $M$ that scales and rotates the canonical icosahedron to match the Gaussian's confidence ellipsoid defined by $\lambda$ [S1]. Populate the OptiX instance buffer with these matrices [S1].
2.  **Ray Generation:** For each pixel, generate a ray $(O, V)$ [S1].
3.  **Intersection Testing:** Perform intersection testing using OptiX Ray Generation and Any-hit shaders [S1].
4.  **Any-Hit Shader Execution:**
    a.  When a ray intersects a proxy icosahedron instance, transform the ray origin $O$ and direction $V$ into the object coordinate frame of the underlying Gaussian:
        $$O_{\text{obj}} = M^{-1}(O - \mu)$$
        $$V_{\text{obj}} = M^{-1}V$$
        [Note: The source defines $O_{\text{obj}}$ and $V_{\text{obj}}$ as the ray origin and direction in the object coordinate frame. The exact transformation depends on the definition of the proxy's local frame relative to the Gaussian. The source formula (4) uses these object-space vectors directly.]
    b.  Compute the intersection parameter $t_{\text{sample}}$ using the formula [S1]:
        $$t_{\text{sample}} = -\frac{\langle O_{\text{obj}}, V_{\text{obj}} \rangle}{\langle V_{\text{obj}}, V_{\text{obj}} \rangle}$$
        where $\langle \cdot, \cdot \rangle$ denotes the dot product [S1].
    c.  Store the tuple $(t_{\text{sample}}, \text{Gaussian index})$ in the 16-entry hit buffer [S1].
    d.  Sort the hit buffer by $t_{\text{sample}}$ to maintain order [S1].
5.  **Buffer Transfer:** If the hit buffer reaches capacity (16 entries), transfer the sorted samples from the hit buffer to the output buffer in the Ray Generation shader [S1].
6.  **Iteration:** Repeat steps 3-5 iteratively until the total number of gathered samples in the output buffer reaches the predefined target sample quota (IPR) [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def iris_ray_geometry_intersection(ray_origin, ray_direction, gaussians, target_ipr, buffer_size=16):
    """
    Simulates the IRIS ray-geometry intersection logic.
    Note: This is a high-level simulation. The actual method uses OptiX shaders.
    """
    # 1. Pre-compute proxy geometries (Affine icosahedra) for all Gaussians
    # In practice, this is done via a CUDA kernel populating the OptiX instance buffer.
    proxies = []
    for i, gaussian in enumerate(gaussians):
        # Determine lambda based on scene type (bounded vs unbounded)
        # For this simulation, we assume a generic lambda or pass it as arg
        lambda_val = gaussian.lambda_threshold 
        # Compute affine transformation matrix M for the icosahedron
        # M maps canonical icosahedron to Gaussian's confidence ellipsoid
        M = compute_affine_matrix(gaussian.mean, gaussian.covariance, lambda_val)
        proxies.append({
            'index': i,
            'matrix': M,
            'inverse_matrix': invert_matrix(M),
            'mean': gaussian.mean
        })

    # 2. Initialize buffers
    hit_buffer = [] # List of (t_sample, gaussian_index)
    output_buffer = [] # Final samples
    sample_count = 0

    # 3. Ray Tracing Simulation
    # In reality, this is handled by OptiX traversal.
    # We simulate the intersection test for all proxies.
    
    # Collect all potential intersections
    all_intersections = []
    
    for proxy in proxies:
        # Transform ray to object space
        # O_obj = M_inv * (O - mean)
        # V_obj = M_inv * V
        # Note: The source formula uses O_obj and V_obj directly.
        # Assuming M includes translation to mean, or O_obj is relative.
        # Standard affine transform: P_obj = M_inv * (P_world - T)
        # If M is just rotation/scale, T is separate. 
        # Source says "object coordinate frame of the underlying Gaussian".
        # Let's assume M_inv handles the full inverse transform to Gaussian local space.
        
        # Simplified: Assume proxy bounding box check first (hardware accelerated)
        if ray_intersects_proxy(ray_origin, ray_direction, proxy):
            # Compute t_sample using formula (4)
            # O_obj and V_obj are in object frame
            O_obj = transform_point(proxy['inverse_matrix'], ray_origin - proxy['mean'])
            V_obj = transform_vector(proxy['inverse_matrix'], ray_direction)
            
            dot_O_V = dot_product(O_obj, V_obj)
            dot_V_V = dot_product(V_obj, V_obj)
            
            if dot_V_V != 0:
                t_sample = -dot_O_V / dot_V_V
                all_intersections.append((t_sample, proxy['index']))

    # 4. Sort and Select
    # Sort by t_sample to find closest intersections
    all_intersections.sort(key=lambda x: x[0])
    
    # Select top 'target_ipr' samples
    selected_samples = all_intersections[:target_ipr]
    
    # 5. Output
    # The hit buffer logic (16 entries) is an optimization to reduce optixTrace calls.
    # In simulation, we just output the final selected samples.
    return selected_samples

def compute_affine_matrix(mean, covariance, lambda_val):
    """
    Computes the affine transformation matrix for the proxy icosahedron.
    The size matches the confidence ellipsoid of the chi-squared distribution.
    """
    # Eigen decomposition of covariance
    eigenvalues, eigenvectors = eig(covariance)
    
    # Scale factors based on lambda (Mahalanobis distance threshold)
    # The proxy size is proportional to sqrt(lambda * eigenvalue)
    scales = sqrt(lambda_val * eigenvalues)
    
    # Construct rotation matrix from eigenvectors
    R = eigenvectors
    
    # Construct scaling matrix
    S = diag(scales)
    
    # Affine matrix M = [R*S, t; 0, 1]
    # Assuming the proxy is centered at mean
    M = identity(4)
    M[0:3, 0:3] = R @ S
    M[0:3, 3] = mean
    
    return M

def ray_intersects_proxy(ray_origin, ray_direction, proxy):
    """
    Simulates hardware-accelerated ray-triangle intersection for the icosahedron.
    """
    # In OptiX, this is handled by the RT core.
    # Here we return True for demonstration if we assume intersection.
    # Real implementation would check against the canonical icosahedron triangles
    # transformed by the instance matrix.
    return True # Placeholder for actual intersection test
```

**Edge Cases and Pitfalls:**
*   **Precision:** The formula for $t_{\text{sample}}$ involves division by $\langle V_{\text{obj}}, V_{\text{obj}} \rangle$. If the ray direction is zero or nearly zero in object space, this causes division by zero or numerical instability [S1].
*   **Boundary Coverage:** The proxy geometry is a conservative bound. Using $\lambda = 6.25$ covers ~90% of mass, and $\lambda = 11.3449$ covers 99%. If the target sample quota is not reached, it may be due to Gaussians falling outside the proxy bounds [S1].
*   **Buffer Overflow:** The 16-entry hit buffer is small. If many Gaussians are intersected, frequent transfers to the output buffer are required, which can be costly if not managed efficiently [S1].

## Operating characteristics
*   **Time Complexity:** Dominated by the number of ray-proxy intersections and the sorting/transfer operations. The use of OptiX hardware acceleration for ray-triangle intersection reduces CPU/GPU overhead [S1].
*   **Memory Complexity:** Low memory footprint due to geometry instancing (storing only one canonical icosahedron) [S1].
*   **Performance:** On NeRF Synthetic dataset, increasing IPR from 16 to 128 improves Avg. PSNR from 29.87 to 32.65 [S1].
*   **Strengths:** Hardware-accelerated intersection tests; efficient sample gathering via Any-hit shaders; reduced memory footprint via instancing [S1].
*   **Limitations:** Proxy geometry is an approximation; may miss samples if Gaussians are not well-covered by the icosahedron bounds [S1].
*   **When to Use:** Implicit editable scenes requiring efficient ray-based sampling from Gaussians [S1].
*   **When Not to Use:** Scenes where exact Gaussian volume intersection is critical and proxy approximation leads to significant error [S1].

## Background
IRIS was introduced in the paper "IRIS: Intersection-aware Ray-based Implicit Editable Scenes" [S1]. It builds on the 3DGRT OptiX API-based approach [S1]. The method is implemented in PyTorch within the Nerfstudio ecosystem [S1]. Experiments were conducted on NVIDIA RTX 5090 GPU [S1].

## Sources
[S1] IRIS: Intersection-aware Ray-based Implicit Editable Scenes, https://doi.org/10.48550/arXiv.2603.15368, fetched 2026-10-29
