# Pre-Integration Lookup
Also called: 3D Pre-integration Table, Pre-integrated Transfer Functions
## Overview
Pre-integration lookup is a technique in direct volume ray casting that pre-computes the optical properties (color and opacity) of volume slabs, assuming linear evolution of scalar data within the slab [S1, unofficial][S2]. It solves the problem of accurately approximating the volume rendering integral for a slab without performing expensive per-slab integration during rendering [S1, unofficial]. It sets itself apart from standard compositing by replacing the per-sample transfer function application with a 3D texture lookup indexed by entry density, exit density, and slab thickness [S1, unofficial][S2].
## How it works
**Inputs and Outputs:**
- Input: 3D scalar volume data, transfer functions (opacity and color), slab thickness parameter $d$.
- Output: Final pixel color via front-to-back compositing.

**Data Structures:**
- **Pre-integration Table:** A 3D texture (or 3D lookup table) of size $N \times N \times N$ (typically $256^3$ for 8-bit scalar data) [S2].
    - Dimensions correspond to:
        - $s_f$: Entry scalar value (front sample) [S2].
        - $s_m$: Middle scalar value [S2].
        - $s_b$: Exit scalar value (back sample) [S2].
    - Stored values: Pre-computed color ($C$) and opacity ($\alpha$) contributions for the slab defined by these three scalar values [S2].
- **Volume Data:** 3D texture or array storing scalar values [S2].

**Method Steps:**
1. **Pre-computation (Offline or Initialization):**
   - For every triplet $(s_f, s_m, s_b)$ in the domain of the transfer functions:
     - Assume the scalar field varies linearly from $s_f$ to $s_b$ through $s_m$ over the slab thickness $2d$ [S2].
     - Compute the integrated color and opacity for this slab using the transfer functions. The integral accounts for the attenuation of light as it passes through the slab with varying density [S1, unofficial][S2].
     - Store the resulting $(C, \alpha)$ in the 3D pre-integration table at index $(s_f, s_m, s_b)$ [S2].
   - *Note:* The sources describe the table as providing "optical properties for volume slices" where data evolution is linear [S1, unofficial]. The specific integral formula is not provided in the text, but the table maps the three scalar endpoints to the slab's cumulative optical effect [S1, unofficial][S2].

2. **Rendering (Per Pixel):**
   - For each pixel, cast a ray through the volume.
   - Divide the ray into slabs of thickness $2d$ [S2].
   - For each slab:
     - Retrieve the three scalar sample values from the volume data: $s_f$ (front), $s_m$ (middle), $s_b$ (back) [S2].
     - Perform a 3D texture lookup in the pre-integration table using $(s_f, s_m, s_b)$ as coordinates [S2].
     - Fetch the pre-computed color $C_{slab}$ and opacity $\alpha_{slab}$ for this slab [S2].
   - Composite the slabs using front-to-back compositing:
     $$ C_{out} = C_{in} + C_{slab} \cdot \alpha_{slab} \cdot (1 - \alpha_{in}) $$
     $$ \alpha_{out} = \alpha_{in} + \alpha_{slab} \cdot (1 - \alpha_{in}) $$
     where $C_{in}, \alpha_{in}$ are the accumulated color and opacity from previous slabs [S2].
   - Repeat for all slabs along the ray.

**Parameters:**
- Slab thickness $d$: Defines the resolution of the pre-integration. Smaller $d$ increases accuracy but requires more slabs and larger tables if the domain is fixed [S2].
- Table size: Typically $256^3$ for 8-bit data [S2].

**Edge Cases and Pitfalls:**
- **Precision:** The table size is limited by memory. For 8-bit data, $256^3$ entries may be large [S2].
- **Linear Assumption:** The method assumes linear scalar evolution within the slab. If the scalar field varies non-linearly within a slab, errors may occur [S1, unofficial].
- **Boundary Conditions:** The correlation $s_f = s_b$ for adjacent slabs ensures continuity [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pre-computation phase
def precompute_table(volume_resolution, slab_thickness, transfer_color, transfer_opacity):
    # Initialize 3D table of size 256x256x256
    table = zeros(256, 256, 256, 4) # RGBA
    
    for sf in range(256):
        for sm in range(256):
            for sb in range(256):
                # Compute integrated color and opacity for slab with front=SF, mid=SM, back=SB
                # This involves integrating the transfer functions over the linear path
                # Formula not explicitly given in sources, but implied by "pre-calculated functions" [S1, unofficial]
                c, a = integrate_slab_optical_properties(sf, sm, sb, slab_thickness, transfer_color, transfer_opacity)
                table[sf, sm, sb] = [c.r, c.g, c.b, a]
    return table

# Rendering phase
def render_ray(ray_origin, ray_direction, volume_data, pre_int_table, slab_thickness):
    accumulated_color = [0.0, 0.0, 0.0, 0.0]
    t = 0.0
    
    while t < ray_length:
        # Define slab boundaries
        slab_front = t
        slab_back = t + 2 * slab_thickness
        
        # Sample scalar values at front, middle, back of slab
        sf = sample_volume(volume_data, ray_origin + ray_direction * slab_front)
        sm = sample_volume(volume_data, ray_origin + ray_direction * (slab_front + slab_thickness))
        sb = sample_volume(volume_data, ray_origin + ray_direction * slab_back)
        
        # Lookup pre-integrated properties
        # Note: sf, sm, sb must be quantized to table indices
        idx_f = quantize(sf)
        idx_m = quantize(sm)
        idx_b = quantize(sb)
        
        slab_color, slab_alpha = pre_int_table[idx_f, idx_m, idx_b]
        
        # Front-to-back compositing
        # C_out = C_in + C_slab * alpha_slab * (1 - alpha_in)
        # alpha_out = alpha_in + alpha_slab * (1 - alpha_in)
        
        if slab_alpha > 0:
            weight = slab_alpha * (1 - accumulated_color[3])
            accumulated_color[0] += slab_color[0] * weight
            accumulated_color[1] += slab_color[1] * weight
            accumulated_color[2] += slab_color[2] * weight
            accumulated_color[3] += slab_alpha * (1 - accumulated_color[3])
        
        t += 2 * slab_thickness
        
    return accumulated_color
```
## Subtopics

### [Pre-Integrated Diffuse Shading with Non-Linear Gradient Interpolation](4932-pre-integrated-diffuse-shading-with-non-linear-gradient-inte.md)
This method computes the diffuse component of Blinn-Phong shading in direct volume ray casting by approximating the integral of light interaction over a sampling step using non-linear gradient interpolation [4932-pre-integrated-diffuse-shading-with-non-linear-gradient-inte.md]. It decomposes the shading integral into four 2D look-up tables (LUTs) indexed by front and back scalar values, weighted by coefficients derived from light direction and gradient angles [4932-pre-integrated-diffuse-shading-with-non-linear-gradient-inte.md]. It sets itself apart from linear gradient interpolation by reducing shading errors caused by large angles between consecutive sample gradients, while avoiding the prohibitive memory cost of full 4D normal-based pre-integration [4932-pre-integrated-diffuse-shading-with-non-linear-gradient-inte.md]. Choose this when accurate shading is required but 4D tables are too large. Link: [Pre-Integrated Diffuse Shading with Non-Linear Gradient Interpolation](4932-pre-integrated-diffuse-shading-with-non-linear-gradient-inte.md).

### [Non-Linear Gradient Interpolation for Pre-Integrated Volume Rendering](4933-non-linear-gradient-interpolation-for-pre-integrated-volume-.md)
This method approximates the non-linear path of a normalized gradient vector between front and back sample normals in pre-integrated volume rendering using a quadratic polynomial, thereby correcting the energy loss and shading errors inherent in linear interpolation. It enables the use of 2D pre-integration lookup tables instead of 3D tables, maintaining low memory usage and allowing real-time updates during transfer function changes. Choose this when shading accuracy is critical and memory is constrained. Link: [Non-Linear Gradient Interpolation for Pre-Integrated Volume Rendering](4933-non-linear-gradient-interpolation-for-pre-integrated-volume-.md).

### Pre-integrated Volume Rendering with Shading
A method for efficiently reconstructing 3D medical images that combines pre-integrated volume rendering with the Shear-Warp algorithm to eliminate aliasing artifacts from under-sampling and improve reconstruction quality via Phong lighting shading [S3, unofficial]. It implements classification coding for volume data by utilizing the correlation of the opacity transfer function (OTF) and a min-max Octree data structure, establishing a new run-length encoded (RLE) data structure for fast reconstruction when OTF and view angle change [S3, unofficial]. It incorporates pre-integrated volume rendering with Phong lighting shading calculation to eliminate aliasing artifacts [S3, unofficial]. Choose this for fast, high-quality medical image reconstruction with dynamic OTF changes.

### Simplified Pre-integration
A faster approximation of pre-integration that neglects color attenuation inside volume slabs, used in the first level of interaction where multiple TFs are modified [S4]. It is applied in the first level of the interface to handle actions that modify several TFs simultaneously, while full pre-integration is applied in the second level where a single TF is under modification, to maintain quality without compromising interactivity [S4]. Choose this for interactive transfer function design tools where speed is prioritized for multi-TF adjustments.

## Operating characteristics
- **Time Complexity:** Pre-computation is $O(N^3)$ where $N$ is the number of scalar levels (e.g., $256^3$) [S2]. Rendering is $O(1)$ per slab for the lookup, plus sampling cost [S2].
- **Memory Complexity:** $O(N^3)$ for the 3D pre-integration table [S2]. For 8-bit data, $256^3$ entries, each storing RGBA, can be large [S2].
- **Measured Performance:** Implemented on GPU for real-time performance [S1, unofficial][S2]. The 3D texture lookup is efficient on modern GPUs [S2].
- **Strengths:** Accurate approximation of the volume rendering integral for slabs; efficient rendering via texture lookup; supports real-time updates if tables are small or recomputed efficiently [S1, unofficial][S2].
- **Limitations:** High memory usage for large tables; assumes linear scalar evolution within slabs; less accurate for non-linear data [S1, unofficial].
- **When to use:** When accurate volume rendering is needed and memory allows for large 3D tables; suitable for GPU implementation [S1, unofficial][S2].
- **When not to use:** When memory is severely constrained; when data varies non-linearly within slabs and high accuracy is required [S1, unofficial].
## Background
Originated in the context of GPU-based volume rendering to accelerate the evaluation of the volume rendering integral [S1, unofficial][S2]. The concept of pre-integrated transfer functions was introduced to provide optical properties for volume slices assuming linear data evolution [S1, unofficial]. It has been implemented on GPUs for real-time rendering [S1, unofficial][S2]. Variants include shading-aware pre-integration [S3, unofficial] and simplified pre-integration for interactive tools [S4].
## Sources
[S1] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
[S2] Second Order Pre-Integrated Volume Rendering, https://people.freedesktop.org/~marcheu/pacificvis08.pdf, fetched 2026-10-29
[S3] The Perspective Shear-Warp Algorithm in a Virtual Environment., https://www.researchgate.net/publication/220943822_The_Perspective_Shear-Warp_Algorithm_in_a_Virtual_Environment, fetched 2026-10-29 (unofficial)
[S4] Volume visualization and exploration through flexible transfer function design, https://doi.org/10.1016/j.cag.2008.04.004, fetched 2026-10-27
