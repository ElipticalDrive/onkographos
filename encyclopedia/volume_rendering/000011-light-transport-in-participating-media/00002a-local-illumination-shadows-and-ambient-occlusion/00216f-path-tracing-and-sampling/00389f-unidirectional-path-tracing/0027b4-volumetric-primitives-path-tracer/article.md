# Volumetric-Primitives Path Tracer

Also called: Volumetric-Primitives Radiance Field (VPRF)

## Overview
A forward rendering method for scattering and emissive media that models the scene as a collection of 3D kernel-based volumetric primitives and uses ray tracing to compute light transport, supporting multiple scattering via Next-Event Estimation [S1, unofficial]. Unlike [Volumetric Path Tracing](1863-volumetric-path-tracing.md) which typically relies on voxel grids or implicit fields requiring numerical integration, VPPT provides closed-form analytical solutions for transmittance and free-flight sampling, eliminating Monte Carlo noise for single-scattering events and enabling efficient hardware-accelerated traversal [S1, unofficial]. It maintains 3D volumetric density and integrates physically, preserving reciprocity and avoiding the view-dependent ordering artifacts and popping seen in delta-function approximations like [3D Gaussian Splatting](https://nvlabs.github.io/3dgs/) [S1, unofficial].

## How it works
**Inputs:**
- A set of $N$ volumetric primitives $\{P_i\}_{i=1}^N$.
- For each primitive $P_i$:
  - Spatial density distribution defined by a kernel function $K_i(\mathbf{x})$ (e.g., Gaussian or Epanechnikov).
  - Optical cross-section $\sigma_i$ (or $\tau_i$ in some notations, representing the peak optical depth scaling).
  - Emission term $Q_i(\omega)$.
  - Phase function $f_{p,i}(\omega' \to \omega)$.
- Camera ray origin $\mathbf{x}_0$ and direction $\omega$.
- Scene bounding volume hierarchy (BVH) built on primitive bounding boxes/shells.

**Outputs:**
- Radiance $L(\mathbf{x}_0, \omega)$ along the ray.

**Data Structures:**
- **Primitive List:** Array of $N$ primitives. Each primitive stores:
  - Kernel parameters (e.g., mean $\mathbf{c}_i$, covariance $\Sigma_i$ for Gaussian; center and radius for Epanechnikov).
  - Scalar optical cross-section $\sigma_i$.
  - Emission color/vector $Q_i$.
  - Phase function parameters.
  - Bounding box (AABB) for acceleration structure.
- **Segment List:** Ordered list of ray segments $S_k = [b_{t_{k,0}}, b_{t_{k,1}}]$ resulting from ray-primitive intersection tests. Each segment stores:
  - Start parameter $b_{t_{k,0}}$ and end parameter $b_{t_{k,1}}$ along the ray.
  - List of primitive indices overlapping this segment.

**Steps:**

1.  **Scene Representation:**
    Define the total extinction coefficient $\mu_t(\mathbf{x})$ at any point $\mathbf{x}$ as the sum of contributions from all primitives:
    $$
    \mu_t(\mathbf{x}) = \sum_{i=1}^{N} \sigma_i K_i(\mathbf{x})
    $$
    where $K_i(\mathbf{x})$ is the normalized kernel function for primitive $i$ [S1, unofficial].

2.  **Ray-Primitive Intersection and Segmentation:**
    Use hardware-accelerated ray tracing (e.g., BVH on OptiX) to find all primitives intersecting the camera ray.
    - Collect all entry and exit points (parameters $t$) where the ray enters or leaves any primitive's bounding volume.
    - Sort these points to define an ordered sequence of segments $S_k = [b_{t_{k,0}}, b_{t_{k,1}}]$.
    - For each segment $S_k$, identify the set of primitives $\mathcal{I}_k$ that overlap with this interval. Note that a segment may contain 0, 1, or multiple overlapping primitives [S1, unofficial].

3.  **Transmittance Calculation:**
    For each segment $S_k$, calculate the transmittance from the segment start $b_{t_{k,0}}$ to a point $t$ within the segment ($b_{t_{k,0}} \le t \le b_{t_{k,1}}$).
    - Calculate the optical depth $\tau_i(\mathbf{x}_a, \mathbf{x}_b)$ for each primitive $i \in \mathcal{I}_k$ along the ray segment from parameter $a$ to $b$:
      $$
      \tau_i(\mathbf{x}_a, \mathbf{x}_b) = \sigma_i \int_{\max(a, t_{i,0})}^{\min(b, t_{i,1})} K_i(\mathbf{x}_{t'}) dt'
      $$
      where $t_{i,0}$ and $t_{i,1}$ are the ray parameters where the ray enters and exits the support of primitive $i$ (or its bounding volume if clipped).
    - The total transmittance for the segment is:
      $$
      T(\mathbf{x}_{b_{t_{k,0}}}, \mathbf{x}_t) = \exp \left( -\sum_{i \in \mathcal{I}_k} \tau_i(\mathbf{x}_{b_{t_{k,0}}}, \mathbf{x}_{\min(t, b_{t_{k,1}})}) \right)
      $$
    - **Closed-form Integration:**
      - For **Gaussian Kernels**, the integral $\int K_i(\mathbf{x}_{t'}) dt'$ can be computed analytically using the error function (erf) [S1, unofficial].
      - For **Epanechnikov Kernels**, which have finite support, the integral is also analytical (polynomial) within the support [S1, unofficial].

4.  **Free-Flight Distance Sampling:**
    Sample a scattering event distance $t$ along the ray proportional to the PDF $p(t) = \mu_t(\mathbf{x}_t)T(\mathbf{x}_0, \mathbf{x}_t)$.
    - **Single Primitive Segment:** If a segment contains only one primitive, use analytical inversion of the cumulative distribution function (CDF) derived from the transmittance formula.
    - **Multiple Overlapping Primitives:** If a segment contains multiple primitives, the CDF may not have a simple closed-form inverse. Use numerical root-finding (e.g., Newton-Raphson or bisection) to invert the CDF, or use uniform sampling approximation if precision allows [S1, unofficial].
    - If no scattering event is sampled within the segment, proceed to the next segment.

5.  **Radiance Accumulation (Segment-wise):**
    Integrate radiance front-to-back over all segments $k=1 \dots M$:
    $$
    L(\mathbf{x}_0, \omega) = \sum_{k=1}^{M} T^{k-1}(\mathbf{x}_0, \mathbf{x}_{b_{t_{k,0}}}) L_k(\mathbf{x}_{b_{t_{k,0}}}, \omega)
    $$
    where $T^{k-1}$ is the transmittance from the camera to the start of segment $k$, and $L_k$ is the radiance contributed by segment $S_k$.
    - $L_k$ includes:
      - **Emission:** Direct emission from primitives within the segment.
      - **Scattering:** Light scattered into the ray direction from other light paths.

6.  **Next-Event Estimation (NEE) for Multiple Scattering:**
    - When a scattering event is sampled at point $\mathbf{x}_s$ within a segment, use Next-Event Estimation to sample direct illumination from light sources.
    - Trace a shadow ray to the light source.
    - Compute the transmittance from $\mathbf{x}_s$ to the light using the same analytical segment-based transmittance logic.
    - Add the estimated radiance to the path contribution.
    - Recursively trace the scattered ray (if not terminated by Russian Roulette) to handle multiple scattering [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def evaluate_kernel_gaussian(x, center, cov_inv):
    """
    Evaluate Gaussian kernel K(x) = exp(-0.5 * (x-c)^T Sigma^-1 (x-c))
    Assumes 1D ray parameterization for simplicity in integration context.
    In practice, this integral is pre-computed analytically using erf.
    """
    diff = x - center
    exponent = -0.5 * diff * cov_inv * diff
    return math.exp(exponent)

def integrate_gaussian_kernel_analytic(t_start, t_end, center, cov_inv):
    """
    Analytical integral of Gaussian kernel along ray segment [t_start, t_end].
    Integral of exp(-a*t^2 + b*t + c) dt involves erf.
    Simplified placeholder for the closed-form solution using erf.
    """
    # This is a placeholder for the actual erf-based closed-form calculation.
    # The exact formula depends on the specific parameterization of the Gaussian
    # along the ray. For a 1D Gaussian along the ray:
    # Integral = sqrt(pi/a) * [erf(...) - erf(...)]
    # Here we assume a simplified constant approximation for structure demonstration.
    # In real implementation, use the derived erf expression from the paper.
    pass

def integrate_epanechnikov_kernel_analytic(t_start, t_end, center, radius):
    """
    Analytical integral of Epanechnikov kernel.
    K(t) = 3/4 * (1 - ((t-center)/radius)^2) for |t-center| <= radius, else 0.
    """
    # Clip integration bounds to kernel support
    t_min = max(t_start, center - radius)
    t_max = min(t_end, center + radius)
    
    if t_min >= t_max:
        return 0.0
    
    # Integral of (1 - ((t-c)/r)^2) dt from t_min to t_max
    # Let u = (t-c)/r, dt = r*du
    # Integral = r * Integral (1 - u^2) du = r * [u - u^3/3]
    
    u_min = (t_min - center) / radius
    u_max = (t_max - center) / radius
    
    val = (u_max - u_max**3/3) - (u_min - u_min**3/3)
    return (3.0 / 4.0) * radius * val

def compute_optical_depth(primitive, t_a, t_b):
    """
    Compute optical depth tau_i for primitive i along ray segment [t_a, t_b].
    """
    if primitive.kernel_type == 'gaussian':
        # Use analytical erf-based integration
        tau = integrate_gaussian_kernel_analytic(t_a, t_b, primitive.center, primitive.cov_inv)
    elif primitive.kernel_type == 'epanechnikov':
        tau = integrate_epanechnikov_kernel_analytic(t_a, t_b, primitive.center, primitive.radius)
    else:
        raise ValueError("Unknown kernel type")
    
    return primitive.sigma * tau

def trace_volumetric_primitives(ray_origin, ray_direction, primitives, lights):
    """
    Main VPPT rendering function.
    """
    # 1. Intersection and Segmentation
    # Find all intersecting primitives and their entry/exit points
    intersections = []
    for p in primitives:
        # Ray-primitive intersection test (e.g., ray-AABB or ray-ellipsoid)
        t_entry, t_exit = p.intersect(ray_origin, ray_direction)
        if t_entry is not None:
            intersections.append((t_entry, t_exit, p))
    
    # Sort by entry point
    intersections.sort(key=lambda x: x[0])
    
    # Build segments
    segments = []
    current_t = ray_origin.t_min # e.g., 0.0 or camera near plane
    prev_exit = None
    
    for t_entry, t_exit, p in intersections:
        if t_entry > current_t:
            # Gap between segments (empty space)
            if prev_exit is not None and prev_exit > current_t:
                segments.append((current_t, prev_exit, []))
            current_t = t_entry
        
        # Add primitive to current segment
        # Handle overlapping primitives by grouping them
        if segments and segments[-1][0] == t_entry:
            segments[-1] = (segments[-1][0], max(segments[-1][1], t_exit), segments[-1][2] + [p])
        else:
            segments.append((t_entry, t_exit, [p]))
            
        current_t = t_exit
        prev_exit = t_exit
        
    # Add final segment if needed
    if current_t < ray_origin.t_max:
        segments.append((current_t, ray_origin.t_max, []))

    # 2. Radiance Accumulation
    total_radiance = 0.0
    transmittance_so_far = 1.0
    
    for t_start, t_end, overlapping_primitives in segments:
        if t_start >= t_end:
            continue
            
        # Calculate transmittance from start of segment to end of segment
        # T_segment = exp(-sum(tau_i for i in overlapping_primitives))
        total_optical_depth = 0.0
        for p in overlapping_primitives:
            tau = compute_optical_depth(p, t_start, t_end)
            total_optical_depth += tau
            
        segment_transmittance = math.exp(-total_optical_depth)
        
        # Contribution from this segment
        # L_k = Emission + Scattering
        # For simplicity, assume direct emission and single scattering approximation here
        # In full VPPT, NEE is used for multiple scattering
        
        segment_radiance = 0.0
        
        # Add emission
        for p in overlapping_primitives:
            # Emission is integrated over the segment
            # Integral of Q_i * K_i dt
            if p.kernel_type == 'gaussian':
                emission_integral = integrate_gaussian_kernel_analytic(t_start, t_end, p.center, p.cov_inv)
            elif p.kernel_type == 'epanechnikov':
                emission_integral = integrate_epanechnikov_kernel_analytic(t_start, t_end, p.center, p.radius)
            
            segment_radiance += p.emission * emission_integral
            
        # Add scattering (simplified: single scattering from lights)
        for light in lights:
            # Sample light direction, check visibility
            # Compute transmittance from segment to light using same method
            # Add contribution
            pass
            
        # Accumulate
        total_radiance += transmittance_so_far * segment_radiance
        
        # Update transmittance for next segment
        transmittance_so_far *= segment_transmittance
        
        # Russian Roulette for path termination could be applied here
        # if transmittance_so_far < threshold:
        #     if random() < survival_prob:
        #         transmittance_so_far /= survival_prob
        #     else:
        #         break

    return total_radiance
```

**Parameters:**
- **Kernel Types:** Gaussian Kernel (infinite support, smooth, uses erf for transmittance) and Epanechnikov Kernel (finite support, sharper edges, more efficient acceleration structures) [S1, unofficial].
- **Optical Cross-Section ($\sigma_i$):** Controls the density/scattering strength of each primitive.
- **Emission ($Q_i$):** Color/intensity of light emitted by the primitive.
- **Phase Function ($f_{p,i}$):** Describes scattering directionality (e.g., Henyey-Greenstein).

**Edge Cases and Pitfalls:**
- **Gaussian Infinite Support:** Gaussian kernels have infinite support, requiring clipping to a finite radius to avoid evaluating all primitives. This clipping must be consistent with the acceleration structure bounds [S1, unofficial].
- **Overlapping Primitives:** Multiple overlapping primitives in a segment require summing optical depths. The transmittance calculation must correctly handle the sum of integrals, not the integral of sums, if kernels are not normalized to 1 in the same way.
- **Numerical Stability:** For very high optical depths, transmittance approaches 0. Use log-transmittance or clamp values to avoid underflow/overflow in exponentiation [G].
- **Degenerate Input:** Primitives with $\sigma_i = 0$ or zero volume should be ignored or handled as empty [G].

## Operating characteristics
- **Time Complexity:** Depends on the number of primitives $N$ and the number of intersections per ray. With BVH acceleration, intersection testing is $O(\log N)$ on average. Transmittance calculation is $O(K)$ per segment where $K$ is the number of overlapping primitives. NEE adds complexity proportional to the number of lights.
- **Memory Complexity:** $O(N)$ to store primitive parameters. Significantly lower than voxel grids for high-resolution scenes [S1, unofficial].
- **Measured Performance:** Significantly faster than voxel-based methods due to analytical solutions and compact representation. Epanechnikov kernels offer faster rendering than Gaussian kernels due to finite support and simpler integration [S1, unofficial].
- **Scaling:** Scales well with the number of primitives. Level-of-Detail (LoD) can be implemented by adjusting the number of primitives [S1, unofficial].
- **Strengths:** Noise-free single-scattering results, efficient memory usage, physically based, supports multiple scattering via NEE, avoids popping artifacts.
- **Limitations:** Requires careful handling of overlapping primitives. Gaussian kernels require clipping. Not as flexible as neural fields for complex, non-parametric shapes [S1, unofficial].

## Background
Originated from the paper "Don’t Splat your Gaussians: Volumetric Ray-Traced Primitives for Modeling and Rendering Scattering and Emissive Media" by Meta (2025) [S1, unofficial]. The method addresses the limitations of voxel grids (high memory, noise) and 3D Gaussian Splatting (lack of physical scattering, ordering artifacts) by introducing kernel-based volumetric primitives with closed-form integration. It is used in forward and inverse rendering, tomography, and radiance field optimization [S1, unofficial].

## Sources
[S1] Don’t Splat your Gaussians: Volumetric Ray-Traced Primitives for Modeling and Rendering Scattering and Emissive Media (2025), https://qiita.com/matumu20/items/67b33681dc97ccbcc849, fetched 2026-10-27 (unofficial)
