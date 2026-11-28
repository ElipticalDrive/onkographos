# Emission-Absorption Mapping

Also called: Color and opacity maps, Isosurface extraction via opacity map

## Overview
Emission-Absorption Mapping is a strategy for mapping scalar data field values $f(\mathbf{x})$ to model parameters (emission $q$ and absorption $\kappa$) in emission-absorption models, where no scattering is taken into account [S1, unofficial]. It solves the problem of translating raw volumetric data into visual properties (color and opacity) suitable for rendering. It differs from sibling methods like [Absorption Term Restriction](1521-absorption-term-restriction.md) or [Homogeneous Media](1813-homogeneous-media.md) by explicitly defining user-controllable or algorithmic functions that determine how data intensity maps to light emission and absorption, often incorporating geometric cues like gradients for shading and boundary emphasis.

## How it works
**Inputs:** Scalar data field $f(\mathbf{x})$ defined on a grid or point set; target isovalue $f_0$ (for isosurface extraction); thickness parameter $r$; light source positions and intensities.
**Outputs:** Opacity $\alpha(\mathbf{x})$ and Color $C(\mathbf{x})$ (RGB) for each voxel or sample point.

**Data Structures:**
*   **Scalar Field:** $f(\mathbf{x})$, typically stored as a 3D array of floats.
*   **Gradient Field:** $\nabla f(\mathbf{x})$, computed via finite differences (e.g., central differences).
*   **Color Map:** A function or lookup table mapping data values to RGB vectors.
*   **Opacity Map:** A function or lookup table mapping data values to scalar opacity $\alpha$.

**Numbered Steps:**

1.  **Compute Local Gradient:** Calculate the gradient vector $\nabla f(\mathbf{x})$ at each sample point. Normalize it to obtain the local normal $\mathbf{N} = \frac{\nabla f(\mathbf{x})}{|\nabla f(\mathbf{x})|}$ [S1, unofficial].
2.  **Determine Opacity:** Calculate opacity $\alpha$ based on the chosen mapping strategy (Standard, Isosurface, or Boundary Emphasis).
3.  **Determine Color:** Calculate color $C$ based on the chosen mapping strategy (Standard RGB, Upson-Keeler Diffuse, or HSV).
4.  **Render:** Pass $\alpha$ and $C$ to the volume rendering integrator (e.g., from [Emission-Absorption Model](927-emission-absorption-model.md)).

**Formulas:**

*   **Standard Mapping:**
    The user defines two maps:
    $$ \text{OpacityMap: } f(\mathbf{x}) \mapsto \alpha $$
    $$ \text{ColorMap: } f(\mathbf{x}) \mapsto C_i, \quad i = \text{r}, \text{g}, \text{b} $$
    Usually, the same opacity $\alpha$ is used for all RGB frequencies [S1, unofficial]. High color value corresponds to high emission coefficient $q$; high opacity corresponds to high absorption coefficient $\kappa$ [S1, unofficial].

*   **Levoy's Isosurface Extraction (Thickness $r$ around $f_0$):**
    $$ \alpha = \alpha_0 \begin{cases} 1 & \text{if } |\nabla f(\mathbf{x})| = 0 \text{ and } f(\mathbf{x}) = f_0 \\ 1 - \frac{1}{r} \frac{|f_0 - f(\mathbf{x})|}{|\nabla f(\mathbf{x})|} & \text{if } |\nabla f(\mathbf{x})| > 0 \text{ and } \frac{|f_0 - f(\mathbf{x})|}{|\nabla f(\mathbf{x})|} < r \\ 0 & \text{otherwise} \end{cases} $$
    The term $\frac{|f_0 - f(\mathbf{x})|}{|\nabla f(\mathbf{x})|}$ is a first-order Taylor expansion estimate for the distance $|\mathbf{x}_0 - \mathbf{x}|$ to a point $\mathbf{x}_0$ where $f(\mathbf{x}_0) = f_0$ [S1, unofficial].

*   **Boundary Emphasis:**
    For datasets with constant regions and discontinuous boundaries, scale the original opacities by a discrete approximation of the local gradient vector $|\nabla f(\mathbf{x})|$ to emphasize boundaries [S1, unofficial].

*   **Upson-Keeler Diffuse Reflection (Color):**
    $$ C = k_a I_a + k_d \sum_{\text{lights}} \mathbf{N} \cdot \mathbf{L}_j I_j $$
    Where:
    *   $\mathbf{N}$ is the normalized local gradient vector (local surface normal).
    *   $\mathbf{L}_j$ is the direction from the current position to the $j$-th light source.
    *   $k_a, k_d$ are ambient and diffuse coefficients derived from the data value via a colormap.
    *   $I_a$ is global ambient intensity.
    *   $I_j$ is the intensity of the $j$-th light source [S1, unofficial].

*   **Sabella's HSV Mapping:**
    *   **V (Value/Brightness):** Obtained from ray-integration (standard volume rendering output) [S1, unofficial].
    *   **H (Hue):** Determined from the peak data value encountered along the ray.
    *   **S (Saturation):** Used for depth cueing, derived from the distance to the peak value or the center of gravity along the ray [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_opacity_levoy(f_val, grad_norm, f0, r, alpha0):
    """
    Computes opacity using Levoy's isosurface extraction formula.
    f_val: scalar data value at current point
    grad_norm: magnitude of gradient |grad f(x)|
    f0: target isovalue
    r: thickness of transition region
    alpha0: maximum opacity
    """
    if grad_norm == 0:
        if abs(f_val - f0) < 1e-9: # Check equality for zero gradient
            return alpha0
        else:
            return 0.0
    
    dist_estimate = abs(f0 - f_val) / grad_norm
    if dist_estimate < r:
        return alpha0 * (1.0 - dist_estimate / r)
    else:
        return 0.0

def compute_color_upson_keeler(f_val, grad_vec, lights, ambient_intensity, colormap):
    """
    Computes color using Upson-Keeler diffuse reflection model.
    f_val: scalar data value
    grad_vec: gradient vector (dx, dy, dz)
    lights: list of dicts with 'direction' (normalized vector to light) and 'intensity'
    ambient_intensity: scalar I_a
    colormap: function mapping f_val to (k_a, k_d, base_color)
    """
    # Normalize gradient to get normal N
    norm = sqrt(grad_vec[0]**2 + grad_vec[1]**2 + grad_vec[2]**2)
    if norm == 0:
        N = (0, 0, 0)
    else:
        N = (grad_vec[0]/norm, grad_vec[1]/norm, grad_vec[2]/norm)
    
    # Get coefficients from colormap
    k_a, k_d, _ = colormap(f_val)
    
    # Ambient term
    color = (k_a * ambient_intensity, k_a * ambient_intensity, k_a * ambient_intensity)
    
    # Diffuse term
    diffuse_sum = (0, 0, 0)
    for light in lights:
        L = light['direction']
        I_j = light['intensity']
        # Dot product N . L
        dot = N[0]*L[0] + N[1]*L[1] + N[2]*L[2]
        # Clamp to 0 to avoid negative lighting (backfacing)
        if dot > 0:
            diffuse_sum = (
                diffuse_sum[0] + dot * I_j,
                diffuse_sum[1] + dot * I_j,
                diffuse_sum[2] + dot * I_j
            )
    
    color = (
        color[0] + k_d * diffuse_sum[0],
        color[1] + k_d * diffuse_sum[1],
        color[2] + k_d * diffuse_sum[2]
    )
    return color
```

**Parameters:**
*   **Frequencies:** The visible spectrum is approximated by finitely many frequencies. Meyer [29] suggests four frequencies for a balance of cost and accuracy, but the RGB model uses three frequencies ($\nu_i$ with $i = \text{red}, \text{green}, \text{blue}$) [S1, unofficial].
*   **Thickness $r$:** Controls the smoothness of the isosurface extraction. Not given in the sources.
*   **$\alpha_0$:** Maximum opacity. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Gradient Estimate:** The term $\frac{|f_0 - f(\mathbf{x})|}{|\nabla f(\mathbf{x})|}$ is an estimate for the distance to the isosurface. It fails if $|\nabla f(\mathbf{x})| = 0$ (handled by the first case in Levoy's formula) or if the gradient is noisy [S1, unofficial].
*   **Self-Shadowing:** In the Upson-Keeler model, light source intensity $I_j$ is not attenuated by material between the light and the current location. Self-shadowing effects are disregarded [S1, unofficial].
*   **Flat Appearance:** Simple color maps (88) tend to look flat and diffuse; gradient-based shading (Upson-Keeler) mitigates this [S1, unofficial].

## Subtopics

### Isosurface Extraction via Opacity Map
A specific application of emission-absorption mapping where opacity is concentrated around a specific data value $f_0$ to visualize a surface. It differs from standard thresholding by using gradient-based smoothing to maintain constant thickness. Choose when visualizing distinct structures in noisy data. [Link](927-emission-absorption-model.md)

### Shading-Based Color Mapping
Uses gradient vectors to simulate lighting (diffuse/specular) on volumetric data. It differs from symbolic color mapping by providing geometric cues. Choose when depth perception and surface-like appearance are desired. [Link](927-emission-absorption-model.md)

### HSV Symbolic Mapping
Maps data properties to Hue, Saturation, and Value rather than physical emission/absorption. It differs from RGB mapping by allowing independent control of brightness (from integration) and color (from data properties). Choose for symbolic visualization or depth cueing. [Link](927-emission-absorption-model.md)

## Operating characteristics
*   **Time Complexity:** The mapping step is $O(N)$ where $N$ is the number of voxels/samples. Computing gradients adds a constant factor.
*   **Memory Complexity:** $O(N)$ to store the scalar field and gradients.
*   **Strengths:** Flexible; allows user interaction via color/opacity maps; gradient-based shading improves depth perception; isosurface extraction handles noisy data better than hard thresholds.
*   **Limitations:** No physical scattering; self-shadowing is ignored in basic shading models; mapping is non-trivial and often requires manual tuning [S1, unofficial].
*   **When to use:** When visualizing scalar fields where emission and absorption are the dominant effects, and geometric structure needs to be emphasized via shading or isosurfaces.
*   **When not to use:** When accurate light transport with scattering is required (use [Particle Model](2005-particle-model.md) or [Point-Sampled Spectrum Model](3468-point-sampled-spectrum-model.md)).

## Background
The method originates from early volume rendering research, notably by Levoy [25] and Upson and Keeler [41], as cited in [S1, unofficial]. It addresses the challenge of making scalar data visually interpretable by linking data values to optical properties. The approach is foundational in medical imaging and scientific visualization where isosurfaces and shaded volumes are standard.

## Sources
[S1] Volume Rendering Mathematical Models and Algorithmic Aspects, https://core.ac.uk/works/16970577, fetched 2026-10-29 (unofficial)
