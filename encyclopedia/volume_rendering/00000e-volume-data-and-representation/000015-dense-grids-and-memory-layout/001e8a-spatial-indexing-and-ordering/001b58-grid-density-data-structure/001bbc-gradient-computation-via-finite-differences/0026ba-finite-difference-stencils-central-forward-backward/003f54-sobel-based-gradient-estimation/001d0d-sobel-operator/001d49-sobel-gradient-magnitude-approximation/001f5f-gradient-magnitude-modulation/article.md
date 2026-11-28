# Gradient Magnitude Modulation

Also called: Gradient Magnitude Approximation, Gradient Magnagnitude Calculation, Gradient Magnitude Illumination Modulation, Gradient Magnitude Opacity Modulation, Maximum Gradient Importance, Maximum Gradient Magnitude Measure

## Overview
Gradient Magnitude Modulation is a volume rendering technique that uses the computed gradient magnitude of scalar data to modulate sample opacity and illumination values [S1, S3, unofficial]. It solves the problem of occlusion by internal features of lower scalar intensity, which often occurs when using maximum intensity projection, by treating high gradient magnitudes as indicators of surface presence or importance [S2]. It sets itself apart from its sibling [Sobel Gradient Magnitude Approximation](1429-sobel-gradient-magnitude-approximation.md) by defining how the calculated magnitude is applied to the rendering pipeline (modulation) rather than just how it is computed [G].

## How it works
**Inputs:**
*   Scalar volume data [S1, S3, unofficial].
*   Current sample opacity ($\alpha$) and illumination ($I$) [S1, unofficial].
*   Accumulated transparency state (for the "Maximum Gradient" variant) [S2].

**Outputs:**
*   Modulated opacity and/or illumination values for the current sample [S1, unofficial].

**Data Structures:**
*   **Gradient Magnitude Lookup Table (GmLUT):** Stores eight table entries specifying points of a piece-wise linear function at equal intervals along the gradient magnitude axis [S1, unofficial].
*   **Registers:**
    *   `Gmimd`: Gradient Magnitude Illumination Modulation for diffuse illumination [S1, unofficial].
    *   `Gmims`: Gradient Magnitude Illumination Modulation for specular illumination [S1, unofficial].
    *   `Gmom`: Gradient Magnitude Opacity Modulation [S1, unofficial].

**Steps:**

1.  **Compute Gradient Magnitude:**
    Calculate the magnitude $|\nabla|$ from the gradient components.
    *   *Approximation (VolumePro):* Compute the square of the gradient magnitude by summing the squares of the components, then use a Newton-Raphson iteration to compute the square root [S1, unofficial]. The result is an 8-bit approximation in the range $[0, 2^{20}]$ [S1, unofficial].
    *   *Direct Estimation:* If components are not pre-computed, estimate them using forward differences [S3, unofficial].
        $$ \nabla_x = V(x+d, y, z) - V(x, y, z) $$
        $$ \nabla_y = V(x, y+d, z) - V(x, y, z) $$
        $$ \nabla_z = V(x, y, z+d) - V(x, y, z) $$
        $$ |\nabla| = \sqrt{\nabla_x^2 + \nabla_y^2 + \nabla_z^2} $$
        where $d$ is the step length [S3, unofficial].

2.  **Map to Modulation Range:**
    Map the computed gradient magnitude into the range $[0, 255]$ using a user-specified piece-wise linear function stored in the GmLUT [S1, unofficial].
    *   The GmLUT contains 8 entries defining the function at equal intervals [S1, unofficial].
    *   Perform linear interpolation between table values to compute an 8-bit repeating-fraction modulation value $\beta \in [0, 1]$ [S1, unofficial].

3.  **Apply Modulation:**
    The registers `Gmimd`, `Gmims`, and `Gmom` enable the optional multiplication of the sample's opacity and illumination by the interpolated modulation value [S1, unofficial].
    $$ \alpha_{mod} = \alpha \cdot \beta $$
    $$ I_{mod} = I \cdot \beta $$

    *Alternative "Maximum Gradient" Accumulation Strategy:*
    In some implementations, $\beta$ is calculated dynamically based on the maximum gradient magnitude encountered along the viewing ray so far ($\text{grad}_{acc}$) [S2].
    *   Initialize $\text{grad}_{acc}$ to the gradient magnitude of the first sample or zero [S2].
    *   For each subsequent sample $i$ with gradient magnitude $\text{grad}_i$:
        $$ \beta = \begin{cases} 1 - (\text{grad}_i - \text{grad}_{acc}) & \text{if } \text{grad}_i > \text{grad}_{acc} \\ 1 & \text{otherwise} \end{cases} $$
        If $\text{grad}_i > \text{grad}_{acc}$, update $\text{grad}_{acc} = \text{grad}_i$ [S2].
    *   Use this $\beta$ to modulate accumulated transparency [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_gradient_magnitude(volume, position, step_length, intensity):
    """
    Computes gradient magnitude using forward differences.
    Returns float magnitude.
    """
    d = step_length
    dx = volume.sample(position + vec3(d, 0, 0)).r - intensity
    dy = volume.sample(position + vec3(0, d, 0)).r - intensity
    dz = volume.sample(position + vec3(0, 0, d)).r - intensity
    return sqrt(dx*dx + dy*dy + dz*dz)

def apply_gradient_modulation(magnitude, alpha, illumination, gmlut, gmom, gmimd, gmims, max_magnitude=2**20):
    """
    Applies gradient magnitude modulation to opacity and illumination.
    gmlut: list of 8 floats defining piece-wise linear function at equal intervals over [0, 255].
    """
    # 1. Normalize magnitude to [0, 255] range for LUT indexing
    t = max(0.0, min(1.0, magnitude / max_magnitude)) * 255.0
    
    # 2. Interpolate using 8 control points at equal intervals
    # 8 points create 7 equal segments over [0, 255]
    seg = min(int(t * 7 / 255.0), 6)
    x0 = seg * 255.0 / 7.0
    x1 = (seg + 1) * 255.0 / 7.0
    y0 = gmlut[seg]
    y1 = gmlut[seg + 1]
    
    # Linear interpolation between segment endpoints
    beta = y0 + (y1 - y0) * (t - x0) / (x1 - x0) if x1 != x0 else y0
    
    mod_alpha = alpha
    mod_illum = illumination
    
    # 3. Apply based on registers
    if gmom:
        mod_alpha = alpha * beta
    if gmimd or gmims:
        mod_illum = illumination * beta
        
    return mod_alpha, mod_illum
```

**Edge Cases and Pitfalls:**
*   **Noise:** Small gradient magnitudes in homogeneous regions (noise) can cause unwanted reflections or opacity if not modulated [S1, unofficial].
*   **Occlusion:** Without modulation (or using max intensity), internal features may be occluded by external structures with higher scalar intensity but lower gradient [S2].
*   **Precision:** The VolumePro implementation uses an 8-bit approximation for the gradient magnitude, which may introduce quantization artifacts [S1, unofficial].

## Subtopics

### [Adjacent Gradient Difference](5787-adjacent-gradient-difference.md)
Calculates importance by comparing gradient magnitudes of two adjacent samples along a viewing ray, increasing transparency when gradient magnitude increases. It differs from absolute gradient magnitude modulation by focusing on the *change* in gradient rather than the absolute value.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample for modulation after gradient computation [S3, unofficial].
*   **Memory Complexity:** Requires storage for the GmLUT (8 entries) and registers [S1, unofficial].
*   **Strengths:** Emphasizes surface boundaries, reduces reflection from non-surfaces (noise), and reveals sharp features that might be occluded in max intensity projection [S1, unofficial][S2].
*   **Limitations:** Requires gradient computation, which increases memory bandwidth usage [S3, unofficial]. The 8-bit approximation may lose precision [S1, unofficial].

## Background
Gradient magnitude modulation was introduced in the VolumePro real-time ray-casting system [S1, unofficial]. It was designed to address the limitations of maximum intensity projection, where internal features of lower scalar intensity are occluded by external structures [S2]. The method uses the gradient magnitude as an indicator of surface presence, modulating opacity and illumination to emphasize surfaces and reduce noise impact [S1, S3, unofficial].

## Sources
[S1] The VolumePro Real-Time Ray-Casting System, https://www.merl.com/publications/docs/TR99-19.pdf, fetched 2026-10-26 (unofficial)
[S2] Fast Volumetric Data Exploration with Importance-Based Accumulated Transparency Modulation, https://doi.org/10.2312/vg/vg10/061-068, fetched 2026-10-26
[S3] GPU-accelerated single-pass volumetric raycasting in Qt ..., https://martinopilia.com/posts/2018/09/17/volume-raycasting.html, fetched 2026-10-27 (unofficial)
