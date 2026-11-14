# Resampling Point Shading

Also called: Improved Resampling Point Shading

## Overview
Resampling Point Shading is a method for direct volume ray casting that improves shading accuracy by applying the lighting model directly to resampling points, rather than interpolating shaded colors from voxel vertices. It solves the problems of Mach band artifacts and interface color confusion inherent in previous approaches that interpolated color and opacity separately or used voxel-reflected light interpolation (Gouraud shading) [S1]. It sets itself apart from siblings like [Wavelet-Assisted Volume Ray Casting](10893-wavelet-assisted-volume-ray-casting.md) by focusing on geometric normal estimation and opacity-weighted color interpolation within the standard ray casting loop, rather than data compression or accumulation strategies.

## How it works
**Inputs and Outputs:**
*   **Input:** Volume data field $F$ (scalar values at voxel vertices), material color values $C_m$ at vertices, opacity values $\alpha$ at vertices, light source parameters (position, intensity), and camera parameters.
*   **Output:** Final pixel color $C_{out}$.

**Data Structures:**
*   **Volume Grid:** A 3D array of voxels. Each voxel vertex $(l, m, n)$ stores a scalar value $F_{l,m,n}$, a material color $C_m(S)_{l,m,n}$, and an opacity $\alpha_{l,m,n}$.
*   **Gradient Vector:** $\nabla F_{l,m,n}$, a 3D vector computed per voxel.
*   **Normal Vector:** $N(S)$, a 3D unit vector at the resampling point.
*   **Material Color:** $\tilde{C}_m(S)$, the opacity-weighted material color at the resampling point.
*   **Shaded Color:** $\tilde{C}_S$, the final lit color at the resampling point.

**Numbered Steps:**

1.  **Gradient Calculation:** For each voxel vertex $(l, m, n)$, calculate the gradient vector $\nabla F_{l,m,n}$ using central differences [S1].
    $$
    \nabla F_{l,m,n} = \begin{bmatrix} (F_{l+1,m,n} - F_{l-1,m,n}) / 2\Delta x \\ (F_{l,m+1,n} - F_{l,m-1,n}) / 2\Delta y \\ (F_{l,m,n+1} - F_{l,m,n-1}) / 2\Delta z \end{bmatrix}
    $$
    where $\Delta x, \Delta y, \Delta z$ are the voxel dimensions [S1].

2.  **Normal Vector Interpolation:** At a resampling point $S$ located within a voxel, identify the eight surrounding voxel vertices. Let $F(S)_j$ be the gradient vector $\nabla F$ at the $j$-th vertex ($j=1 \dots 8$). Normalize each vertex gradient, then trilinearly interpolate the normalized vectors to obtain the normal $N(S)$ at $S$ [S1]:
    $$
    N(S) = \text{normalize}\left( f_{\text{Trilinear}}(\text{normalize}(F(S)_j)) \right)
    $$
    where $f_{\text{Trilinear}}$ denotes trilinear interpolation [S1].

3.  **Opacity-Weighted Material Color Interpolation:** To avoid interface color confusion, do not interpolate color and opacity separately. Instead, compute the opacity-weighted material color $\tilde{C}_m(S)$ at $S$ by trilinearly interpolating the product of opacity $\alpha_j$ and material color $C_m(S)_j$ at the eight surrounding vertices [S1]:
    $$
    \tilde{C}_m(S) = f_{\text{Trilinear}}(\alpha_j C_m(S)_j)
    $$
    where $\alpha_j$ is the opacity and $C_m(S)_j$ is the material color at vertex $j$ [S1]. The material color $C_m(S)_j$ itself is obtained by applying the color transfer function to the scalar data value at vertex $j$ [S1].

4.  **Lighting Model Application:** Calculate the shaded color $\tilde{C}_S$ at the resampling point using the Blinn-Phong lighting model [S1]. This involves the ambient, diffuse, and specular components [S1]:
    $$
    \tilde{C}_S = (k_a + k_d (L(S) \cdot N(S))) \tilde{C}_m(S) + k_s (H(S) \cdot N(S))^n
    $$
    where:
    *   $k_a, k_d, k_s$ are ambient, diffuse, and specular reflection coefficients [S1].
    *   $L(S)$ is the light direction vector at $S$ [S1].
    *   $H(S)$ is the half-vector between the view direction and light direction [S1].
    *   $n$ is the specular exponent [S1].
    *   $\tilde{C}_m(S)$ is the material color from Step 3 [S1].

5.  **Compositing:** Synthesize the final pixel color $C_{out}$ using the front-to-back compositing equation, substituting the newly calculated shaded color $\tilde{C}_S$ (denoted as $\tilde{C}(S_a)$ in the formula) for the resampling points $S_a$ along the ray [S1]:
    $$
    C_{out} = \sum_{a=1}^{m} \tilde{C}(S_a) \alpha(S_a) \prod_{b=1}^{a-1} (1 - \alpha(S_b))
    $$
    where $\alpha(S_a)$ is the opacity at resampling point $S_a$, interpolated via standard trilinear interpolation of vertex opacities [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def resampling_point_shading(volume_data, transfer_function, light_params, camera_params):
    """
    volume_data: 3D array of shape (Nx, Ny, Nz) containing scalar values.
    transfer_function: function mapping scalar value to (color, opacity).
    light_params: dict containing ka, kd, ks, n, light_direction, view_direction.
    camera_params: dict containing ray origins and directions.
    """
    Nx, Ny, Nz = volume_data.shape
    dx, dy, dz = 1.0, 1.0, 1.0 # Assume unit voxel size for simplicity
    
    # Precompute gradients for all voxels
    # Gradient at (l, m, n) uses neighbors. Boundary handling is not specified in sources [G].
    # We assume zero-padding or clamping for boundaries.
    gradients = np.zeros((Nx, Ny, Nz, 3))
    for l in range(1, Nx - 1):
        for m in range(1, Ny - 1):
            for n in range(1, Nz - 1):
                fx = (volume_data[l+1, m, n] - volume_data[l-1, m, n]) / (2 * dx)
                fy = (volume_data[l, m+1, n] - volume_data[l, m-1, n]) / (2 * dy)
                fz = (volume_data[l, m, n+1] - volume_data[l, m, n-1]) / (2 * dz)
                gradients[l, m, n] = [fx, fy, fz]
    
    # Normalize gradients
    normalized_gradients = np.zeros_like(gradients)
    for l in range(Nx):
        for m in range(Ny):
            for n in range(Nz):
                grad = gradients[l, m, n]
                norm = np.linalg.norm(grad)
                if norm > 0:
                    normalized_gradients[l, m, n] = grad / norm
                else:
                    normalized_gradients[l, m, n] = [0, 0, 0]

    # Precompute opacity-weighted material colors for vertices
    # First, get material color and opacity for each vertex
    vertex_colors = np.zeros((Nx, Ny, Nz, 3))
    vertex_opacities = np.zeros((Nx, Ny, Nz))
    
    for l in range(Nx):
        for m in range(Ny):
            for n in range(Nz):
                val = volume_data[l, m, n]
                color, opacity = transfer_function(val)
                vertex_colors[l, m, n] = color
                vertex_opacities[l, m, n] = opacity
                
    # Compute alpha * C for interpolation
    alpha_weighted_colors = vertex_opacities[:, :, :, np.newaxis] * vertex_colors

    # Ray casting loop
    for ray in rays:
        t = ray.origin
        alpha_acc = 0.0
        color_acc = [0.0, 0.0, 0.0]
        
        # Iterate along ray
        while t is within volume bounds:
            # Get voxel coordinates containing t
            l, m, n = get_voxel_coords(t)
            
            # Get fractional coordinates within voxel
            u, v, w = get_fractional_coords(t)
            
            # 1. Interpolate Normal N(S)
            # Gather normalized gradients from 8 corners
            g000 = normalized_gradients[l, m, n]
            g100 = normalized_gradients[l+1, m, n]
            g010 = normalized_gradients[l, m+1, n]
            g110 = normalized_gradients[l+1, m+1, n]
            g001 = normalized_gradients[l, m, n+1]
            g101 = normalized_gradients[l+1, m, n+1]
            g011 = normalized_gradients[l, m+1, n+1]
            g111 = normalized_gradients[l+1, m+1, n+1]
            
            # Trilinear interpolation of normalized gradients
            # f(u,v,w) = (1-u)(1-v)(1-w)g000 + u(1-v)(1-w)g100 + ...
            n_interp = (
                (1-u)*(1-v)*(1-w)*g000 +
                u*(1-v)*(1-w)*g100 +
                (1-u)*v*(1-w)*g010 +
                u*v*(1-w)*g110 +
                (1-u)*(1-v)*w*g001 +
                u*(1-v)*w*g101 +
                (1-u)*v*w*g011 +
                u*v*w*g111
            )
            N_S = normalize(n_interp)
            
            # 2. Interpolate Opacity-Weighted Material Color C_m_tilde
            # Gather alpha*C from 8 corners
            c000 = alpha_weighted_colors[l, m, n]
            c100 = alpha_weighted_colors[l+1, m, n]
            c010 = alpha_weighted_colors[l, m+1, n]
            c110 = alpha_weighted_colors[l+1, m+1, n]
            c001 = alpha_weighted_colors[l, m, n+1]
            c101 = alpha_weighted_colors[l+1, m, n+1]
            c011 = alpha_weighted_colors[l, m+1, n+1]
            c111 = alpha_weighted_colors[l+1, m+1, n+1]
            
            C_m_tilde = (
                (1-u)*(1-v)*(1-w)*c000 +
                u*(1-v)*(1-w)*c100 +
                (1-u)*v*(1-w)*c010 +
                u*v*(1-w)*c110 +
                (1-u)*(1-v)*w*c001 +
                u*(1-v)*w*c101 +
                (1-u)*v*w*c011 +
                u*v*w*c111
            )
            
            # 3. Interpolate Opacity alpha(S) for compositing
            a000 = vertex_opacities[l, m, n]
            a100 = vertex_opacities[l+1, m, n]
            a010 = vertex_opacities[l, m+1, n]
            a110 = vertex_opacities[l+1, m+1, n]
            a001 = vertex_opacities[l, m, n+1]
            a101 = vertex_opacities[l+1, m, n+1]
            a011 = vertex_opacities[l, m+1, n+1]
            a111 = vertex_opacities[l+1, m+1, n+1]
            
            alpha_S = (
                (1-u)*(1-v)*(1-w)*a000 +
                u*(1-v)*(1-w)*a100 +
                (1-u)*v*(1-w)*a010 +
                u*v*(1-w)*a110 +
                (1-u)*(1-v)*w*a001 +
                u*(1-v)*w*a101 +
                (1-u)*v*w*a011 +
                u*v*w*a111
            )
            
            # 4. Apply Lighting Model
            # L(S) and H(S) depend on ray direction and light position
            L_S = normalize(light_params['direction']) # Simplified
            H_S = normalize(L_S + camera_params['view_direction'])
            
            ka = light_params['ka']
            kd = light_params['kd']
            ks = light_params['ks']
            n_spec = light_params['n']
            
            diffuse_term = kd * dot(L_S, N_S)
            specular_term = ks * pow(dot(H_S, N_S), n_spec)
            
            # C_S_tilde = (ka + kd(L.N)) * C_m_tilde + ks(H.N)^n
            # Note: C_m_tilde is a vector. The term (ka + diffuse) is scalar.
            shaded_color = (ka + diffuse_term) * C_m_tilde + specular_term
            
            # 5. Compositing
            # C_out = sum( C_S * alpha * product(1-alpha_prev) )
            # This is done incrementally in the loop
            weight = alpha_S * (1 - alpha_acc)
            color_acc += shaded_color * weight
            alpha_acc += weight * (1 - alpha_acc) # Standard front-to-back alpha accumulation
            
            t += ray.step_size
            
        final_pixel_color = color_acc

    return final_pixel_color
```

**Parameters:**
*   $k_a, k_d, k_s$: Ambient, diffuse, specular coefficients. Typical values depend on the material model, but are not specified in the sources [S1].
*   $n$: Specular exponent. Not given in the sources [S1].
*   $\Delta x, \Delta y, \Delta z$: Voxel dimensions. Not given in the sources [S1].

**Edge Cases and Pitfalls:**
*   **Interface Color Confusion:** Interpolating color and opacity separately causes errors. This is handled by using opacity-weighted color interpolation ($\alpha_j C_m(S)_j$) [S1].
*   **Mach Band Phenomenon:** Voxel-reflected light interpolation (Gouraud shading) causes Mach bands when normal vectors change drastically. This method avoids this by calculating normals at the resampling point [S1].
*   **Specular Reflection Errors:** Previous methods could not accurately represent specular reflection at the resampling point. This method calculates the normal at the resampling point to correctly apply the Blinn-Phong model [S1].
*   **Boundary Conditions:** The gradient calculation uses central differences, which require neighbors. The sources do not specify how to handle boundaries (e.g., clamping, reflection, zero-padding) [G].

## Operating characteristics
*   **Time Complexity:** The method adds overhead to the standard ray casting loop. For each resampling point, it requires computing trilinear interpolation for 8 vectors (normals) and 8 vectors (opacity-weighted colors), plus the lighting calculation. This is more expensive than simple Gouraud shading but provides higher accuracy.
*   **Memory Complexity:** Requires storing precomputed gradients and normalized gradients for all voxels, in addition to the original volume data and transfer function outputs. This increases memory usage compared to basic ray casting.
*   **Strengths:** Improves shading accuracy, reduces Mach band artifacts, and prevents interface color confusion. It provides a more realistic 3D sense of reality, especially for specular highlights [S1].
*   **Limitations:** Higher computational cost due to normal vector calculation and opacity-weighted interpolation. Performance scaling is not explicitly measured in the sources [S1].
*   **When to use:** When high-precision visualization is required, particularly for data with sharp interfaces or where specular reflections are important (e.g., explosion fields) [S1].
*   **When not to use:** When computational resources are severely limited and visual fidelity is secondary.

## Background
The method was introduced in a 2024 paper by authors publishing in *AIP Advances* [S1]. It addresses limitations in previous volume rendering techniques for explosion field visualization, where standard resampling point coloring led to large computational errors and inaccurate specular reflections [S1]. The work builds on the Blinn-Phong lighting model [S1] and critiques the use of voxel-reflected light interpolation (Gouraud shading) which suffers from Mach band phenomena [S1].

## Sources
[S1] A volume rendering visualization method oriented to explosion field features, https://doi.org/10.1063/5.0226283, fetched 2026-10-26
