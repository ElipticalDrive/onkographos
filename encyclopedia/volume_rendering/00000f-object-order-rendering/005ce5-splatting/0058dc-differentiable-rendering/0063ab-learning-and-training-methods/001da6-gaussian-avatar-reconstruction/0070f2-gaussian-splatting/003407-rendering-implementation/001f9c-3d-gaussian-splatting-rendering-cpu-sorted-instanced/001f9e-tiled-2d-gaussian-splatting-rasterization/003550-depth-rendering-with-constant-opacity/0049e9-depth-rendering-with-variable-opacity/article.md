# Depth Rendering with Variable Opacity

Also called: Gaussian Splatting Depth Output, Nearest Depth Rendering, Per-pixel Depth Buffer

## Overview
This method extends Gaussian Splatting rasterization to produce a per-pixel depth buffer (NDC depth, $+\infty$ where no surface is visible) while respecting the variable opacity of the Gaussians [S1, unofficial]. Unlike [Depth Rendering with Constant Opacity](7462-depth-rendering-with-constant-opacity.md), which forces uniform opacity to drive depth accumulation by geometric distance, this method computes depth based on the actual alpha-weighted contribution of the splats [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of 3D Gaussians (position, covariance, opacity $\alpha$, color), camera projection matrices, and a configuration flag `GsplatConfig::outputDepth` set to true [S1, unofficial].
*   **Outputs:** A color image (unchanged from standard rendering) and a per-pixel depth buffer storing NDC depth values [S1, unofficial].

**Data Structures:**
*   **Depth Buffer:** A texture or buffer of size $W \times H$ (image resolution) storing floating-point NDC depth values [S1, unofficial].
*   **Raster Backend Auxiliary:** An order-preserving unsigned integer encoding of depth, used for atomic operations in the fragment shader and decoded by a subsequent compute pass [S1, unofficial].

**Steps:**
1.  **Configuration:** Enable `GsplatConfig::outputDepth` to activate depth buffer writing [S1, unofficial]. Ensure the device feature `fragmentStoresAndAtomics` is enabled if using the raster backend, as it requires atomic operations [S1, unofficial].
2.  **Projection Correction:** Apply `cropProjection()` if rendering an off-centre view or crop [S1, unofficial]. This centers the Gaussian projection guard band on the view rather than the optical axis to prevent distortion of Gaussians elongated along the view direction [S1, unofficial].
3.  **Sorting:** Sort Gaussians by depth relative to the camera [S2, unofficial].
4.  **Rasterization and Compositing:**
    *   **Compute Backend:** Blend Gaussians front-to-back [S1, unofficial]. For each pixel, track the accumulated transmittance $T_n$ [S1, unofficial]. Record the depth $z_n$ of the Gaussian at which the transmittance first falls to 0.5 (median depth) [S1, unofficial].
    *   **Raster Backend:** Blend Gaussians back-to-front [S1, unofficial]. For each pixel, use `atomicMin` on an order-preserving uint encoding of depth to keep the nearest depth of a single splat that alone covers the pixel with an opacity of at least 0.5 [S1, unofficial]. After the render pass, a compute pass decodes this uint encoding into the final depth buffer [S1, unofficial].
5.  **Expected Depth Calculation:** If expected depth is required instead of median/nearest, compute the alpha-weighted average depth using the formula below [S2, unofficial].

**Formulas:**
The accumulated transparency $T_n$ for the $n$-th Gaussian at pixel $(x, y)$ is:
$$ T_n = \prod_{j=1}^{Q-1} (1 - \alpha_j) $$
where $Q$ is the number of Gaussians covering the pixel, and $\alpha_j$ is the opacity of the $j$-th Gaussian in depth-sorted order [S2, unofficial].

The **Expected Depth** $d_{x,y}^{exp}$ is:
$$ d_{x,y}^{exp} = \frac{\sum_{n=1}^N z_n \cdot \alpha_n \cdot T_n}{\sum_{n=1}^N \alpha_n \cdot T_n} $$
where $z_n$ is the Z-depth of the $n$-th Gaussian [S2, unofficial].

The **Accumulated Depth** $d_{x,y}^{acc}$ (numerator for expected depth) is:
$$ d_{x,y}^{acc} = \sum_{n=1}^N z_n \cdot \alpha_n \cdot T_n $$
[S2, unofficial]

**Parameters:**
*   `GsplatConfig::outputDepth`: Boolean flag to enable depth output [S1, unofficial].
*   Opacity threshold: 0.5 is used for median depth crossing (Compute) and nearest splat coverage (Raster) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Off-centre Projection Distortion:** If the guard band is not centered on the view (e.g., in cropped renders), Gaussians away from the optical axis are incorrectly clamped, distorting their footprints [S1, unofficial]. Use `cropProjection()` to fix this [S1, unofficial].
*   **Backend Disagreement:** The Compute backend (median depth) and Raster backend (nearest splat with $\alpha \ge 0.5$) use different definitions [S1, unofficial]. They agree on opaque surfaces with a median difference of 0.4% on the lantern scene, but may differ in semi-transparent regions [S1, unofficial].
*   **Hardware Support:** The Raster backend requires `fragmentStoresAndAtomics` support on the GPU [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Standard linear depth encoding/decoding [G]
def linear_encode_depth(z, min_z, max_z):
    return int(((z - min_z) / (max_z - min_z)) * 0xFFFFFFFF)

def linear_decode_depth(encoded_z, min_z, max_z):
    return (encoded_z / 0xFFFFFFFF) * (max_z - min_z) + min_z

# Compute Backend Implementation (Front-to-Back)
def compute_depth_buffer(gaussians, camera, width, height, min_z, max_z):
    depth_buffer = [float('inf')] * (width * height)
    sorted_gaussians = sorted(gaussians, key=lambda g: g.depth_at(camera))
    
    for y in range(height):
        for x in range(width):
            pixel_idx = y * width + x
            transmittance = 1.0
            median_depth = float('inf')
            
            for gaussian in sorted_gaussians:
                # Standard rasterization coverage check [G]
                if not gaussian.intersects_pixel(x, y):
                    continue
                    
                alpha = gaussian.opacity_at(x, y)
                z = gaussian.depth_at(x, y)
                
                # Update transmittance before checking crossing
                transmittance *= (1 - alpha)
                
                # Check for median depth crossing (transmittance <= 0.5)
                if transmittance <= 0.5 and median_depth == float('inf'):
                    median_depth = z
                    
            depth_buffer[pixel_idx] = median_depth
    return depth_buffer

# Raster Backend Implementation (Back-to-Front, Pseudocode for Fragment Shader Logic)
def raster_depth_buffer(gaussians, camera, width, height, min_z, max_z):
    # Requires atomicMin support [G]
    depth_uint_buffer = [0xFFFFFFFF] * (width * height)
    sorted_gaussians = sorted(gaussians, key=lambda g: g.depth_at(camera), reverse=True)
    
    for gaussian in sorted_gaussians:
        # Iterate over pixels covered by this Gaussian [G]
        for x, y in gaussian.covered_pixels():
            pixel_idx = y * width + x
            alpha = gaussian.opacity_at(x, y)
            z = gaussian.depth_at(x, y)
            
            # Encode depth into order-preserving uint
            encoded_z = linear_encode_depth(z, min_z, max_z)
            
            # Atomic minimum to keep nearest depth
            # Only update if this splat covers the pixel with alpha >= 0.5
            if alpha >= 0.5:
                if encoded_z < depth_uint_buffer[pixel_idx]:
                    depth_uint_buffer[pixel_idx] = encoded_z
                    
    # Decode uint buffer in a separate compute pass
    depth_buffer = [linear_decode_depth(z, min_z, max_z) for z in depth_uint_buffer]
    return depth_buffer
```

## Operating characteristics
*   **Time Complexity:** Same as standard Gaussian Splatting rasterization, $O(N)$ for sorting and $O(P \cdot G)$ for rasterization where $P$ is pixels and $G$ is Gaussians per pixel [S1, unofficial][S2, unofficial]. The additional depth calculation is negligible per pixel [S2, unofficial].
*   **Memory Complexity:** Adds one $W \times H$ depth buffer [S1, unofficial]. The Raster backend uses an order-preserving unsigned integer encoding for atomic operations, decoded by a compute pass after rendering [S1, unofficial].
*   **Performance:** The Compute backend may be slower on some hardware due to lack of parallel atomic optimizations compared to the Raster backend, but offers more consistent depth definitions (median) [S1, unofficial]. The Raster backend requires `fragmentStoresAndAtomics` support [S1, unofficial].
*   **Strengths:** Enables compositing of other geometry into the splat scene [S1, unofficial]. Handles variable opacity naturally [S2, unofficial].
*   **Limitations:** Two different depth definitions (median vs. nearest) lead to slight discrepancies between backends in semi-transparent regions [S1, unofficial]. Off-centre projections require special handling to avoid distortion [S1, unofficial].

## Background
This feature was introduced in the `klartraum` project by fortmeier and others in October 2026 to enable depth output for both Gaussian splatting backends (Compute and Raster) [S1, unofficial]. It addresses the need to composite other geometry into a splat scene, which was previously not possible as the splats would always render over other geometry [S1, unofficial]. The implementation also fixed a bug related to off-centre projection guard bands [S1, unofficial]. The expected depth formula is derived from the `gsplat` library documentation [S2, unofficial].

## Sources
[S1] feat: depth output for the Gaussian splatting backends, https://github.com/fortmeier/klartraum/pull/46, fetched 2026-10-27 (unofficial)
[S2] gsplat: An Open-Source Library for Gaussian Splatting, https://doi.org/10.48550/arXiv.2409.06765, fetched 2026-10-29 (unofficial)
