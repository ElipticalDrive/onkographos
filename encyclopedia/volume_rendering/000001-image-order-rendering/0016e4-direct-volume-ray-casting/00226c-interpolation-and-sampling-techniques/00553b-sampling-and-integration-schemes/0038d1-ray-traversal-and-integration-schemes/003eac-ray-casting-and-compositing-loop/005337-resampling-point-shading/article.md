# Resampling Point Shading

Also called: Direct Lighting on Resampling Points, Improved Resampling Point Shading, New Volume Rendering Method for Explosion Fields

## Overview
Resampling Point Shading is a volume rendering technique that improves shading accuracy by applying the lighting model directly to resampling points using interpolated normal vectors and opacity-weighted color interpolation, thereby avoiding Mach band artifacts and interface color confusion [S1, S2]. It sets itself apart from its sibling [Accumulative Rendering](14504-accumulative-rendering.md) and [Template-based Ray-Casting](14511-template-based-ray-casting.md) by calculating normals and lighting at the resampling point itself rather than interpolating pre-shaded vertex colors or lighting results, which preserves microstructure details and physical color consistency at media interfaces [S1, S2].

## How it works
The method modifies the standard ray casting compositing loop by introducing a specific shading calculation phase for each resampling point $S$ along the ray. It replaces the interpolation of shaded vertex colors (Gouraud-style shading) with the interpolation of geometric and material properties, followed by a single lighting evaluation per point.

### Inputs and Outputs
**Inputs:**
*   Volume dataset $F$ with dimensions $L \times M \times N$.
*   For each voxel $(l, m, n)$:
    *   Scalar value $F_{l,m,n}$.
    *   Material color $C_m(S)_{l,m,n}$ (output of transfer function).
    *   Opacity $\alpha(S)_{l,m,n}$ (output of transfer function).
*   Ray parameters: origin, direction, step size.
*   Lighting parameters: ambient/diffuse/specular coefficients ($k_a, k_d, k_s$), light direction $L(S)$, view direction $V(S)$, half-vector $H(S)$, specular exponent $n$.

**Outputs:**
*   Final pixel color $C_{out}$.

### Data Structures
The method operates on the existing voxel grid. No new complex data structures are required beyond standard arrays for the volume data and pre-computed gradients if desired (though gradients can be computed on-the-fly).

*   **Voxel Data:** 3D array of floats for scalar values.
*   **Transfer Function Output:** 3D arrays for opacity ($\alpha$) and RGB color ($C_m$).
*   **Gradient Vectors:** 3D array of 3-component vectors ($\nabla F$).

### Numbered Steps

1.  **Gradient Calculation:** Calculate the gradient vector $\nabla F_{l,m,n}$ for each voxel using central differences [S1].
    $$
    \nabla F_{l,m,n} = \begin{bmatrix} (F_{l+1,m,n} - F_{l-1,m,n})/2\Delta x \\ (F_{l,m+1,n} - F_{l,m-1,n})/2\Delta y \\ (F_{l,m,n+1} - F_{l,m,n-1})/2\Delta z \end{bmatrix}
    $$
    where $\Delta x, \Delta y, \Delta z$ are the voxel spacings [S1].

2.  **Ray Casting Loop:** For each pixel, cast a ray through the volume. For each resampling point $S$ along the ray:
    a.  **Identify Surrounding Voxels:** Determine the 8 voxel corners surrounding the resampling point $S$. Let these corners be indexed by $j=1 \dots 8$.
    b.  **Interpolate Normal Vector:** Calculate the normal vector $N(S)$ at the resampling point by normalizing the trilinearly interpolated gradient vectors of the 8 surrounding voxel points [S1, S2].
        $$
        N(S) = \text{normalize}\left( f_{\text{Trilinear}}(\text{normalize}(\nabla F_j)) \right)
        $$
        where $f_{\text{Trilinear}}$ is the trilinear interpolation function [S1].
    c.  **Interpolate Opacity and Material Color:** Calculate the opacity $\alpha(S)$ and the opacity-weighted material color $\tilde{C}_m(S)$ at the resampling point using trilinear interpolation [S1, S2].
        $$
        \alpha(S) = f_{\text{Trilinear}}(\alpha_j)
        $$
        $$
        \tilde{C}_m(S) = f_{\text{Trilinear}}(\alpha_j C_m(S)_j)
        $$
        where $\alpha_j$ is the opacity and $C_m(S)_j$ is the material color at vertex $j$ [S1]. Note: The sources specify that opacity-weighted color interpolation is used to avoid interface color confusion [S1].
    d.  **Lighting Calculation:** Perform a single lighting operation using the interpolated normal $N(S)$ and the interpolated material color $\tilde{C}_m(S)$ [S1, S2].
        $$
        \tilde{C}_S = (k_a + k_d (L(S) \cdot N(S))) \tilde{C}_m(S) + k_s (H(S) \cdot N(S))^n
        $$
        where $\tilde{C}_S$ is the shaded color at the resampling point [S1].
    e.  **Compositing:** Accumulate the shaded color $\tilde{C}_S$ and opacity $\alpha(S)$ into the final pixel color using the front-to-back compositing equation [S1]:
        $$
        C_{out} = \sum_{a=1}^{m} \tilde{C}(S_a) \alpha(S_a) \prod_{b=1}^{a-1} (1 - \alpha(S_b))
        $$
        where $\tilde{C}(S_a)$ is the shaded color at resampling point $S_a$ [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def resampling_point_shading(volume_data, transfer_function, ray_origin, ray_direction, 
                             voxel_size, light_dir, view_dir, ka, kd, ks, n_specular):
    """
    Performs volume rendering using Resampling Point Shading.
    
    Args:
        volume_data: 3D array of scalar values.
        transfer_function: Function mapping scalar value -> (color, opacity).
        ray_origin: 3D vector.
        ray_direction: 3D vector.
        voxel_size: Scalar spacing (assumed uniform dx=dy=dz).
        light_dir: 3D vector (normalized).
        view_dir: 3D vector (normalized).
        ka, kd, ks: Lighting coefficients.
        n_specular: Specular exponent.
        
    Returns:
        final_color: 3D vector (RGB).
    """
    
    # Precompute gradients for all voxels
    # Dimensions L, M, N
    L, M, N = volume_data.shape
    gradients = np.zeros((L, M, N, 3))
    
    for l in range(1, L-1):
        for m in range(1, M-1):
            for n in range(1, N-1):
                # Central differences
                dx = (volume_data[l+1, m, n] - volume_data[l-1, m, n]) / (2 * voxel_size)
                dy = (volume_data[l, m+1, n] - volume_data[l, m-1, n]) / (2 * voxel_size)
                dz = (volume_data[l, m, n+1] - volume_data[l, m, n-1]) / (2 * voxel_size)
                gradients[l, m, n] = [dx, dy, dz]
                
    # Normalize gradients
    for l in range(L):
        for m in range(M):
            for n in range(N):
                grad = gradients[l, m, n]
                norm = np.linalg.norm(grad)
                if norm > 1e-8:
                    gradients[l, m, n] /= norm
                else:
                    gradients[l, m, n] = [0, 0, 0] # Fallback for zero gradient

    # Ray casting parameters
    num_samples = 100 # Example, depends on ray length and step size
    step_size = voxel_size # Example
    
    accumulated_color = [0.0, 0.0, 0.0]
    accumulated_opacity = 0.0
    
    # Traverse ray
    current_pos = ray_origin
    for i in range(num_samples):
        # Check if current_pos is within volume bounds
        if not is_inside_volume(current_pos, L, M, N, voxel_size):
            break
            
        # Get voxel coordinates
        l, m, n = get_voxel_indices(current_pos, voxel_size)
        
        # Identify 8 surrounding voxels for trilinear interpolation
        # Indices for corners of the voxel containing current_pos
        # Assuming current_pos is inside voxel (l,m,n)
        # Corners are (l,m,n) to (l+1,m+1,n+1)
        corner_indices = []
        for il in range(l, l+2):
            for im in range(m, m+2):
                for in_ in range(n, n+2):
                    corner_indices.append((il, im, in_))
        
        # Trilinear interpolation weights based on fractional part of current_pos
        # This is a simplified representation; actual implementation requires 
        # calculating weights w_l, w_m, w_n based on distance to corner
        weights = calculate_trilinear_weights(current_pos, voxel_size, l, m, n)
        
        # 1. Interpolate Normal Vector
        interpolated_grad = [0.0, 0.0, 0.0]
        for idx, (il, im, in_) in enumerate(corner_indices):
            # Get gradient at corner, handle boundary conditions if necessary
            grad = gradients[il, im, in_]
            # Weight by trilinear weight
            interpolated_grad += grad * weights[idx]
            
        # Normalize interpolated gradient to get Normal
        norm_val = np.linalg.norm(interpolated_grad)
        if norm_val > 1e-8:
            N_S = interpolated_grad / norm_val
        else:
            N_S = [0, 0, 0] # Default normal if gradient is zero
            
        # 2. Interpolate Opacity and Weighted Material Color
        alpha_sum = 0.0
        weighted_color_sum = [0.0, 0.0, 0.0]
        
        for idx, (il, im, in_) in enumerate(corner_indices):
            # Get scalar value at corner
            val = volume_data[il, im, in_]
            # Apply transfer function
            color, alpha = transfer_function(val)
            
            # Store for interpolation
            # Note: We need to interpolate alpha and (alpha * color) separately
            # But the formula says: alpha(S) = f_trilinear(alpha_j)
            # and C_m_tilde(S) = f_trilinear(alpha_j * C_m_j)
            
            alpha_sum += alpha * weights[idx]
            weighted_color_sum[0] += (alpha * color[0]) * weights[idx]
            weighted_color_sum[1] += (alpha * color[1]) * weights[idx]
            weighted_color_sum[2] += (alpha * color[2]) * weights[idx]
            
        alpha_S = alpha_sum
        C_m_tilde_S = weighted_color_sum
        
        # 3. Lighting Calculation
        # Calculate half vector H = normalize(L + V)
        H_S = normalize(light_dir + view_dir)
        
        # Dot products
        L_dot_N = dot(light_dir, N_S)
        H_dot_N = dot(H_S, N_S)
        
        # Blinn-Phong model
        # I(S) = ka * Ia + kd * Id * (L.N) + ks * Is * (H.N)^n
        # Assuming Ia=Id=Is=1 for simplicity or incorporated into ka,kd,ks
        # The formula in source: C_S_tilde = (ka + kd(L.N)) * C_m_tilde + ks(H.N)^n
        
        # Note: The source formula multiplies the material color by the diffuse term
        # and adds specular term. It does not multiply specular by material color.
        
        diffuse_term = ka + kd * L_dot_N
        specular_term = ks * (H_dot_N ** n_specular)
        
        # Apply to material color
        # C_S_tilde = diffuse_term * C_m_tilde + specular_term
        # Note: specular_term is a scalar intensity, usually added to the color
        # If specular_term is meant to be white light, it's added directly.
        # If it's colored, it might need material color, but source formula implies scalar addition.
        
        shaded_color = [0.0, 0.0, 0.0]
        for c in range(3):
            shaded_color[c] = diffuse_term * C_m_tilde_S[c] + specular_term
            
        # 4. Compositing
        # C_out = sum( C(S_a) * alpha(S_a) * product(1 - alpha(S_b)) )
        # We maintain accumulated_opacity to compute the product term
        
        contribution = [0.0, 0.0, 0.0]
        for c in range(3):
            contribution[c] = shaded_color[c] * alpha_S
            
        # Weight by transmittance from previous steps
        transmittance = 1.0 - accumulated_opacity
        
        for c in range(3):
            accumulated_color[c] += contribution[c] * transmittance
            
        accumulated_opacity += alpha_S * transmittance
        
        # Advance ray
        current_pos += ray_direction * step_size
        
    return accumulated_color

def is_inside_volume(pos, L, M, N, voxel_size):
    # Check if pos is within the bounds of the volume grid
    # Implementation depends on coordinate system setup
    return True # Placeholder

def get_voxel_indices(pos, voxel_size):
    # Convert world position to voxel indices
    return int(pos[0]/voxel_size), int(pos[1]/voxel_size), int(pos[2]/voxel_size)

def calculate_trilinear_weights(pos, voxel_size, l, m, n):
    # Calculate 8 weights for trilinear interpolation
    # This is a standard implementation detail
    # Returns list of 8 floats summing to 1
    return [1/8] * 8 # Placeholder for actual weight calculation

def normalize(v):
    norm = np.linalg.norm(v)
    if norm == 0: return [0,0,0]
    return [x/norm for x in v]

def dot(v1, v2):
    return sum(a*b for a,b in zip(v1, v2))
```

### Parameters
*   **Lighting Coefficients:** $k_a, k_d, k_s$ (ambient, diffuse, specular reflection coefficients) and specular exponent $n$. Typical values depend on the desired material appearance; not explicitly given in sources [S1].
*   **Voxel Spacing:** $\Delta x, \Delta y, \Delta z$. Assumed uniform in examples [S1].
*   **Interpolation:** Trilinear interpolation is used for normals, opacity, and weighted color [S1].

### Edge Cases and Pitfalls
*   **Interface Color Confusion:** Interpolating color and opacity separately causes incorrect colors at interfaces (e.g., green coating at red/green interface) [S1, S2]. This is handled by using opacity-weighted color interpolation ($\tilde{C}_m(S) = f_{\text{Trilinear}}(\alpha_j C_m(S)_j)$) [S1].
*   **Mach Band Phenomenon:** Interpolating shaded vertex colors (Gouraud shading) causes Mach bands when normal vectors change drastically [S1]. Calculating normals at the resampling point avoids this [S1].
*   **Microstructure Loss:** Interpolating lighting results flattens curved surfaces and loses detail when viewing angles change [S2]. Calculating normals at the resampling point preserves concavity/convexity [S2].
*   **Jaggedness:** The method retains a slightly fuzzy contour line to avoid jaggedness and distortion [S2].
*   **Zero Gradient:** If the gradient vector is zero (e.g., inside a homogeneous region), normalization fails. A fallback normal (e.g., [0,0,0] or a default) should be used [G].

## Subtopics

### Direct Lighting on Resampling Points
Same method, emphasizing the direct application of lighting. [Resampling Point Shading](52-resampling-point-shading.md).

### Improved Resampling Point Shading
Same method, emphasizing the improvements over the "current method". [Resampling Point Shading](52-resampling-point-shading.md).

### New Volume Rendering Method for Explosion Fields
Same method, applied specifically to explosion field visualization. [Resampling Point Shading](52-resampling-point-shading.md).

## Operating characteristics
*   **Time Complexity:** The method reduces lighting calculations from 8 to 1 per resampling point, significantly improving efficiency compared to the "current method" which performs 8 lighting calculations [S2]. Interpolation operations are increased to 7 per resampling point (for color, opacity, and normal vector) [S2].
*   **Memory Complexity:** No additional significant memory requirements beyond the volume data and pre-computed gradients [S2].
*   **Measured Performance:**
    *   Rendering speed improvements: 31% to 39% across various datasets (Groups A-E) [S2].
    *   Specific times on PC i3-4010U CPU, 8 GB RAM, R5 M230 GPU:
        *   Group A (4.88Mb, 1.0e4 meshes): 14.21s (Current) vs 8.93s (New) [S2].
        *   Group D (413.11Mb, 1.7e7 meshes): 227.3s (Current) vs 144.00s (New) [S2].
*   **Image Quality:**
    *   Image entropy reduction: 14.15% (Group 3) and 5.68% (Group 4) [S2].
    *   Mean gradient increase: 12% (Group C), 43% (Group D), 2.6% (Group E) [S2].
    *   Image mean and standard deviation reduction for Group E: Max reductions of 28.38% and 34.88% at initial moment; average reductions of ~15% and ~5% during shockwave generation [S2].
*   **Strengths:**
    *   Avoids interface color confusion [S1, S2].
    *   Preserves microstructure details (concavity/convexity) under changing viewing angles [S2].
    *   Reduces computational cost without requiring GPU acceleration or parallel computing [S2].
    *   Produces physically consistent colors at interfaces [S2].
*   **Limitations:**
    *   Requires calculation of gradients for all voxels [S1].
    *   Performance gain is relative to the "current method" which is already inefficient; absolute performance depends on dataset size and hardware [S2].
*   **When to Use:** Suitable for volume rendering of explosion fields and other datasets where microstructure detail and accurate interface coloring are critical, and where computational resources are limited [S1, S2].
*   **When Not to Use:** If GPU acceleration is available and the "current method" is optimized, the relative gain may be less significant. If microstructure detail is not important, simpler methods may suffice.

## Background
The method was introduced in 2024 by authors publishing in *AIP Advances* and *Research Square* as a solution to the limitations of existing volume rendering methods for explosion field visualization [S1, S2]. The "current method" referred to in the sources involves interpolating shaded vertex colors, which leads to Mach band artifacts and interface color confusion [S1, S2]. The new method was developed to improve shading accuracy and rendering efficiency for explosion field data, which contains complex media interfaces and microstructures [S1, S2].

## Sources
[S1] A volume rendering visualization method oriented to explosion field features, https://doi.org/10.1063/5.0226283, fetched 2026-10-26
[S2] A Volume Rendering Visualization Method Oriented to Explosion Field ..., https://www.researchsquare.com/article/rs-4492344/v1, fetched 2026-10-29
