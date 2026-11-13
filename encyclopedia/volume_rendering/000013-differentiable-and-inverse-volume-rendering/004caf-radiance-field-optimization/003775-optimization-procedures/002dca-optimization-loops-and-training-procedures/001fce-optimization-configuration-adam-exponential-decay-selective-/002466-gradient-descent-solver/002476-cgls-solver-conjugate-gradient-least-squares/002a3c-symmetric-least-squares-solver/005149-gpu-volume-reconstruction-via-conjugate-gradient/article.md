# GPU Volume Reconstruction via Conjugate Gradient

Also called: Implicit Matrix Solution, Ray Marching Back-Projection

## Overview
This method reconstructs spatially varying volume properties (albedo, emission, extinction) by solving the ill-conditioned linear system $Wa = o$ using Conjugate Gradient (CG) minimization on the GPU [S1]. It avoids explicit storage of the large sparse matrix $W$ by implicitly computing matrix-vector products via ray marching [S1]. Unlike [Symmetric Least Squares Solver](4723-symmetric-least-squares-solver.md), which uses SVD-based pseudoinverse, this approach leverages the sparsity of the rendering operator to converge faster without diagonalizing the system [S1].

## How it works
**Inputs:**
- Target images $o$ (observation pixels).
- Initial volume estimate $a$ (stored in 3D textures).
- Camera poses and intrinsics.
- Optional: Per-pixel constraint weights, visual hull masks, regularization weights.

**Outputs:**
- Reconstructed volume properties $a$ (albedo, emission, extinction) in 3D textures.

**Data Structures:**
- **Volume Properties ($a$):** Stored in 3D textures. Fields correspond to albedo, emission, and extinction. Precision can be 16-bit to reduce bandwidth [S1].
- **Target Images ($o$):** 2D textures or buffers.
- **Residual ($c$):** 2D buffer storing $Wa - o$.
- **Gradient/Direction Vectors:** 3D textures for volume updates.

**Formulas:**
The objective is to minimize the quadratic error function:
$$ E(a) = ||Wa - o||^2 $$
With regularization terms $\Omega(a)$:
$$ E_{reg}(a) = ||Wa - o||^2 + \lambda_L ||La||^2 + \lambda_0 ||a||^2 + \lambda_1 ||a - 1||^2 $$
Where:
- $W$ is the sparse rendering matrix.
- $L$ is the Laplacian operator.
- $\lambda_L, \lambda_0, \lambda_1$ are user-defined weights (typically weak, e.g., $2^{-9}$) [S1].
- Constraint weights $w_p$ are applied per pixel: $||W(a) \odot w - o \odot w||^2$ [S1].

**Numbered Steps:**
1. **Initialization:** Initialize $a$ (e.g., zeros or previous estimate). Compute initial residual $r_0 = o - Wa_0$. Set search direction $p_0 = r_0$.
2. **Iteration Loop:** For $k = 0, 1, \dots$ until convergence:
    a. **Matrix-Vector Product ($Wp_k$):** Render the volume with properties $p_k$ to generate an image. This is the forward pass.
    b. **Residual Update:** Compute $s_k = Wp_k$.
    c. **Gradient Norm:** Compute $\alpha_k = \frac{r_k^T r_k}{p_k^T W^T W p_k}$. Note: $p_k^T W^T W p_k = (Wp_k)^T (Wp_k) = s_k^T s_k$ if no regularization, or requires additional terms if regularization is present [S1].
    d. **Update Solution:** $a_{k+1} = a_k + \alpha_k p_k$.
    e. **Update Residual:** $r_{k+1} = r_k - \alpha_k W^T W p_k$. *Correction:* Standard CG updates residual as $r_{k+1} = r_k - \alpha_k W^T (W p_k)$. The term $W^T (W p_k)$ is the back-projection of the rendered image $s_k$.
    f. **Back-Projection ($W^T s_k$):** Distribute the rendered image $s_k$ back to the volume voxels. This is the backward pass.
    g. **Check Convergence:** If $||r_{k+1}|| < \epsilon$, stop.
    h. **Compute Beta:** $\beta_k = \frac{r_{k+1}^T r_{k+1}}{r_k^T r_k}$.
    i. **Update Search Direction:** $p_{k+1} = r_{k+1} + \beta_k p_k$.

**Pseudocode:**
The core operations are implemented as shaders. The unified ray marching loop handles both forward (rendering) and backward (back-projection) passes.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Unified Ray Marching Kernel
# mode: 'RENDER' or 'BACKPROJECT'
# volEmission, volAlbedo, volExtinction: 3D textures (vectors a)
# targetImage: 2D texture (o) or buffer for residual
# outputImage: 2D texture for rendered result or residual buffer
# backProjBuffer: 3D texture for accumulating back-projected values

def ray_marching_kernel(pixel_x, pixel_y, mode, camera_ray):
    # Initialize accumulation
    accumulated_transmittance = 1.0
    accumulated_radiance = 0.0
    accumulated_back_proj = 0.0
    
    # Ray traversal parameters
    t_min, t_max = camera_ray.intersect_volume()
    step_size = get_step_size(t_min, t_max) # Based on resolution
    
    t = t_min
    while t < t_max:
        # Sample volume properties at current position
        # For refractive rays, use Euler forward scheme to update ray direction
        # position = camera_ray.origin + t * camera_ray.direction
        
        emission = tex3D(volEmission, position)
        albedo = tex3D(volAlbedo, position)
        extinction = tex3D(volExtinction, position)
        
        # Weight calculation: exp(-sumExt) * (1 - exp(extinction))
        # This represents the contribution of the current voxel to the pixel
        weight = accumulated_transmittance * (1.0 - exp(-extinction * step_size))
        
        if mode == 'RENDER':
            # Forward pass: accumulate radiance
            # outputRadiance += weight * texRead(volEmission, ray)
            # Note: Albedo affects scattering, simplified here for emission-only or single bounce
            # For full scattering, integrate phase function and multiple bounces
            accumulated_radiance += weight * emission
            
        elif mode == 'BACKPROJECT':
            # Backward pass: back-project residual
            # Get residual value for this pixel from targetImage or residual buffer
            # This requires gathering the residual value corresponding to the current ray
            # In practice, this is done by storing residuals in a buffer accessible to the kernel
            residual = get_residual_for_pixel(pixel_x, pixel_y)
            
            # Apply constraint weights if present
            # residual = residual * constraint_weight[pixel_x, pixel_y]
            
            # Distribute residual to voxels
            # texWriteAdd(volEmission, ray, weight * backProj)
            # backProj is the residual scaled by weight
            accumulated_back_proj += weight * residual
            
        # Advance ray
        # For refraction: update direction based on refractive index
        # t += step_size
        
        accumulated_transmittance *= exp(-extinction * step_size)
        t += step_size
        
    if mode == 'RENDER':
        outputImage[pixel_x, pixel_y] = accumulated_radiance
    elif mode == 'BACKPROJECT':
        # Write accumulated back-projection values to volume textures
        # Use atomic add or shader_image_load_store for synchronization
        write_to_volume(position, accumulated_back_proj)

# Optimization Loop on GPU
def cg_optimization_loop():
    # Initialize a, r, p
    a = initialize_volume()
    r = compute_residual(a) # o - Wa
    p = r.copy()
    
    # Precompute r^T r
    r_dot_r = dot_product(r, r)
    
    for iteration in range(max_iterations):
        # 1. Compute Wp (Forward Pass)
        # Render volume with properties p
        Wp = render_volume(p)
        
        # 2. Compute W^T W p (Backward Pass of Wp)
        # Back-project Wp to get W^T(Wp)
        WTWp = backproject_volume(Wp)
        
        # 3. Compute alpha
        # alpha = (r^T r) / (p^T W^T W p)
        # p^T W^T W p = dot_product(p, WTWp)
        p_dot_WTWp = dot_product(p, WTWp)
        
        # Add regularization terms to denominator if present
        # For Laplacian: p^T L^T L p
        # For L2: p^T p
        # p_dot_WTWp += lambda_L * dot_product(Lp, Lp) + lambda_0 * dot_product(p, p)
        
        alpha = r_dot_r / p_dot_WTWp
        
        # 4. Update a
        a = a + alpha * p
        
        # 5. Update r
        # r_new = r - alpha * W^T W p
        r = r - alpha * WTWp
        
        # 6. Compute new r^T r
        new_r_dot_r = dot_product(r, r)
        
        # 7. Check convergence
        if new_r_dot_r < tolerance:
            break
            
        # 8. Compute beta
        beta = new_r_dot_r / r_dot_r
        
        # 9. Update p
        p = r + beta * p
        
        # 10. Update r_dot_r
        r_dot_r = new_r_dot_r

# Handling Synchronization in Back-Projection
def backproject_volume(image):
    # To avoid synchronization stalls from neighboring rays writing to same voxels,
    # use an interleaved pattern of 6x6 pixels.
    # In each round, only one ray from these sub-windows is shot.
    # This decreases conflicts and speeds up computation by ~10%.
    
    # Initialize back-projection buffer to zero
    back_proj_buffer = zeros_like(volume_texture)
    
    # Process pixels in interleaved pattern
    for sub_window in interleaved_6x6_windows():
        for pixel in sub_window:
            ray = get_camera_ray(pixel)
            # Perform back-projection kernel for this ray
            # Accumulate into back_proj_buffer using atomic operations
            back_proj_buffer += ray_marching_kernel(pixel, mode='BACKPROJECT', camera_ray=ray)
            
    return back_proj_buffer
```

**Edge Cases and Pitfalls:**
- **Synchronization Stalls:** Data scattering during back-projection (texture writes) causes stalls because neighboring rays write to the same voxels. Mitigation: Use an interleaved pattern of $6 \times 6$ pixels, shooting only one ray from each sub-window per round [S1].
- **Precision:** Use 16-bit textures instead of 32-bit to reduce bandwidth and speed up computation by ~25% [S1].
- **Resolution Mismatch:** Albedo and emission can use lower resolution than extinction to accelerate computation and reduce memory, though this may affect quality [S1].
- **Refractive Rays:** For non-constant refractive indices, use arbitrary integration curves computed via Euler forward scheme [S1].
- **Visual Hulls:** Limit solution domain to voxels with non-zero extinction (for albedo/emission) or user-defined masks (for extinction) to improve quality and performance [S1].

## Operating characteristics
- **Time Complexity:** Depends on the number of CG iterations and the cost of ray marching per iteration. Each iteration requires one forward render and one back-projection.
- **Memory Complexity:** $O(N_v)$ for volume storage, where $N_v$ is the number of voxels. No explicit storage of $W$ ($O(N_p N_v)$) is required [S1].
- **Performance:** 16-bit textures provide ~25% speedup. Interleaved pixel pattern provides ~10% speedup by reducing synchronization stalls [S1].
- **Strengths:** Handles large sparse systems without explicit matrix storage. Flexible regularization. Supports refraction and visual hulls.
- **Limitations:** Ill-conditioned systems may require careful regularization. Synchronization issues in back-projection can limit parallel efficiency.
- **When to Use:** When reconstructing volume properties from multiple views and explicit matrix storage is infeasible.
- **When Not to Use:** When high precision is critical and 16-bit textures are insufficient. When real-time performance is required and CG convergence is too slow.

## Background
This method was introduced in "Property and Lighting Manipulations for Static Volume Stylization Using a Painting Metaphor" (2014) [S1]. It builds on the linearity of light transport and the sparsity of the rendering operator. The use of Conjugate Gradient for volume reconstruction allows for efficient optimization of volume properties without the memory overhead of direct matrix inversion.

## Sources
[S1] Property and Lighting Manipulations for Static Volume Stylization Using a Painting Metaphor, https://doi.org/10.1109/TVCG.2014.13, fetched 2026-10-26
