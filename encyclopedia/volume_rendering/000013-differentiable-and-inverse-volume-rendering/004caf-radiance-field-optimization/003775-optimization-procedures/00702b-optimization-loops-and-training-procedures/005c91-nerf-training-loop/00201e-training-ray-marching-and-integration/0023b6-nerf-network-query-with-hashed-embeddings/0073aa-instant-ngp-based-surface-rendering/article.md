# Instant NGP-based Surface Rendering

Also called: Radiance Surface Rendering, Radiance Surface Rendering via Instant NGP

## Overview
Instant NGP-based Surface Rendering is a method that repurposes the Instant NGP ray-marching infrastructure for explicit surface reconstruction by treating the radiance field as an occupancy field and terminating rays at the first surface intersection [S1]. Unlike standard NeRF which integrates volumetric color along the entire ray, this approach returns the color of the first sample where the occupancy value exceeds a threshold, resulting in a 2.4× average speedup in frames per second (FPS) compared to the baseline on MipNeRF360 scenes [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Camera rays, a radiance field represented by an interpolated hash grid lookup combined with a lightweight Multi-Layer Perceptron (MLP) [S1].
*   **Output:** Pixel color and depth (implicitly defined by the surface hit) [S1].

**Data Structures:**
*   **Hash Grid:** Stores feature vectors for spatial locations. The specific bit-packing or grid resolution is not detailed in the sources, but it is the core lookup structure used by Instant NGP [S1].
*   **MLP:** A lightweight network that processes the interpolated features to predict occupancy and color [S1].

**Algorithm Steps:**
1.  **Ray Marching:** For each pixel, generate a ray and sample points along it. The field $(\sigma_p, c_p)$ is evaluated at each sample point $p$ using the hash grid lookup and MLP [S1].
2.  **Occupancy Check:** For each sample along the ray, retrieve the occupancy value $\sigma_p$ [S1].
3.  **Surface Hit Detection:** Check if the occupancy value exceeds the threshold $\tau = 0.5$ [S1].
4.  **Termination and Color Return:** If $\sigma_p > 0.5$, return the color $c_p$ of that sample and terminate the ray [S1]. If the threshold is not met within the ray bounds, the ray is considered to have missed the surface (background) [S1].

**Parameters:**
*   **Occupancy Threshold ($\tau$):** 0.5. Samples with occupancy $> 0.5$ are considered surface intersections [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_surface_ray(ray_origin, ray_direction, hash_grid, mlp, threshold=0.5):
    """
    Renders a single ray using Instant NGP-based surface rendering.
    
    Args:
        ray_origin: 3D vector, origin of the ray.
        ray_direction: 3D vector, direction of the ray.
        hash_grid: Data structure for spatial hashing lookups.
        mlp: Lightweight MLP for feature processing.
        threshold: Float, occupancy threshold for surface detection (default 0.5).
        
    Returns:
        color: 3D vector, color of the surface hit.
        depth: Float, distance to the surface hit.
    """
    # Standard ray marching setup for Instant NGP
    # The sources do not specify the exact number of samples or step size,
    # so we assume a standard fixed-step or adaptive step loop typical of NGP.
    
    t = 0.0
    step_size = 0.01 # [G] Typical step size for NGP; not explicitly given in sources.
    max_steps = 1000 # [G] Safety limit; not explicitly given in sources.
    
    for i in range(max_steps):
        # Current sample position
        pos = ray_origin + ray_direction * t
        
        # Evaluate occupancy and color using Instant NGP's hash grid and MLP
        # This involves querying the hash grid at pos and passing features through MLP
        occupancy, color = evaluate_ngp_field(pos, hash_grid, mlp)
        
        # Check for surface intersection
        if occupancy > threshold:
            return color, t
        
        # Advance along the ray
        t += step_size
        
    # If no surface is hit, return background color (e.g., black or white)
    # [G] Background handling is not detailed in sources, assuming standard black.
    return [0.0, 0.0, 0.0], t

def evaluate_ngp_field(pos, hash_grid, mlp):
    """
    Evaluates the Instant NGP field at a position.
    """
    # Query hash grid for multi-resolution features
    features = query_hash_grid(pos, hash_grid)
    
    # Process features through MLP
    # [G] Specific MLP architecture (layers, activations) is not detailed in sources.
    # Assuming a standard lightweight MLP as described in Instant NGP.
    occupancy, color = mlp(features)
    
    return occupancy, color
```

**Edge Cases and Pitfalls:**
*   **Precision:** The method relies on the occupancy value crossing 0.5. If the network output is noisy near the surface, jittering may occur.
*   **Boundaries:** If the ray does not intersect any surface within the sampled range, the method must handle the "miss" case, typically by returning a background color.
*   **Degenerate Input:** Simple scenes with enough observations reconstruct surfaces accurately without additional constraints, but complex scenes may require regularization (e.g., Laplacian refinement) to smooth small kinks [S1].

## Subtopics

### Meshed Isosurface Rasterization
An optimization that replaces ray-marching with the rasterization of a meshed isosurface extracted from the radiance field, boosting rendering performance while maintaining visual quality [S1]. The color network is used only for mesh shading [S1]. Performance can be further boosted by storing precomputed hash grid lookups alongside mesh vertices or by projecting the directional MLP dependence into spherical harmonics [S1]. This variant provides an additional 2× speedup over the ray-marching version [S1].

### Relaxed Radiance Surface Rendering
A variant that switches to volumetric rendering in hard regions where the surface representation is insufficient, improving visual quality [S1]. This approach encourages surface-like, sparse distributions, resulting in more empty space that the renderer can efficiently skip. Consequently, Instant NGP automatically spawns more rays at equal batch size, covering more reference pixels per batch [S1]. When restricted to match NeRF's ray count, the relaxed variant delivers results approximately equal to NeRF [S1].

## Operating characteristics
**Time and Memory Complexity:**
*   **Speed:** The method achieves a 2.4× average speedup in FPS compared to the baseline NeRF on MipNeRF360 scenes [S1]. The relaxed variant achieves a 2.0× speedup [S1]. Meshed Isosurface Rasterization provides an additional 2× speedup over the ray-marching version [S1].
*   **Training Time:** On the DTU dataset, the method achieves an average Chamfer distance of 0.80 with 1 minute of training time, compared to NeuS2's 0.68 with 5 minutes of training time [S1].

**Measured Performance:**
*   **MipNeRF360 Indoor:** Mean PSNR of 29.02 dB (standard) and 29.41 dB (relaxed), compared to NeRF's 29.19 dB [S1].
*   **MipNeRF360 Outdoor:** Mean PSNR of 22.41 dB (standard) and 22.62 dB (relaxed), compared to NeRF's 22.47 dB [S1].
*   **DTU Geometry:** Average Chamfer distance of 0.80 (1 min training) vs. NeuS2's 0.68 (5 min training) [S1].

**Strengths and Limitations:**
*   **Strengths:** Fast rendering due to early termination; efficient surface extraction at any point without intermediate volume representation [S1]. The relaxed variant can exceed NeRF's visual quality in hard regions [S1].
*   **Limitations:** The standard surface rendering may struggle in hard regions without the relaxed variant [S1]. Geometry quality (Chamfer distance) is lower than NeuS2, though training is significantly faster [S1].

**When to Use:**
*   Use when fast surface extraction and rendering are prioritized over high-fidelity geometry reconstruction.
*   Use when training time is a constraint (e.g., 1 minute vs. 5 minutes for NeuS2) [S1].

**When Not to Use:**
*   Use when high-fidelity geometry is required and training time is not a constraint (NeuS2 is better) [S1].
*   Use when volumetric effects (transparency, subsurface scattering) are essential, as this method is designed for surface rendering [S1].

## Background
The method is based on the paper "Radiance Surfaces: Optimizing Surface Representations with a 5D Radiance Field Loss" [S1]. It builds on the Instant NGP codebase, which uses interpolated hash grid lookups combined with a lightweight MLP to represent radiance fields [S1]. The approach was developed to leverage the efficiency of Instant NGP for surface rendering tasks, offering a trade-off between speed and geometric accuracy compared to traditional volumetric NeRF methods [S1].

## Sources
[S1] Radiance Surfaces: Optimizing Surface Representations with a 5D Radiance Field Loss, https://doi.org/10.1145/3721238.3730713, fetched 2026-10-26
