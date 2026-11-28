# RBF Particle Volume Path Tracing

Also called: RBF volume rendering, SPH volume rendering, Smoothed Particle Hydrodynamics (SPH) volume rendering

## Overview
RBF Particle Volume Path Tracing renders volumetric particle data by modeling the scene as a continuous scalar field composed of additive weighted radial basis functions (RBFs), specifically Gaussians [S1, book]. It solves the visual clutter and pathological BVH performance issues of overlapping splatted particles by treating the volume as a unified field, enabling full path tracing through integration into a volume sampling kernel API [S1, book].

## How it works
The method combines a Bounding Volume Hierarchy (BVH) for spatial acceleration with Delta Tracking (Woodcock tracking) for unbiased Monte Carlo integration of the Radiative Transfer Equation (RTE) [S1, book].

### Inputs and Outputs
*   **Inputs**:
    *   Particle dataset: $N$ particles, each defined by position $\mathbf{x}_i$, radius $r_i$, and weight $w_i$ [S1, book].
    *   Rendering parameters: `radiusSupportFactor` ($\sigma$, default 3), `clampMaxCumulativeValue` (optional threshold) [S1, book].
    *   Transfer functions: Color map $C(\cdot)$ and opacity map $\alpha(\cdot)$ mapping scalar field values to color and extinction [S1, book].
    *   Ray: Origin $\mathbf{o}$, direction $\omega$ [G].
*   **Outputs**:
    *   Radiance $L(0, \omega)$ along the ray [S1, book].

### Data Structures
1.  **Particle Volume**:
    *   Arrays of size $N$: `positions` ($\mathbb{R}^3$), `radii` ($\mathbb{R}$), `weights` ($\mathbb{R}$) [S1, book].
2.  **BVH**:
    *   Built using Embree [S1, book].
    *   Leaf nodes: Each leaf corresponds to a single particle [S1, book].
    *   Node bounds: Determined by the particle's base radius multiplied by `radiusSupportFactor` ($\sigma r_i$) [S1, book].
    *   Value Ranges: Each node stores the minimum and maximum scalar field values $\Phi$ achievable within its bounding volume [S1, book]. These are estimated during preprocessing [S1, book].

### Preprocessing
1.  **BVH Construction**: Build a BVH over the $N$ particles. The bounding box for particle $i$ is expanded by a factor of $\sigma$ (default 3) to account for the Gaussian kernel's support [S1, book].
2.  **Value Range Estimation**:
    *   For each leaf node (particle $i$), evaluate the RBF field $\Phi$ on a small grid (e.g., $10^3$ points) within the support radius $\sigma r_i$ [S1, book].
    *   Determine the local minimum and maximum values for that leaf [S1, book].
    *   Propagate these ranges up the tree to update interior node ranges [S1, book].
    *   Store the global minimum and maximum values for the entire dataset [S1, book].
    *   *Note*: This step has worst-case complexity $O(N)$ and is optional if the user can manually specify `clampMaxCumulativeValue` [S1, book].

### Mathematical Formulation

**1. RBF Scalar Field**
The scalar field $\Phi(\mathbf{x})$ at query point $\mathbf{x}$ is the sum of Gaussian kernels from all contributing particles $i$:
$$
\Phi(\mathbf{x}) = \sum_i \phi_i(d_i(\mathbf{x}))
$$
where $d_i(\mathbf{x}) = \|\mathbf{x} - \mathbf{x}_i\|$ is the distance from particle center $\mathbf{x}_i$ [S1, book].

**2. Gaussian Kernel**
The kernel for particle $i$ with weight $w_i$ and radius $r_i$ is:
$$
\phi_i(\mathbf{x}) = w_i e^{-\frac{1}{2}\left(\frac{d_i(\mathbf{x})}{r_i}\right)^2}
$$
The kernel is truncated at a support radius of $\sigma r_i$. Outside this radius, the contribution is zero [S1, book].

**3. Radiative Transfer Equation (RTE)**
Assuming non-scattering media ($\sigma_s = 0$), the RTE simplifies to:
$$
L(0, \omega) = \int_0^\infty T(0, t) \cdot \sigma_e(t) L^e(t, \omega) dt
$$
where $T(0, t) = \exp\left(-\int_0^t \sigma_t(s) ds\right)$ is the transmittance, $\sigma_e(t)$ is the emission coefficient, and $\sigma_t(t)$ is the extinction coefficient [S1, book].

**4. Transfer Function Application**
The emission and extinction coefficients are derived from the scalar field $\Phi(t)$ evaluated along the ray:
$$
L^e(t, -\omega) = L^e(C(\Phi(t)), -\omega)
$$
$$
T(t, -\omega) = T(\alpha(\Phi(t)), -\omega)
$$
Where $C$ is a color map and $\alpha$ is an opacity/extinction map [S1, book].

**5. Delta Tracking (Woodcock Tracking)**
To sample distances unbiasedly in the non-homogeneous medium defined by $\Phi(t)$:
*   Estimate $\sigma_{t,\max}$, the maximum extinction coefficient along the ray. This is estimated during preprocessing by evaluating the RBF field over a grid, or manually specified by the user [S1, book].
*   Sample a distance $t$ to the next event:
    $$
    t = -\frac{\ln(1-\xi)}{\sigma_{t,\max}}
    $$
    where $\xi \in [0, 1)$ is a uniform random number [S1, book].
*   Evaluate $\sigma_t(t)$ at the sampled point [S1, book].
*   Reject the event as "virtual" with probability:
    $$
    P[\text{virtual}] = 1 - \frac{\sigma_t(t)}{\sigma_{t,\max}}
    $$
    If rejected, sample a new distance $t$ from the current position and repeat [S1, book].

### Algorithm Steps
1.  **Initialize**: Set accumulated radiance $L_{acc} = 0$, optical depth $\tau = 0$, and current ray distance $t_{curr} = 0$ [G].
2.  **Ray Marching Loop**:
    a.  Sample distance $t$ using Delta Tracking formula with $\sigma_{t,\max}$ [S1, book].
    b.  Compute sample position $\mathbf{x} = \mathbf{o} + t\omega$ [G].
    c.  **Evaluate Field**:
        i.  Traverse BVH to find all particles whose support radius intersects $\mathbf{x}$ [S1, book].
        ii. For each intersecting particle $i$, compute $d_i(\mathbf{x})$.
        iii. If $d_i(\mathbf{x}) < \sigma r_i$, compute $\phi_i(\mathbf{x}) = w_i e^{-\frac{1}{2}(d_i(\mathbf{x})/r_i)^2}$ [S1, book].
        iv. Accumulate $\Phi(\mathbf{x}) = \sum \phi_i(\mathbf{x})$.
        v.  **Early Termination**: If `clampMaxCumulativeValue` is set and $\Phi(\mathbf{x})$ exceeds it, clamp $\Phi(\mathbf{x})$ to this value and stop summing further particles [S1, book].
    d.  **Apply Transfer Function**:
        i.  Compute $\sigma_t = \alpha(\Phi(\mathbf{x}))$ [S1, book].
        ii. Compute $L^e = C(\Phi(\mathbf{x}))$ [S1, book].
    e.  **Check Virtual Event**:
        i.  Generate $\xi \in [0, 1)$. If $\xi < 1 - \sigma_t/\sigma_{t,\max}$, the event is virtual; continue loop without advancing $t_{curr}$ [S1, book].
        ii. If accepted, update accumulated radiance: $L_{acc} \leftarrow L_{acc} + e^{-\tau} \cdot \sigma_e L^e / \sigma_{t,\max}$ [S1, book].
        iii. Update optical depth: $\tau \leftarrow \tau + \sigma_t \cdot t$ [G].
        iv. Advance ray: $t_{curr} \leftarrow t_{curr} + t$ [G].
3.  **Return** $L_{acc}$ [G].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rbf_volume_path_trace(ray_origin, ray_direction, bvh, particles, transfer_func, sigma_t_max, clamp_max, max_steps=100):
    """
    ray_origin: vec3
    ray_direction: vec3
    bvh: BVH structure built on particles
    particles: list of {position, radius, weight}
    transfer_func: function(phi) -> (sigma_t, emission_color)
    sigma_t_max: float, maximum extinction coefficient
    clamp_max: float, optional clamp for accumulated phi
    """
    accumulated_radiance = 0.0
    optical_depth = 0.0
    t_current = 0.0
    
    for _ in range(max_steps):
        if exp(-optical_depth) < 1e-4:
            break
            
        # 1. Sample distance using Delta Tracking
        xi = uniform(0.0, 1.0)
        t_step = -log(1.0 - xi) / sigma_t_max
        t_sample = t_current + t_step
        
        # 2. Evaluate Scalar Field Phi at sample point
        sample_pos = ray_origin + t_sample * ray_direction
        phi_value = evaluate_rbf_field(sample_pos, bvh, particles, clamp_max)
        
        # 3. Apply Transfer Function
        sigma_t, emission_color = transfer_func(phi_value)
        
        # 4. Check for Virtual Event
        acceptance_prob = sigma_t / sigma_t_max
        if uniform(0.0, 1.0) < acceptance_prob:
            # Event is real
            # Transmittance from origin to sample point
            transmittance = exp(-optical_depth)
            # Contribution to radiance: T(t) * sigma_e * L_e / sigma_t_max
            # Assuming emission_color represents sigma_e * L_e
            accumulated_radiance += transmittance * emission_color / sigma_t_max
            
            # Update optical depth for next steps
            optical_depth += sigma_t * t_step
            t_current = t_sample
        # If virtual, we simply sample a new distance from the current position (loop continues)
        
    return accumulated_radiance

def evaluate_rbf_field(sample_pos, bvh, particles, clamp_max):
    phi = 0.0
    candidates = bvh.traverse(sample_pos)
    for particle in candidates:
        delta = sample_pos - particle.position
        dist_sq = dot(delta, delta)
        radius_sq = particle.radius * particle.radius
        support_factor = 3.0
        if dist_sq < (support_factor * particle.radius) ** 2:
            val = particle.weight * exp(-0.5 * dist_sq / radius_sq)
            phi += val
            if clamp_max > 0.0 and phi >= clamp_max:
                phi = clamp_max
                break
    return phi
```

### Parameters
*   `radiusSupportFactor` ($\sigma$): Multiple of base particle radius for BVH bounds and kernel truncation. Default: 3. Smaller values improve performance but may cause truncation artifacts [S1, book].
*   `clampMaxCumulativeValue`: Threshold for accumulated $\Phi$. If exceeded, evaluation terminates early. Useful in regions of high overlap to preserve contours at the cost of dynamic range [S1, book].

### Edge Cases and Pitfalls
*   **Overlapping Particles**: In splatting, overlapping particles cause visual clutter and BVH performance issues. RBF volume rendering handles this by treating the volume as a continuous field [S1, book].
*   **$\sigma_{t,\max}$ Estimation**: Worst-case complexity $O(N)$ for value estimation. Costly for large datasets. It is only necessary for small datasets with unpredictable overlap or if the user cannot guess the cumulative maximum. For large datasets, manual specification of `clampMaxCumulativeValue` is recommended to skip preprocessing [S1, book].
*   **Truncation Artifacts**: Setting `radiusSupportFactor` too low can cause visible artifacts where the Gaussian kernel is abruptly cut off [S1, book].

## Operating characteristics
*   **Time Complexity**: The preprocessing step for value estimation has worst-case time complexity $O(N)$, where $N$ is the number of particles [S1, book]. Rendering complexity depends on the number of particles intersected by the ray and the depth of the BVH.
*   **Memory Complexity**: Requires storing the particle data ($N$ particles) and the BVH structure.
*   **Performance**: Allows interactive rendering of hundreds of millions of particles at high resolution (e.g., 4000) without simplification or proxy geometry [S1, book].
*   **Strengths**:
    *   Handles overlapping particles without visual clutter.
    *   Supports full path tracing (unlike previous RBF implementations restricted to ray casting) [S1, book].
    *   Adjustable smoothing/sharpening via kernel radius and weight [S1, book].
    *   Analytical derivatives available for shading [S1, book].
*   **Limitations**:
    *   Value estimation for $\sigma_{t,\max}$ can be costly for large, complex datasets [S1, book].
    *   Requires careful tuning of `radiusSupportFactor` to balance performance and artifact avoidance [S1, book].
*   **When to Use**: Suitable for scientific visualization of particle-based data (e.g., SPH fluids, cosmology) where continuous field representation is desired and full path tracing is needed.
*   **When Not to Use**: If the dataset is extremely large and $\sigma_{t,\max}$ estimation is prohibitive, and manual clamping is not feasible, or if simple splatting is sufficient and path tracing is not required.

## Background
RBF volume rendering is frequently referred to as a Smoothed Particle Hydrodynamics (SPH) approach due to its use in visualizing SPH fluid dynamics codes [S1, book]. The method was implemented in Intel OSPRay and the Open Volume Kernel Library (VKL) using ISPC [S1, book]. It builds on the Woodcock-tracking volume path tracer of Raab and integrates RBF reconstruction into a volume sampling kernel API, leveraging interval iterators for space skipping [S1, book]. The approach addresses limitations of earlier RBF implementations that were restricted to ray casting [S1, book].

## Sources
[S1] Path Tracing RBF Particle Volumes, https://link.springer.com/chapter/10.1007/978-1-4842-7185-8_44, fetched 2026-10-25 (book)
