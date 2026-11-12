# Hybrid Preshading with Postclassification

Also called: Hybrid Classification and Shading

## Overview
Hybrid Preshading with Postclassification is a real-time volume rendering optimization that decouples the classification step from the shading step to balance visual fidelity and performance [S1]. It solves the trade-off between the speed of preclassification/preshading (which often requires opacity-weighted interpolation to avoid artifacts) and the visual quality of postclassification/postshading (which is computationally expensive) [S1]. Unlike its siblings, it performs classification after interpolation (postclassification) to ensure accurate material assignment, but performs shading before interpolation (preshading) using a precomputed reflectance map to maintain high rendering speeds [S1].

## How it works
**Inputs:**
- Volume dataset $V$.
- Transfer function for classification (mapping density to color/opacity).
- Lighting model (e.g., Phong) and light positions.
- Camera parameters.

**Outputs:**
- Rendered image with volumetric shading.

**Data Structures:**
1. **Volume Grid:** Standard 3D grid storing scalar density values.
2. **Gradient Map:** Precomputed for each voxel. The gradient vector $\nabla V$ at each voxel is quantized to one of $N$ discrete space directions.
   - **Layout:** For each voxel $(i, j, k)$, store an integer index $d \in [0, N-1]$ representing the quantized gradient direction.
   - **Size:** $N = 8192$ quantized directions [S1].
3. **Preshaded Reflectance Map:** A lookup table storing the shaded color for each of the $N$ quantized directions.
   - **Layout:** Array of size $N$, where each entry contains the RGB color resulting from applying the lighting model to the base material color (derived from classification) for that specific direction.
   - **Generation:** Before rendering, for each of the $N$ directions, the algorithm computes the shading result and stores it in the map.

**Steps:**
1. **Precomputation (Gradient Quantization):**
   - Compute the gradient $\nabla V$ for every voxel in the volume.
   - Quantize each gradient vector to the nearest of the $N$ predefined space directions.
   - Store the index of this direction in the Gradient Map for each voxel.

2. **Precomputation (Preshaded Reflectance Map):**
   - For each of the $N$ quantized directions $d_i$:
     - Determine the base material color $C_{base}$ using the transfer function on the volume data (or a representative value).
     - Apply the lighting model (e.g., Phong illumination) using the direction $d_i$ as the surface normal or view-dependent vector to compute the shaded color $C_{shaded}$.
     - Store $C_{shaded}$ in the Preshaded Reflectance Map at index $i$.

3. **Rendering (Ray Marching):**
   - For each pixel, cast a ray through the volume.
   - Sample the volume along the ray.
   - **Interpolation:** Perform trilinear interpolation of the volume data (density) and the Gradient Map indices at the sampling location [S1]. This yields an interpolated density $\rho$ and an interpolated direction index $d_{interp}$ (which may be fractional or require nearest-neighbor selection for the map lookup, though the text implies shading the 8 vertices before interpolation, see below).
   - **Correction on Vertex Shading:** The source specifies shading the 8 vertices before color interpolation. This implies:
     - Identify the 8 voxels surrounding the current sampling point.
     - For each of the 8 vertices, retrieve its precomputed gradient direction index $d_v$.
     - Look up the shaded color $C_v$ from the Preshaded Reflectance Map using $d_v$.
     - Perform trilinear interpolation of these 8 shaded colors $C_v$ and the opacities $\alpha_v$ at the sampling location.
   - **Classification:** Use the interpolated density $\rho$ to determine the final opacity and potentially refine the color if the transfer function is non-linear or depends on density in a way not captured by the preshaded map (though the text emphasizes using the preshaded map for shading, classification is post-interpolation). The text states "postclassification but only preshading". This means the opacity and base color selection happen after interpolation, but the *lighting* is applied to the vertices before interpolation.
   - **Compositing:** Composite the resulting color and opacity along the ray using standard volume integration (e.g., front-to-back).

**Parameters:**
- Number of quantized space directions: $N = 8192$ [S1].
- Sampling rate $\Delta s = 0.5$ (distance between consecutive interpolations), allowing 2-4 samplings per cell [S1].
- Image resolution: $512 \times 512$ pixels [S1].
- Pixel subdivisions and precomputed rays: Fixed to $16^2$ [S1].

**Edge Cases and Pitfalls:**
- **Artifacts:** Mueller et al. reported artifacts with preshading and postclassification [S1]. This implementation claims no visible artifacts are present, suggesting careful handling of the interpolation and quantization [S1].
- **Quantization Error:** Using only 8192 directions may introduce directional aliasing or lighting inaccuracies, especially for smooth gradients.
- **Memory:** Storing the gradient map and preshaded reflectance map requires additional memory, though the preshaded map is small ($8192 \times 3$ floats).

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Precomputation Phase
def precompute_gradient_map(volume):
    """
    Computes the gradient for each voxel and quantizes it to one of N directions.
    Returns a 3D grid of indices.
    """
    N = 8192
    # Predefined set of N unit vectors representing space directions
    directions = get_quantized_directions(N) 
    
    gradient_map = zeros_like(volume, dtype=int)
    
    for each voxel (i, j, k):
        grad = compute_gradient(volume, i, j, k)
        # Find nearest direction index
        idx = argmin([dot(grad, d) for d in directions])
        gradient_map[i, j, k] = idx
    return gradient_map

def precompute_reflectance_map(volume, transfer_function, lighting_model):
    """
    Generates a preshaded reflectance map for each quantized direction.
    """
    N = 8192
    directions = get_quantized_directions(N)
    reflectance_map = zeros((N, 3)) # RGB colors
    
    # Determine a representative base color for the volume
    # In a more complex scenario, this might depend on density, 
    # but the text implies a single map is used for shading vertices.
    # We assume the transfer function maps density to color.
    # For preshading, we might use a mean density or the transfer function's 
    # output for the specific material being rendered.
    base_color = compute_base_color_from_transfer_function(transfer_function)
    
    for i in range(N):
        direction = directions[i]
        # Apply lighting model to the base color using the direction
        shaded_color = lighting_model(base_color, direction)
        reflectance_map[i] = shaded_color
        
    return reflectance_map

# Rendering Phase
def render_volume(volume, gradient_map, reflectance_map, camera, transfer_function):
    """
    Renders the volume using hybrid preshading and postclassification.
    """
    N = 8192
    image = zeros((height, width, 3))
    
    for each pixel (x, y):
        ray = camera.get_ray(x, y)
        color = (0, 0, 0)
        opacity = 0
        
        # Ray marching
        t = ray.origin
        while t < ray.end:
            # Get the 8 surrounding voxels for trilinear interpolation
            voxel_coords = get_voxel_coords(t)
            
            # Retrieve precomputed gradient indices for the 8 vertices
            indices = [gradient_map[v] for v in voxel_coords]
            
            # Retrieve preshaded colors for the 8 vertices
            preshaded_colors = [reflectance_map[idx] for idx in indices]
            
            # Retrieve densities and opacities for the 8 vertices
            densities = [volume[v] for v in voxel_coords]
            opacities = [transfer_function.get_opacity(d) for d in densities]
            base_colors = [transfer_function.get_color(d) for d in densities]
            
            # Trilinear interpolation of the preshaded colors
            # Note: The text says "shade the 8 vertices before color interpolation"
            # This implies we interpolate the already-shaded colors.
            interp_color = trilinear_interpolate(preshaded_colors, t)
            
            # Trilinear interpolation of opacity
            interp_opacity = trilinear_interpolate(opacities, t)
            
            # Postclassification: Determine final contribution based on interpolated density
            # The preshaded color is used, but opacity is determined post-interpolation
            # Standard front-to-back compositing
            alpha = interp_opacity
            if alpha > 0:
                color = color + interp_color * alpha * (1 - opacity)
                opacity = opacity + alpha * (1 - opacity)
            
            t += sampling_rate
            
        image[x, y] = color
        
    return image
```

## Subtopics

### [Fast Two-Pass Shading](2311-fast-two-pass-shading.md)
Fast Two-Pass Shading is a rendering optimization that avoids redundant per-fragment computations—specifically volume texture access, classification, and lighting—by sharing intermediate results between pixels covered by the same splat. It improves rendering performance by 5%–10% on typical hardware, distinguishing itself from sibling methods like Congradient Shading by explicitly decoupling the classification/illumination step from the geometric filtering step via a P-buffer.

### [Congradient Shading](2341-congradient-shading.md)
Congradient shading is a real-time volumetric shading acceleration technique that defines the surface orientation of a voxel as one of a finite set of local gradients, enabling a fast table-driven mechanism. It solves the performance bottleneck of per-voxel normal calculation and lighting integration by integrating shading into the projection pipeline, generating "congradient values" (concatenated color and gradient data) that serve as direct indices to a precomputed illumination look-up table. Unlike sibling methods that may perform shading in a separate post-processing stage or require complex normal quantization, congradient shading operates as a pipeline with the viewing process, accruing no additional time for shading because the congradient calculation time is much less than the projection time.

### [Fake Phong Illumination via Normal Quantization](5846-fake-phong-illumination-via-normal-quantization.md)
Fake Phong Illumination via Normal Quantization is a real-time approximation technique for volume rendering that replaces per-voxel Phong lighting calculations with a pre-computed color lookup table indexed by quantized surface normals. It solves the performance bottleneck of interactive volumetric rendering by moving the computational cost of the Phong equation to a pre-processing stage, allowing rendering to proceed via simple texture mapping. Unlike real-time Phong illumination which calculates lighting dynamically for every sample, this method sacrifices image fidelity to achieve speed, specifically introducing directional aliasing artifacts due to the discrete nature of the normal vector quantization.

### [Hybrid Gradient Computation](7505-hybrid-gradient-computation.md)
A technique to compute gradient directions for lighting that adaptively align with post-classified density variations while using pre-computed gradient directions for sampling, avoiding the need for two-sided lighting. It differs from standard pre-classified gradients by flipping normals to ensure single-sided lighting correctly illuminates low-scalar-value regions (like dentin) without lighting back-facing surfaces.

## Operating characteristics
- **Time Complexity:** The per-voxel shading cost is reduced to a table lookup during rendering, as shading is precomputed. The dominant cost is the trilinear interpolation of the preshaded colors and opacities. The precomputation phase is $O(N \cdot V)$ where $N$ is the number of directions and $V$ is the number of voxels, but this is done once.
- **Memory Complexity:** Requires storage for the volume data, the gradient map (1 integer per voxel), and the preshaded reflectance map ($N$ colors). With $N=8192$, the reflectance map is small.
- **Performance:** Benchmarks were performed on platforms using MMX, SSE, or 3DNow! SIMD extensions for memory copying and trilinear interpolation [S1]. The method allows high-quality rendering on $512 \times 512$ images with up to 4x zoom without visible noise [S1].
- **Strengths:** Combines the speed of preshading with the visual quality of postclassification. Avoids the artifacts associated with preclassification/preshading (which requires opacity-weighted interpolation) and the slowness of postclassification/postshading [S1].
- **Limitations:** Relies on quantization of gradient directions, which may introduce directional artifacts. The claim of "no artifacts" contradicts Mueller et al., suggesting potential sensitivity to specific volume characteristics or transfer functions [S1].

## Background
The method was introduced in "A new object-order ray-casting algorithm" [S1]. It addresses the limitations of traditional preclassification/preshading and postclassification/postshading approaches by proposing a hybrid strategy. The authors argue against the interpretation by Mueller et al. that preshading and postclassification inherently produce bad artifacts, claiming their implementation shows no visible artifacts [S1]. The technique is part of a broader effort to optimize real-time volume rendering using SIMD instructions and precomputed data structures.

## Sources
[S1] A new object-order ray-casting algorithm, https://doi.org/10.5555/602099.602130, fetched 2026-10-26
