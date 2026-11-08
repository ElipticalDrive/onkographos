# Lighting Transfer Function with Gradient Aligned Sampling

Also called: Gradient Aligned Sampling, LTF

## Overview
Lighting Transfer Function with Gradient Aligned Sampling is a non-photorealistic volume rendering technique that enhances material boundaries by using two scalar samples taken along the normalized gradient direction as inputs to a 2D lighting transfer function, while the center sample is used for color and opacity [S1]. It sets itself apart from standard Phong shading by dynamically adjusting lighting coefficients based on the local data structure (homogeneous vs. boundary) rather than just surface orientation, and differs from [Lit Sphere Shading](785-lit-sphere-shading.md) by relying on scalar value pairs rather than geometric sphere intersections to determine shading properties [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Volume scalar field $V$, light direction $L$, view direction $V_{dir}$, sample position $P$, and gradient $\nabla V$ at $P$.
*   **Output:** Final color $C$ and opacity $\alpha$ for the sample.

**Data Structures:**
1.  **Color Transfer Function (ColorTF):** A 1D lookup table (texture) mapping a scalar value $S$ to a base color vector $C_{base}$ and opacity $\alpha$.
2.  **Lighting Transfer Function (LTF):** Three separate 2D lookup tables (textures) indexed by the pair of scalar values $(S_1, S_2)$.
    *   $LTF_{ka}(S_1, S_2)$: Ambient lighting coefficient.
    *   $LTF_{kd}(S_1, S_2)$: Diffuse lighting coefficient.
    *   $LTF_{ks}(S_1, S_2)$: Specular lighting coefficient.
    *   *Layout:* The textures are indexed by normalized scalar values. If the scalar domain is $[S_{min}, S_{max}]$, the indices $i, j$ for $S_1, S_2$ are calculated as:
        $$ i = \text{clamp}\left(\frac{S_1 - S_{min}}{S_{max} - S_{min}} \times (W_{LTF} - 1), 0, W_{LTF} - 1\right) $$
        $$ j = \text{clamp}\left(\frac{S_2 - S_{min}}{S_{max} - S_{min}} \times (H_{LTF} - 1), 0, H_{LTF} - 1\right) $$
        where $W_{LTF}$ and $H_{LTF}$ are the width and height of the LTF textures.

**Parameters:**
*   **Gradient Sample Distance ($d$):** The distance between the center sample and the two supplemental samples along the gradient. Typical value: 1 voxel [S1].
*   **Light Scaling Factor ($K_{light}$):** A scalar multiplier applied to all light intensities to accommodate high dynamic range lighting for translucent materials. Typical value: $>1$ if necessary [S1].

**Numbered Steps:**
1.  **Gradient Calculation:** Compute the gradient vector $\mathbf{G} = \nabla V(P)$ at the current sample position.
2.  **Normalization:** Calculate the normalized gradient direction $\mathbf{N} = \mathbf{G} / \|\mathbf{G}\|$. If $\|\mathbf{G}\| \approx 0$, set $\mathbf{N} = (0,0,0)$ and skip LTF lookup (use default coefficients) [G].
3.  **Gradient-Aligned Sampling:**
    *   Calculate the offset vector $\mathbf{O} = d \cdot \mathbf{N}$.
    *   Read the scalar value $S_1$ at position $P + \mathbf{O}$ (in the direction of $\mathbf{N}$).
    *   Read the scalar value $S_2$ at position $P - \mathbf{O}$ (in the opposite direction $-\mathbf{N}$).
    *   Read the center scalar value $S$ at position $P$.
4.  **Base Color Lookup:** Use $S$ to lookup the base color $C_{base}$ and opacity $\alpha$ from the 1D ColorTF.
5.  **Lighting Coefficient Lookup:** Use the pair $(S_1, S_2)$ to lookup the ambient coefficient $k_a$, diffuse coefficient $k_d$, and specular coefficient $k_s$ from the respective 2D LTF textures.
6.  **Lighting Computation:**
    *   Calculate the half-vector $\mathbf{H} = \frac{\mathbf{L} + \mathbf{V}_{dir}}{\|\mathbf{L} + \mathbf{V}_{dir}\|}$ (assuming normalized $\mathbf{L}$ and $\mathbf{V}_{dir}$) [G].
    *   Compute the specular reflection vector $\mathbf{R} = 2(\mathbf{N} \cdot \mathbf{L})\mathbf{N} - \mathbf{L}$ [G].
    *   Calculate the final color $C$ using the Phong model modified by the LTF coefficients:
        $$ C = C_{base} \left( k_a + k_d \max(\mathbf{N} \cdot \mathbf{L}, 0) \right) + k_s \max(\mathbf{N} \cdot \mathbf{R}, 0)^n $$
        where $n$ is the specular exponent (shininess) [S1].
    *   Apply the light scaling factor: $C = C \times K_{light}$ [S1].
7.  **Compositing:** Composite the resulting color $C$ and opacity $\alpha$ into the frame buffer. Note that compositing must be done in the fragment program (or via alpha modulation) because $C$ may exceed $\alpha$ in magnitude [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def lighting_transfer_function_sample(V, P, L, V_dir, d, K_light, ColorTF, LTF_ka, LTF_kd, LTF_ks, n):
    # 1. Gradient Calculation
    G = compute_gradient(V, P)
    
    # 2. Normalization
    norm_G = length(G)
    if norm_G < 1e-6:
        N = (0, 0, 0)
        k_a, k_d, k_s = 0.2, 0.5, 0.5 # Default coefficients [G]
    else:
        N = G / norm_G
        
        # 3. Gradient-Aligned Sampling
        O = N * d
        S1 = V.sample(P + O)
        S2 = V.sample(P - O)
        S = V.sample(P)
        
        # 5. Lighting Coefficient Lookup
        # Normalize S1, S2 to [0, 1] assuming V domain is [S_min, S_max]
        # Note: S_min and S_max must be known to the fragment program or passed as uniforms
        norm_S1 = clamp((S1 - S_min) / (S_max - S_min), 0.0, 1.0)
        norm_S2 = clamp((S2 - S_min) / (S_max - S_min), 0.0, 1.0)
        
        k_a = LTF_ka.lookup(norm_S1, norm_S2)
        k_d = LTF_kd.lookup(norm_S1, norm_S2)
        k_s = LTF_ks.lookup(norm_S1, norm_S2)
        
    # 4. Base Color Lookup
    C_base, alpha = ColorTF.lookup(S)
    
    # 6. Lighting Computation
    # Normalize light and view directions
    L_norm = normalize(L)
    V_norm = normalize(V_dir)
    
    # Diffuse term
    N_dot_L = dot(N, L_norm)
    diffuse = max(N_dot_L, 0.0)
    
    # Specular term
    R = 2.0 * N_dot_L * N - L_norm
    N_dot_R = dot(N, R)
    specular = max(N_dot_R, 0.0) ** n
    
    # Combine
    C = C_base * (k_a + k_d * diffuse) + k_s * specular
    
    # Apply light scaling
    C = C * K_light
    
    return C, alpha
```

**Edge Cases and Pitfalls:**
*   **Boundary Thickness Sensitivity:** The technique is sensitive to the thickness of material boundaries. If the sample pair distance $d$ is greater than the boundary thickness, the method reads from adjacent homogeneous regions, potentially misclassifying the boundary type [S1].
*   **High Dynamic Range:** Since surfaces rendered with this method do not have enhanced opacity, the alpha-multiplied RGB color can exceed the alpha value. The unmodulated color for a voxel must be larger than one. Hardware implementations must support this, and compositing must be handled in the fragment program rather than the framebuffer blending stage [S1].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample, but with a higher constant factor than standard volume rendering due to additional texture lookups (3 LTF lookups + 1 ColorTF lookup) and gradient computation.
*   **Memory Complexity:** Requires storage for three 2D LTF textures in addition to the standard 1D ColorTF and volume data.
*   **Strengths:** Effectively highlights material boundaries and transitions by adapting lighting based on local scalar values. It provides a more illustrative appearance than standard Phong shading.
*   **Limitations:** Sensitive to the choice of gradient sample distance $d$. Requires careful tuning of LTF textures. High dynamic range handling requires specific implementation care [S1].
*   **When to use:** When the goal is to clearly distinguish different materials in a volume through lighting cues rather than just opacity.
*   **When not to use:** When performance is critical and the overhead of LTF lookups is prohibitive, or when the volume has very thin boundaries that cannot be resolved by the chosen $d$.

## Background
The Lighting Transfer Function with Gradient Aligned Sampling method was introduced by [S1] to address the limitation that combined scalar value, gradient magnitude, and second derivative do not directly distinguish different types of material boundaries. The method was implemented in graphics hardware using view-aligned textured polygons and fragment programs [S1]. It builds on the principles of illustrative volume rendering, specifically aiming to enhance the perception of material interfaces through lighting rather than just color/opacity mapping.

## Sources
[S1] Lighting Transfer Functions Using Gradient Aligned Sampling, https://www.cs.ucdavis.edu/~ma/papers/lightingtf.pdf, fetched 2026-10-29
