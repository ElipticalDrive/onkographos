# Particle Splatting for Density Field Construction

Also called: 3D Density Field Construction via Splatting, CPU splatting, wax density field

## Overview
This method constructs a volumetric density field on the CPU by rasterizing interpolated particles into a 3D grid, enabling subsequent ray-marching of complex surfaces like wax [S1, unofficial]. It differs from its parent [Splatting](23-splatting.md) by explicitly targeting CPU-side grid construction for density-based rendering rather than direct image-space compositing, and from its child [Image-Space Particle Splatting for Density Field Construction](10567-image-space-particle-splatting-for-density-field-constructio.md) by operating in object space with a fixed uniform grid, which avoids the resolution limits of image-space buffers but requires managing 3D memory allocation [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $N$ particles with positions $\mathbf{p}_i$, interpolated between fixed time steps, and associated scalar attributes (e.g., temperature).
*   **Output:** A 3D texture (density field) stored in GPU memory, containing density and coupled attributes per voxel.

**Data Structures:**
*   **Grid:** A 3D voxel grid aligned with the container boundaries.
    *   **Dimensions:** $48 \times 94 \times 48$ voxels [S1, unofficial].
    *   **Layout:** Two channels per voxel:
        1.  Density ($\rho$)
        2.  Temperature $\times$ Density ($T \cdot \rho$) [S1, unofficial].
    *   **Format:** `Data3DTexture` with RG channel layout. Internal format is `RG32F` (float32) or `RG16F` if 32-bit float filtering is unsupported [S1, unofficial].
*   **Occupancy Grid:** A $4^3$ block occupancy grid used to skip empty regions during ray-marching [S1, unofficial].

**Parameters:**
*   **Kernel:** Separable kernel defined as $h(u) = (1 - u^2)^3$ [S1, unofficial].
*   **Temporal Smoothing Time Constant:** $\tau = 0.04$ seconds [S1, unofficial].
*   **Normalization:** Density is normalized to approximately 1.0 at the core of the wax [S1, unofficial].

**Steps:**
1.  **Interpolation:** Interpolate particle positions between two fixed time steps to ensure temporal consistency [S1, unofficial].
2.  **Splatting:** For each particle $i$ at position $\mathbf{p}_i$:
    *   Determine the bounding box of the kernel footprint within the 3D grid.
    *   For each voxel $v$ within the footprint:
        *   Calculate the normalized distance $u$ from the voxel center to $\mathbf{p}_i$.
        *   Compute the kernel weight $w = (1 - u^2)^3$.
        *   Add $w$ to the density channel of voxel $v$.
        *   Add $w \cdot T_i$ to the temperature-density channel of voxel $v$, where $T_i$ is the particle's temperature [S1, unofficial].
3.  **Normalization:** Normalize the density values such that the core density is $\approx 1$ [S1, unofficial].
4.  **Temporal Smoothing:** Apply in-place temporal smoothing to the density field to reduce flickering:
    $$ \rho_{new} = \rho_{old} \cdot e^{-dt/\tau} $$
    where $dt$ is the frame time delta and $\tau = 0.04$ s [S1, unofficial].
5.  **Upload:** Upload the resulting 3D grid to the GPU as a `Data3DTexture` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def construct_density_field(particles, grid_width, grid_height, grid_depth, tau, dt):
    """
    Constructs a 3D density field by splatting particles into a grid.
    
    Args:
        particles: List of dicts with 'pos' (vec3), 'temp' (float).
        grid_width, grid_height, grid_depth: Integers defining grid size.
        tau: Temporal smoothing constant (0.04).
        dt: Time delta in seconds.
        
    Returns:
        grid: 3D array of shape (grid_width, grid_height, grid_depth, 2).
              Channel 0: Density.
              Channel 1: Temperature * Density.
    """
    # Initialize grid with zeros
    # Dimensions: W x H x D x 2
    grid = zeros((grid_width, grid_height, grid_depth, 2))
    
    # Kernel radius for (1 - u^2)^3 is 1.0 (where u < 1)
    kernel_radius = 1.0
    
    for particle in particles:
        p = particle['pos']
        temp = particle['temp']
        
        # Determine voxel bounds
        min_x = max(0, floor(p.x - kernel_radius))
        max_x = min(grid_width - 1, ceil(p.x + kernel_radius))
        min_y = max(0, floor(p.y - kernel_radius))
        max_y = min(grid_height - 1, ceil(p.y + kernel_radius))
        min_z = max(0, floor(p.z - kernel_radius))
        max_z = min(grid_depth - 1, ceil(p.z + kernel_radius))
        
        for x in range(min_x, max_x + 1):
            for y in range(min_y, max_y + 1):
                for z in range(min_z, max_z + 1):
                    # Calculate normalized distance
                    # Assuming grid coordinates are in world space aligned with particles
                    # If grid is unit-aligned, u = distance in voxel units
                    dx = x - p.x
                    dy = y - p.y
                    dz = z - p.z
                    u_sq = dx*dx + dy*dy + dz*dz
                    
                    if u_sq < 1.0:
                        u = sqrt(u_sq)
                        weight = (1.0 - u*u)**3
                        
                        # Accumulate density
                        grid[x, y, z, 0] += weight
                        # Accumulate temperature * density
                        grid[x, y, z, 1] += weight * temp
                        
    # Normalize density to ~1 at core (implementation detail: scale by max or target)
    # Note: Exact normalization factor depends on particle distribution; 
    # source states it is normalized to ~1 at core [S1, unofficial].
    # We assume a global scaling factor S is applied here.
    # S = 1.0 / max(grid[:,:,0]) # Example heuristic, exact S not specified
    
    # Temporal smoothing: keep = exp(-dt/tau)
    keep_factor = exp(-dt / tau)
    
    # Apply smoothing in-place
    # The source implies the previous frame's data is mixed with new splats.
    # Assuming 'grid' here represents the *new* splat contributions,
    # and we need to blend with existing 'prev_grid'.
    # However, the source says "Lissage temporel en place" (in-place temporal smoothing)
    # using the update rule. This usually implies:
    # grid_new = grid_old * keep + grid_splat * (1 - keep) ?
    # Or simply decaying the old values before adding?
    # The formula "keep = exp(-dt/tau)" suggests a decay factor.
    # Standard exponential smoothing: val = val * decay + new * (1-decay)
    # But the source only gives "keep = exp(-dt/tau)".
    # Let's assume the standard interpretation of temporal coherence in this context:
    # The existing density decays, and new splats are added.
    
    # Re-reading source: "Lissage temporel en place, keep = exp(-dt/tau)"
    # This likely means the *existing* grid values are multiplied by keep,
    # and then the new splats are added? Or is it a low-pass filter?
    # Given "wax density field", persistence is key.
    # Let's implement the decay of the *previous* frame's grid.
    
    # If 'grid' is the buffer holding the state:
    # grid *= keep
    # grid += splats
    
    # Since the pseudocode above computes 'grid' from scratch, 
    # we must assume 'grid' is the persistent buffer.
    
    # Correct logic based on "in-place":
    # 1. Decay existing grid
    # 2. Add new splats
    
    # Resetting grid for new frame accumulation? 
    # No, "in-place" implies modifying the existing texture.
    
    # Let's rewrite the loop to assume 'grid' is the persistent buffer.
    pass 

# Revised Pseudocode for clarity of "in-place" operation
def splat_and_smooth(particles, grid, tau, dt):
    """
    Updates the persistent density grid in-place.
    """
    keep_factor = exp(-dt / tau)
    
    # 1. Decay existing density
    grid[:, :, :, 0] *= keep_factor
    grid[:, :, :, 1] *= keep_factor
    
    # 2. Splat new particles
    kernel_radius = 1.0
    
    for particle in particles:
        p = particle['pos']
        temp = particle['temp']
        
        min_x = max(0, floor(p.x - kernel_radius))
        max_x = min(grid.shape[0] - 1, ceil(p.x + kernel_radius))
        min_y = max(0, floor(p.y - kernel_radius))
        max_y = min(grid.shape[1] - 1, ceil(p.y + kernel_radius))
        min_z = max(0, floor(p.z - kernel_radius))
        max_z = min(grid.shape[2] - 1, ceil(p.z + kernel_radius))
        
        for x in range(min_x, max_x + 1):
            for y in range(min_y, max_y + 1):
                for z in range(min_z, max_z + 1):
                    dx = x - p.x
                    dy = y - p.y
                    dz = z - p.z
                    u_sq = dx*dx + dy*dy + dz*dz
                    
                    if u_sq < 1.0:
                        u = sqrt(u_sq)
                        weight = (1.0 - u*u)**3
                        
                        grid[x, y, z, 0] += weight
                        grid[x, y, z, 1] += weight * temp

    # 3. Normalize (approximate)
    # Source says "normalized to approx 1 at core". 
    # This might be a post-process step or handled by particle intensity.
    # We note it as a step.
    normalize_core_density(grid)
```

**Edge Cases and Pitfalls:**
*   **Precision:** Use `RG32F` if possible; fall back to `RG16F` only if 32-bit float filtering is unsupported, which may introduce quantization artifacts [S1, unofficial].
*   **Boundaries:** Clamp voxel indices to grid dimensions to prevent out-of-bounds access [S1, unofficial].
*   **Temporal Jitter:** The smoothing constant $\tau=0.04$ s helps mitigate flickering, but high frame-rate jitter (±30%) can still cause surface thresholding artifacts if not handled by the ray-marcher's depth testing [S1, unofficial].

## Subtopics

### [Image-Space Particle Splatting for Density Field Construction](10567-image-space-particle-splatting-for-density-field-constructio.md)
Evaluates density directly in the image buffer using multiple render targets, avoiding 3D grid memory limits and preserving sharp features, but at the cost of image-space resolution constraints.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot R^3)$ per frame, where $N$ is the number of particles and $R$ is the kernel radius in voxels. The source reports median CPU splatting times of 1.1–1.25 ms and p95 of 1.3–1.9 ms on a Xeon 2.1 GHz processor [S1, unofficial].
*   **Memory Complexity:** $O(W \cdot H \cdot D)$ for the 3D grid. For a $48 \times 94 \times 48$ grid with 2 channels of float32, this is approximately $48 \cdot 94 \cdot 48 \cdot 8$ bytes $\approx 1.7$ MB [S1, unofficial].
*   **Strengths:** Low CPU overhead; simple implementation; effective for wax-like surfaces with temporal coherence [S1, unofficial].
*   **Limitations:** Fixed resolution grid limits detail; memory usage scales with volume size; not suitable for highly dynamic, sparse, or large-scale scenes without hierarchical structures [S1, unofficial].
*   **When to Use:** Small to medium-sized volumetric simulations (e.g., wax lamps) where CPU-side density construction is preferred for simplicity and compatibility with software rendering pipelines [S1, unofficial].

## Background
This method was implemented in the "lava-lamp" project by Nathan Drll, featuring a wax surface constructed via CPU splatting and ray-marched in a glass container [S1, unofficial]. It represents a practical application of object-order splatting for density field construction in real-time software rendering contexts.

## Sources
[S1] Étape 4 : surface de cire raymarchée et liquide volumique, https://github.com/nathanddrl/lava-lamp/pull/4, fetched 2026-10-27 (unofficial)
