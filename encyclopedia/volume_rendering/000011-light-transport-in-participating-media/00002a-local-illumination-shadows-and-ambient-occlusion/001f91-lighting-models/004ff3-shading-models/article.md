# Shading models

Also called: Diffuse and Specular Shading for Volume Rendering

## Overview
Shading models for volume rendering apply local illumination calculations to enhance the perception of 3D shape and surface orientation within volumetric data [S1, unofficial]. Unlike global illumination techniques that simulate light transport through participating media, this method approximates surface-like appearance by treating scalar field gradients as pseudo-normals and applying standard reflection models (diffuse and specular) to the volume samples [S1, unofficial]. It sets itself apart from [Gradient-based Shading with Phase Functions](1020-gradient-based-shading-with-phase-functions.md) by using simplified local reflection rather than complex phase functions [S1, unofficial].

## How it works
The method evaluates shading contributions along a ray traversing the volume [S1, unofficial]. It determines whether a region of the ray should be shaded based on data values and gradient magnitudes at the entry and exit points of that region [S1, unofficial]. If shading is applied, it computes diffuse and specular components using averaged data values and gradients from the region boundaries [S1, unofficial].

### Inputs and Outputs
*   **Inputs**:
    *   Scalar volume data $V$.
    *   Ray segment defined by entry point $\mathbf{p}_{in}$ and exit point $\mathbf{p}_{out}$ [G].
    *   Shading parameters: diffuse amount $k_d$, specular amount $k_s$, specular shininess $\alpha$, shading opacity $\beta$, shading range $[v_{min}, v_{max}]$, and minimum gradient threshold $g_{min}$ [S1, unofficial].
    *   Light direction $\mathbf{l}$ (assumed uniform or local) [G].
    *   View direction $\mathbf{v}$ (assumed uniform or local) [G].
*   **Outputs**:
    *   Shading color contribution $C_{shade}$ (RGB) [G].

### Data Structures
The implementation requires access to the scalar field $V(\mathbf{x})$ and its gradient $\nabla V(\mathbf{x})$ [G].
*   **Scalar Value**: $v = V(\mathbf{x})$ [G].
*   **Gradient Vector**: $\mathbf{g} = \nabla V(\mathbf{x}) = [\frac{\partial V}{\partial x}, \frac{\partial V}{\partial y}, \frac{\partial V}{\partial z}]$ [G].
*   **Gradient Magnitude**: $|\mathbf{g}| = \sqrt{\mathbf{g} \cdot \mathbf{g}}$ [G].

The gradient is typically computed as the derivative of the interpolation function used to sample the volume (e.g., trilinear interpolation) [S1, unofficial].

### Steps
1.  **Evaluate Boundary Properties**:
    For the current ray segment (defined by entry $\mathbf{p}_{in}$ and exit $\mathbf{p}_{out}$):
    *   Compute scalar values: $v_{in} = V(\mathbf{p}_{in})$, $v_{out} = V(\mathbf{p}_{out})$ [G].
    *   Compute gradient vectors: $\mathbf{g}_{in} = \nabla V(\mathbf{p}_{in})$, $\mathbf{g}_{out} = \nabla V(\mathbf{p}_{out})$ [G].
    *   Compute gradient magnitudes: $g_{in} = |\mathbf{g}_{in}|$, $g_{out} = |\mathbf{g}_{out}|$ [G].

2.  **Determine Shading Eligibility**:
    Shading is applied if:
    *   ($v_{in} \in [v_{min}, v_{max}]$ OR $v_{out} \in [v_{min}, v_{max}]$) [S1, unofficial]
    *   AND ($g_{in} > g_{min}$ OR $g_{out} > g_{min}$) [S1, unofficial].
    If not eligible, return $C_{shade} = \mathbf{0}$ [G].

3.  **Compute Averaged Data Value**:
    Calculate the diffuse reflectivity value $v_{shade}$:
    $$ v_{shade} = \frac{v_{in} + v_{out}}{2} $$
    [S1, unofficial].

4.  **Compute Averaged Gradient**:
    Determine the shading normal $\mathbf{n}$:
    *   If both $g_{in} > g_{min}$ and $g_{out} > g_{min}$:
        $$ \mathbf{g}_{avg} = \frac{\mathbf{g}_{in} + \mathbf{g}_{out}}{2} $$
    *   Else if only $g_{in} > g_{min}$:
        $$ \mathbf{g}_{avg} = \mathbf{g}_{in} $$
    *   Else if only $g_{out} > g_{min}$:
        $$ \mathbf{g}_{avg} = \mathbf{g}_{out} $$
    *   Else (should not happen if step 2 passed, but for safety):
        $$ \mathbf{g}_{avg} = \mathbf{0} $$
    Normalize the gradient to get the pseudo-normal:
    $$ \mathbf{n} = \frac{\mathbf{g}_{avg}}{|\mathbf{g}_{avg}|} $$
    [S1, unofficial].

5.  **Calculate Diffuse Component**:
    *   Diffuse color $C_{diff}$ is the color from the transfer function evaluated at $v_{shade}$ [S1, unofficial].
    *   Compute diffuse factor: $L_d = \max(0, \mathbf{n} \cdot \mathbf{l})$ [G].
    *   Diffuse contribution: $C_{diff\_comp} = k_d \cdot L_d \cdot C_{diff}$ [G].

6.  **Calculate Specular Component**:
    *   Specular color is assumed to be white ($\mathbf{1}$) [S1, unofficial].
    *   Compute reflection vector $\mathbf{r}$:
        $$ \mathbf{r} = 2(\mathbf{n} \cdot \mathbf{l})\mathbf{n} - \mathbf{l} $$
    *   Compute specular factor: $L_s = \max(0, \mathbf{r} \cdot \mathbf{v})^\alpha$ [G].
    *   Specular contribution: $C_{spec\_comp} = k_s \cdot L_s$ [G].

7.  **Combine and Apply Opacity**:
    *   Total shading color: $C_{total} = C_{diff\_comp} + C_{spec\_comp}$ [G].
    *   Apply shading opacity $\beta$:
        $$ C_{shade} = \beta \cdot C_{total} $$
    [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def dot(a, b):
    return a[0]*b[0] + a[1]*b[1] + a[2]*b[2]

def length(v):
    return math.sqrt(dot(v, v))

def normalize(v):
    l = length(v)
    if l == 0.0: return [0.0, 0.0, 0.0]
    return [v[0]/l, v[1]/l, v[2]/l]

def compute_shading(p_in, p_out, volume_data, shading_params, light_dir, view_dir):
    """
    Computes shading for a ray segment in a volume.
    
    Args:
        p_in: Entry point of the ray segment.
        p_out: Exit point of the ray segment.
        volume_data: Object providing V(p) and grad_V(p).
        shading_params: Dict with keys:
            k_d: float (diffuse amount)
            k_s: float (specular amount)
            alpha: float (specular shininess)
            beta: float (shading opacity)
            v_min: float
            v_max: float
            g_min: float
        light_dir: Normalized vector.
        view_dir: Normalized vector.
        
    Returns:
        RGB color vector for shading contribution.
    """
    v_min = shading_params['v_min']
    v_max = shading_params['v_max']
    g_min = shading_params['g_min']
    k_d = shading_params['k_d']
    k_s = shading_params['k_s']
    alpha = shading_params['alpha']
    beta = shading_params['beta']

    v_in = volume_data.get_value(p_in)
    v_out = volume_data.get_value(p_out)
    
    g_in = volume_data.get_gradient(p_in)
    g_out = volume_data.get_gradient(p_out)
    
    g_in_mag = length(g_in)
    g_out_mag = length(g_out)
    
    in_range = v_min <= v_in <= v_max
    out_range = v_min <= v_out <= v_max
    in_thresh = g_in_mag > g_min
    out_thresh = g_out_mag > g_min
    
    if not ((in_range or out_range) and (in_thresh or out_thresh)):
        return [0.0, 0.0, 0.0]
    
    v_shade = (v_in + v_out) / 2.0
    
    if in_thresh and out_thresh:
        g_avg = [(g_in[i] + g_out[i])/2.0 for i in range(3)]
    elif in_thresh:
        g_avg = g_in
    elif out_thresh:
        g_avg = g_out
    else:
        return [0.0, 0.0, 0.0]
        
    n = normalize(g_avg)
    
    # Transfer function lookup at v_shade [G]
    c_diff = volume_data.get_transfer_function_color(v_shade)
    l_d = max(0.0, dot(n, light_dir))
    c_diff_comp = [k_d * l_d * c_diff[i] for i in range(3)]
    
    r = [2.0 * dot(n, light_dir) * n[i] - light_dir[i] for i in range(3)]
    l_s = max(0.0, dot(r, view_dir)) ** alpha
    c_spec_comp = [k_s * l_s for _ in range(3)]
    
    c_total = [c_diff_comp[i] + c_spec_comp[i] for i in range(3)]
    c_shade = [beta * c_total[i] for i in range(3)]
    
    return c_shade
```

### Parameters
*   **Diffuse Amount ($k_d$)**: Controls the intensity of the diffuse reflection [S1, unofficial].
*   **Specular Amount ($k_s$)**: Controls the intensity of the specular highlight [S1, unofficial].
*   **Specular Shininess ($\alpha$)**: Exponent for the specular term; higher values produce sharper highlights [S1, unofficial].
*   **Shading Opacity ($\beta$)**: Controls the transparency of the shading contribution, allowing it to blend with the underlying volume color [S1, unofficial].
*   **Shading Range ($[v_{min}, v_{max}]$)**: Scalar value range within which shading is considered [S1, unofficial].
*   **Minimum Gradient Threshold ($g_{min}$)**: Minimum gradient magnitude required to apply shading, filtering out flat regions [S1, unofficial].
*   **Diffuse Color**: Derived from the transfer function at the averaged data value $v_{shade}$ [S1, unofficial].
*   **Specular Color**: Fixed to white [S1, unofficial].

### Edge Cases and Pitfalls
*   **Gradient Discontinuity**: Using the derivative of the trilinear interpolation function for gradients can cause discontinuities at shared cell faces, potentially introducing artifacts [S1, unofficial]. Finite differences are an alternative but more expensive [S1, unofficial].
*   **Zero Gradient**: If the averaged gradient is zero (e.g., in a perfectly flat region where both gradients are zero or cancel out), normalization fails [G]. The logic in Step 4 handles cases where one gradient is valid, but if both are zero, the region is not shaded (Step 2) [S1, unofficial].
*   **Out of Range Gradients**: The logic explicitly handles cases where one boundary gradient is below the threshold and the other is above, using only the valid one [S1, unofficial].

## Subtopics

### [Frequency Domain Directional Shading](852-frequency-domain-directional-shading.md)
A technique for implementing directional diffuse reflection in volume rendering by performing shading computations entirely in the frequency domain. It solves the problem of applying shape cues without spatial differentiation overhead. Choose when computational efficiency in the frequency domain is preferred over spatial gradient calculation.

### [Simplified Phong Lighting Model](885-simplified-phong-lighting-model.md)
A local illumination technique that modulates volumetric samples with diffuse and specular components, treating the gradient at the first significant surface intersection as a pseudo-surface normal. It differs by typically omitting shadow calculations for performance. Choose for fast, surface-like highlights in ray-cast volumes.

### [Gradient-based Shading with Phase Functions](1020-gradient-based-shading-with-phase-functions.md)
Adds an illumination term to the emissive term to simulate single-scattering effects or computes shading for implicit iso-surfaces using phase functions like Henyey-Greenstein. Choose when angular scattering properties are important for the appearance of the volume.

### [Lambert Shading with Shadow Rays](1133-lambert-shading-with-shadow-rays.md)
Computes diffuse lighting using the Lambertian reflectance model and enhances depth perception by casting shadow rays. It treats the volume as an opaque isosurface. Choose when only diffuse reflection and accurate shadows are needed, without specular highlights.

### [Volume Shading Model](2991-volume-shading-model.md)
A local illumination technique that calculates diffuse and specular contributions based on scalar data values and their gradients, managing the interplay between material opacity and light transport modes (indirect vs. direct) within a single shading pass. Choose when a comprehensive local shading model that handles both indirect and direct lighting components is required.

### [Colored Stochastic Shadow Mapping for Volume Rendering](9816-colored-stochastic-shadow-mapping-for-volume-rendering.md)
A two-step rendering technique that generates colored, translucent shadows using stochastic sampling. It sets itself apart by producing softer, colored shadows with less memory than Deep Shadow Maps. Choose when accurate, colored, translucent shadows are needed for direct volume rendering.

### [Gradient-based shading](13956-gradient-based-shading.md)
A local illumination technique for direct volume rendering that approximates surface lighting by estimating normals from the scalar field gradient and applying a Phong-like reflection model. It is computationally efficient but limited to local surface properties. Choose for fast, crisp depth cues in non-translucent media.

### [Volumetric Shading](14079-volumetric-shading.md)
Captures shadow information and global occlusion effects by branching a secondary ray towards the light source at each primary ray sample location. It computes transmittance at runtime. Choose when global ambient occlusion and runtime shadow computation are preferred over pre-computed shadow maps.

### Fire Shading Model
A shading model that combines Lambertian diffuse lighting, ambient light, and rim lighting, modulated by a fire-colored palette driven by animated fBM noise [S2, unofficial]. It computes the surface normal, light direction, diffuse factor, view direction, and rim light factor, then combines them with an ambient constant and maps the result through a fire palette [S2, unofficial]. Choose when stylized, animated volumetric effects are required.

## Operating characteristics
*   **Time Complexity**: $O(1)$ per shading evaluation, assuming gradient computation is $O(1)$ (e.g., via trilinear derivative) [G].
*   **Memory Complexity**: $O(1)$ additional memory for shading calculations [G].
*   **Measured Performance**: Shading adds computational overhead to ray casting but is generally cheaper than global illumination techniques [G]. It helps considerably in bringing out features and establishing orientations [S1, unofficial].
*   **Scaling**: Performance scales linearly with the number of ray samples. Parallelization is effective as each ray's shading computation is independent [S1, unofficial].
*   **Strengths**: Enhances perception of 3D shape and surface detail; computationally efficient; easy to implement within existing ray casting frameworks [S1, unofficial].
*   **Limitations**: Approximates surface-like appearance; does not account for subsurface scattering or global light transport; gradient discontinuities may cause artifacts; limited to local illumination [S1, unofficial].
*   **When to Use**: When enhancing visual clarity of features in volumetric data is prioritized over physical accuracy of light transport [S1, unofficial].
*   **When Not to Use**: When accurate simulation of participating media light transport (e.g., subsurface scattering, global illumination) is required [G].

## Background
The method was implemented within a ray casting volume renderer to calculate diffuse and specular contributions based on data values and gradients [S1, unofficial]. It builds on the principle that shading helps considerably in bringing out features in the data and establishing orientations [S1, unofficial]. The approach uses the derivative of the trilinear function as the gradient of the data, leveraging the fact that the function is already being used to model data at interior nodes of the tree [S1, unofficial].

## Sources
[S1] [PDF] Hierarchically Accelerated Ray Casting for Volume Rendering with ..., https://tr.soe.ucsc.edu/sites/default/files/technical-reports/UCSC-CRL-95-31.pdf, fetched 2026-10-27 (unofficial)
[S2] OmerNYU/Kaboom_Diffusion, https://github.com/OmerNYU/Kaboom_Diffusion, fetched 2026-10-26 (unofficial)
