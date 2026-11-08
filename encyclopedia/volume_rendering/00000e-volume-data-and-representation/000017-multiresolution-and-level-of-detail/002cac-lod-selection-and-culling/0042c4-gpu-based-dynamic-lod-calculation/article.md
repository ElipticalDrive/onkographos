# GPU-Based Dynamic LOD Calculation

Also called: Compute shader LOD, Ruggedness-aware LOD

## Overview
This method calculates Level-of-Detail (LOD) levels for visible tiles entirely on the GPU using compute shaders, replacing traditional CPU-side computation based on viewpoint distance and screen-space error [S1]. It distinguishes itself by integrating terrain ruggedness into the LOD decision logic and performing the calculation in parallel for thousands of tiles, thereby reducing CPU overhead [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Tile center position, viewpoint position, camera field of view ($\theta$), tile ground resolution ($r$), allowable screen-space error threshold ($\varepsilon$), normalized standard deviation of elevation values within the tile ($\sigma$), and terrain relief influence coefficient ($\alpha$) [S1].
*   **Outputs:** The calculated LOD level and the corresponding vertex buffer offset for each visible tile [S1].

**Data Structures:**
The method operates on a grid of tiles. Each tile requires precomputed or accessible metadata:
1.  **Tile Center:** 3D coordinates $(x, y, z)$ [S1].
2.  **Elevation Statistics:** The normalized standard deviation of elevation values ($\sigma$) for the tile's area [S1].
3.  **Geometry:** The tile's inscribed circle radius is required for the transition interpolation [S1].

**Parameters:**
*   $\theta$: Field of view.
*   $r$: Ground resolution of the tile (meters per pixel) [S1].
*   $\varepsilon$: Allowable screen-space error threshold. Typical value: 1–2 pixels [S1].
*   $\alpha$: Coefficient adjusting the influence weight of terrain relief. Determined by testing $\alpha \in [0.0, 1.0]$ on 30 terrain samples (plains, hills, mountains, high mountains) against original resolution benchmarks to minimize root-mean-square pixel error. Final value: $\alpha = 0.5$ [S1].
*   $\sigma$: Normalized standard deviation of elevation values within the tile, reflecting terrain ruggedness [S1].

**Steps:**
1.  **Frustum and Occlusion Culling:** Determine which tiles are visible. This may utilize parallelization strategies from WebGPU compute pipelines or big data processing techniques for large-scale terrain mapping [S1].
2.  **LOD Calculation:** For each visible tile, compute the LOD level using the following formula [S1]:
    $$ \text{LOD} = \max\left(0, \left\lfloor \log_2 \left( \frac{d \cdot \tan(\theta/2)}{r \cdot \varepsilon \cdot (1 + \alpha \cdot \sigma)} \right) \right\rfloor \right) $$
    Where:
    *   $d$ is the distance from the tile center to the viewpoint [S1].
    *   The term $(1 + \alpha \cdot \sigma)$ compensates for visual importance differences between flat and rugged terrains, as rugged terrain requires higher resolution to represent details [S1].
3.  **Output Generation:** Output the calculated LOD level and the vertex buffer offset for the tile [S1].
4.  **Vertex Shader Interpolation (Mitigation):** To prevent visual "popping" artifacts caused by discrete LOD jumps, perform linear interpolation of LOD levels in the vertex shader [S1].
    *   Calculate $t$ for each vertex: $t = \frac{\min \text{ distance from vertex to tile boundary}}{\text{tile's inscribed circle radius}}$, where $t \in [0, 1]$ [S1].
    *   Compute interpolated LOD: $\text{LOD}_{\text{interp}} = (1 - t) \cdot \text{LOD}_{\text{current}} + t \cdot \text{LOD}_{\text{neighbor}}$ [S1].
    *   Use $\text{LOD}_{\text{interp}}$ for vertex position sampling [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Compute Shader Stage
def compute_tile_lod(tile_center, viewpoint, fov, ground_resolution, error_threshold, ruggedness, relief_weight):
    """
    Calculates LOD for a single tile.
    tile_center: vec3
    viewpoint: vec3
    fov: float (radians)
    ground_resolution: float (meters per pixel)
    error_threshold: float (pixels)
    ruggedness: float (normalized std dev of elevation)
    relief_weight: float (alpha)
    """
    # Distance from tile center to viewpoint
    d = length(viewpoint - tile_center)
    
    # Calculate numerator: d * tan(theta/2)
    numerator = d * tan(fov / 2.0)
    
    # Calculate denominator: r * epsilon * (1 + alpha * sigma)
    denominator = ground_resolution * error_threshold * (1.0 + relief_weight * ruggedness)
    
    # Avoid division by zero
    if denominator == 0:
        return 0
        
    # Calculate log2 ratio
    log_ratio = log2(numerator / denominator)
    
    # Floor and clamp to 0
    lod = floor(log_ratio)
    if lod < 0:
        lod = 0
        
    return lod

def vertex_shader_interpolation(vertex_pos, tile_center, tile_inscribed_radius, lod_current, lod_neighbor):
    """
    Interpolates LOD for smooth transitions.
    vertex_pos: vec3
    tile_center: vec3
    tile_inscribed_radius: float
    lod_current: float
    lod_neighbor: float
    """
    # Distance from vertex to tile center
    dist_to_center = length(vertex_pos - tile_center)
    
    # Distance from vertex to tile boundary (approximated as radius - dist_to_center)
    # Note: The source defines t as min distance to boundary / inscribed radius.
    # Assuming a square tile, min distance to boundary is roughly inscribed_radius - dist_to_center
    # if vertex is inside. If outside, t=0 or handled by culling.
    # For implementation safety, we assume vertex is within the tile's influence area.
    
    dist_to_boundary = tile_inscribed_radius - dist_to_center
    if dist_to_boundary < 0:
        dist_to_boundary = 0
        
    t = dist_to_boundary / tile_inscribed_radius
    if t < 0.0:
        t = 0.0
    if t > 1.0:
        t = 1.0
        
    interpolated_lod = (1.0 - t) * lod_current + t * lod_neighbor
    return interpolated_lod
```

**Edge Cases and Pitfalls:**
*   **LOD Popping:** Discrete jumps in LOD levels cause visual artifacts. Mitigated by linear interpolation in the vertex shader [S1].
*   **Division by Zero:** If the denominator in the LOD formula is zero, the result is undefined. Guard against this [G].
*   **Negative LOD:** The formula uses `max(0, ...)` to ensure LOD does not drop below 0 [S1].

## Operating characteristics
*   **Time Complexity:** The LOD calculation is performed in parallel per tile on the GPU compute shader, making it efficient for large numbers of visible tiles (several thousand) [S1].
*   **Memory Complexity:** Requires storing tile metadata (center, elevation stats) and vertex buffers for different LOD levels [S1].
*   **Strengths:** Reduces CPU overhead by migrating LOD decisions to the GPU; accounts for terrain ruggedness for better visual quality [S1].
*   **Limitations:** Requires precomputed or accessible elevation statistics ($\sigma$) for each tile; interpolation adds complexity to the vertex shader [S1].
*   **When to Use:** Large-scale terrain rendering where CPU bottleneck is significant and visual quality needs to adapt to terrain complexity [S1].
*   **When Not to Use:** Scenes with uniform flat terrain where ruggedness adjustment is unnecessary, or when GPU compute capabilities are limited [G].

## Background
This method was introduced in "Efficient Parsing and Rendering of Large-Scale Terrain Data Based on WebGPU" (CACG 2026) [S1]. It addresses the limitation of CPU-side LOD computation, which becomes non-negligible when handling thousands of visible tiles [S1]. The approach draws on parallelization strategies from WebGPU compute pipelines and big data processing techniques for large-scale terrain mapping [S1].

## Sources
[S1] Efficient Parsing and Rendering of Large-Scale Terrain Data Based on WebGPU, https://doi.org/10.1145/3844212.3844228, fetched 2026-11-02
