# Volume Illustration Tone Shading

Also called: Modified Gooch tone shading for volume models, Tone illumination model

## Overview
Volume Illustration Tone Shading is a non-photorealistic illumination model for volume rendering that combines gaseous illumination with directed shading based on the signed dot product between the volume sample gradient and the light vector [S1, unofficial]. Unlike standard Gooch shading, the illuminated object contribution is zero at orientations orthogonal to the light vector, matching diffuse illumination models [S1, unofficial]. It differs from [Vignette Shading](1830-vignette-shading.md) and [Blinn-Phong Toon Shading](4708-blinn-phong-toon-shading.md) by using continuous warm/cool color interpolation for tone and clamped dot products for directed illumination, rather than silhouette-based darkening or discrete toon bands [S1, unofficial].

## How it works
**Inputs:**
- Volume dataset $V$ with scalar values.
- Gradient field $\nabla f_n$ (normalized volume sample gradient).
- Light vectors $L_i$ (unit vectors, $i=1 \dots NL$).
- Illuminated object color $I_i$ (from transfer function or base color).
- Gaseous illumination color $I_G$ (from transfer function).
- Parameters: $k_{ta}, k_{td}, k_{ty}, k_{tb}$.

**Outputs:**
- Final color $c$ for each volume sample.

**Data Structures:**
- Colors are 3-component vectors $(R, G, B)$.
- $\nabla f_n$ is a normalized 3D vector.
- $L$ is a normalized 3D vector.

**Parameters:**
- $k_{ta}$: Controls gaseous illumination amount. Typical: $1.0$ [S1, unofficial].
- $k_{td}$: Controls directed illumination amount. Typical: $0.6$ [S1, unofficial].
- $k_{ty}$: Warm tone intensity. Typical: $0.3$ [S1, unofficial].
- $k_{tb}$: Cool tone intensity. Typical: $0.3$ [S1, unofficial].
- $NL$: Number of light sources. Note: More than a few lights likely results in confusing images [S1, unofficial].

**Formulas:**
1. Warm color: $c_w = (k_{ty}, k_{ty}, 0)$ [S1, unofficial].
2. Cool color: $c_c = (0, 0, k_{tb})$ [S1, unofficial].
3. Tone contribution per light:
   $$ I_t = \left(\frac{1.0 + \nabla f_n \cdot L}{2}\right) c_w + \left(1 - \frac{1.0 + \nabla f_n \cdot L}{2}\right) c_c $$
   [S1, unofficial]
4. Directed illumination contribution per light:
   $$ I_o = \begin{cases} k_{td} I_i (\nabla f_n \cdot L) & \text{if } \nabla f_n \cdot L > 0 \\ 0 & \text{if } \nabla f_n \cdot L \le 0 \end{cases} $$
   [S1, unofficial]
5. Final color:
   $$ c = k_{ta} I_G + \sum_{i=1}^{NL} (I_t + I_o) $$
   [S1, unofficial]

**Pitfalls:**
- Using more than a few light sources results in confusing images because humans are not used to interpreting complex illumination from many lights [S1, unofficial].
- The directed illumination $I_o$ is zero when the gradient is orthogonal to the light, unlike standard diffuse models which might use absolute values or different clamping [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def volume_illustration_tone_shading(gradient, light_vectors, I_i, I_G, k_ta=1.0, k_td=0.6, k_ty=0.3, k_tb=0.3):
    """
    Compute color for a volume sample using Volume Illustration Tone Shading.
    
    Args:
        gradient: Normalized 3D vector (nx, ny, nz).
        light_vectors: List of normalized 3D light direction vectors.
        I_i: Illuminated object color (3D vector).
        I_G: Gaseous illumination color (3D vector).
        k_ta: Gaseous illumination weight.
        k_td: Directed illumination weight.
        k_ty: Warm tone intensity.
        k_tb: Cool tone intensity.
        
    Returns:
        Final color (3D vector).
    """
    # Define tone colors
    c_w = (k_ty, k_ty, 0.0)
    c_c = (0.0, 0.0, k_tb)
    
    # Initialize result
    color = (0.0, 0.0, 0.0)
    
    # Add gaseous illumination
    color = vector_add(color, vector_scale(I_G, k_ta))
    
    # Sum contributions from all lights
    for L in light_vectors:
        # Compute dot product
        dot = vector_dot(gradient, L)
        
        # Compute tone contribution It
        # It = ((1.0 + dot) / 2) * cw + (1 - (1.0 + dot) / 2) * cc
        t1 = (1.0 + dot) / 2.0
        t2 = 1.0 - t1
        I_t = vector_add(vector_scale(c_w, t1), vector_scale(c_c, t2))
        
        # Compute directed illumination contribution Io
        if dot > 0:
            I_o = vector_scale(I_i, k_td * dot)
        else:
            I_o = (0.0, 0.0, 0.0)
            
        # Add to total
        color = vector_add(color, vector_add(I_t, I_o))
        
    return color

def vector_add(v1, v2):
    return (v1[0] + v2[0], v1[1] + v2[1], v1[2] + v2[2])

def vector_scale(v, s):
    return (v[0] * s, v[1] * s, v[2] * s)

def vector_dot(v1, v2):
    return v1[0]*v2[0] + v1[1]*v2[1] + v1[2]*v2[2]
```

## Subtopics

### [Gooch Shading](1670-gooch-shading.md)
A shading technique applied in the fragment shader using vertex normals sampled on the fly.

## Operating characteristics
- **Time Complexity:** Linear in the number of volume samples and the number of light sources $NL$. Each sample requires $NL$ dot products and vector operations.
- **Memory Complexity:** Low, primarily storing the volume data and gradients. No large auxiliary buffers required beyond standard volume rendering structures.
- **Performance:** Depends on hardware acceleration for volume ray casting or slicing. The per-sample computation is lightweight but involves multiple vector operations per light.
- **Strengths:** Enhances shape perception by providing continuous tone variation and directional cues. Preserves base tissue colors while adding illustrative depth. Effective for medical volumes where structure clarity is key [S1, unofficial].
- **Limitations:** Can become visually confusing with many light sources. Relies on accurate gradient estimation, which can be noisy in low-resolution or noisy data.
- **When to use:** When illustrative clarity and shape perception are prioritized over photorealism, especially in medical or scientific visualization [S1, unofficial].
- **When not to use:** When photorealistic rendering is required, or when the scene has many light sources that would create visual clutter.

## Background
Originated in the paper "Non-Photorealistic Rendering of Volume Models" [S1, unofficial]. It extends Gooch shading to volume data by using volume gradients instead of surface normals and modifying the directed illumination component to be zero at orthogonal orientations, aligning with diffuse illumination models [S1, unofficial]. It is used in medical visualization to enhance the perception of 3D structures like liver, kidney, and intestine [S1, unofficial].

## Sources
[S1] Non-Photorealistic Rendering of Volume Models, https://userpages.cs.umbc.edu/ebert/npr/volillus.pdf, fetched 2026-10-27 (unofficial)
