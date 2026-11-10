# IsosurfaceClassifier Lighting Model

Also called: Isosurface Volume Rendering with Phong Lighting

## Overview
The IsosurfaceClassifier Lighting Model is a volume rendering technique that visualizes an isosurface within a volume dataset by treating samples as surface points with opacity determined by a Gaussian transfer function, and computing color using a Phong illumination model (ambient, diffuse, specular) with normals derived from the volume gradient [S1, unofficial]. It applies surface-like shading to volumetric data near an isovalue, with transparency based on distance from the isovalue [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A `VolumeSampleData` structure containing the scalar value, position, and gradient of the current sample; a `Camera` object for view direction; an array of `Light` objects; parameters `isovalue`, `sigma`, `kappa`, `ambient_magnitude`, `specular_shininess`, and step size `dt`; and a threshold `DELTA` [S1, unofficial].
*   **Output:** An `OpticsData` structure containing the accumulated emission color and transparency (opacity) for the sample [S1, unofficial].

**Data Structures**
*   `VolumeSampleData`: Contains `position` (3D vector), `gradient` (3D vector), and scalar value accessors [S1, unofficial].
*   `OpticsData`: Contains `color` (3D vector, RGB) and `transparency` (3D vector) [S1, unofficial].
*   `Light`: Contains `getPosition()` (3D vector) and `getRadiance()` (3D vector) [S1, unofficial].
*   `Camera`: Contains `getPosition()` (3D vector) [S1, unofficial].

**Parameters**
*   `isovalue`: The scalar value defining the isosurface. Range: $[-1.0, 1.0]$. Default: $0.0$ [S1, unofficial].
*   `sigma`: Standard deviation for the Gaussian transfer function. Range: $[0.005, 0.1]$. Default: $0.01$ [S1, unofficial].
*   `kappa`: Absorption coefficient for Beer-Lambert transparency. Default: $4000.0$ [S1, unofficial].
*   `ambient_magnitude`: Multiplier for ambient light intensity. Range: $[0.01, 0.5]$ [S1, unofficial].
*   `specular_shininess`: Exponent for the Blinn-Phong specular highlight. Range: $[1.0, 50.0]$ [S1, unofficial].
*   `DELTA`: Threshold for the transfer value. If the transfer value is less than or equal to `DELTA`, no lighting is computed. Exact numerical value is not given in the sources [S1, unofficial].
*   `dt`: Step size (distance) along the ray [S1, unofficial].

**Algorithm Steps**
1.  **Compute Transfer Value:** Calculate the Gaussian transfer value `val` based on the distance between the sample's scalar value and the `isovalue`, scaled by `sigma` [S1, unofficial].
    $$ val = \exp\left(-\frac{(v_{data}.value - isovalue)^2}{2 \cdot \sigma^2}\right) $$
2.  **Initialize Emission:** Initialize the emission accumulator vector $q$ to $(0, 0, 0)$ [S1, unofficial].
3.  **Check Threshold:** If $val \le DELTA$, skip lighting calculations. The sample contributes no emission [S1, unofficial].
4.  **Iterate Lights:** For each light $L_i$ in the light list (from $i = 0$ to $lis\_num - 1$):
    a.  **Compute View Direction:** Calculate the normalized vector from the sample position to the camera position [S1, unofficial].
        $$ \vec{v} = \frac{\text{cam}.position - v_{data}.position}{\|\text{cam}.position - v_{data}.position\|} $$
    b.  **Compute Normal:** Normalize the volume gradient to obtain the surface normal [S1, unofficial].
        $$ \vec{n} = \frac{v_{data}.gradient}{\|v_{data}.gradient\|} $$
    c.  **Map Color:** Retrieve the base visualization color $C_{base}$ using the transfer value and normal [S1, unofficial].
    d.  **Compute Ambient Component:** Calculate the ambient light contribution [S1, unofficial].
        $$ C_a = L_i.radiance \cdot ambient\_magnitude $$
    e.  **Compute Light Direction:** Calculate the normalized vector from the sample position to the light position [S1, unofficial].
        $$ \vec{l} = \frac{L_i.position - v_{data}.position}{\|L_i.position - v_{data}.position\|} $$
    f.  **Compute Diffuse Component:** Calculate the Lambertian diffuse shading factor [S1, unofficial].
        $$ diff = \max(0.0, \vec{n} \cdot \vec{l}) $$
        $$ C_d = diff \cdot L_i.radiance $$
    g.  **Compute Specular Component:** Calculate the Blinn-Phong specular highlight [S1, unofficial].
        $$ \vec{h} = \frac{\vec{v} + \vec{l}}{\|\vec{v} + \vec{l}\|} $$
        $$ spec = \max(0.0, \vec{n} \cdot \vec{h})^{specular\_shininess} $$
        $$ C_s = spec \cdot L_i.radiance $$
    h.  **Accumulate Emission:** Add the weighted lighting contribution to the emission accumulator using element-wise multiplication [S1, unofficial].
        $$ q = q + (C_a + C_d + C_s) \odot C_{base} $$
5.  **Compute Transparency:** Calculate the transparency using the Beer-Lambert law [S1, unofficial].
    $$ T = \exp(-\kappa \cdot val \cdot dt) $$
6.  **Compute Output Color:** Scale the accumulated emission by the transfer value, step size, and a constant scaling factor [S1, unofficial].
    $$ color = 1000.0 \cdot q \cdot val \cdot dt $$

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def IsosurfaceClassifier_transfer(v_data, cam, lights, lis_num, 
                                  isovalue, sigma, kappa, 
                                  ambient_magnitude, specular_shininess, dt, DELTA):
    # 1. Compute Transfer Value
    diff_val = v_data.value - isovalue
    val = math.exp(-(diff_val * diff_val) / (2.0 * sigma * sigma))
    
    # 2. Initialize Emission
    q = [0.0, 0.0, 0.0]
    
    # 3. Check Threshold
    if val > DELTA:
        # 4. Iterate Lights
        for i in range(lis_num):
            li = lights[i]
            
            # 4a. Compute View Direction
            view_dir = normalize([cam.position[j] - v_data.position[j] for j in range(3)])
            
            # 4b. Compute Normal
            normal = normalize(v_data.gradient)
            
            # 4c. Map Color
            # Sources do not specify the exact colormap mapping logic [G]
            surface_color = [0.0, 0.0, 0.0]  # Placeholder for getMappedColor(val, normal)
            
            # 4d. Compute Ambient Component
            C_a = [li.radiance[j] * ambient_magnitude for j in range(3)]
            
            # 4e. Compute Light Direction
            pos_diff = [li.position[j] - v_data.position[j] for j in range(3)]
            li_dist = math.sqrt(sum(d*d for d in pos_diff))
            li_dir = [d / li_dist for d in pos_diff]
            
            # 4f. Compute Diffuse Component
            diff_factor = max(0.0, dot(normal, li_dir))
            C_d = [diff_factor * li.radiance[j] for j in range(3)]
            
            # 4g. Compute Specular Component (Blinn-Phong)
            halfway_dir = normalize([view_dir[j] + li_dir[j] for j in range(3)])
            spec_factor = pow(max(0.0, dot(normal, halfway_dir)), specular_shininess)
            C_s = [spec_factor * li.radiance[j] for j in range(3)]
            
            # 4h. Accumulate Emission
            lighting_sum = [C_a[j] + C_d[j] + C_s[j] for j in range(3)]
            q = [q[j] + lighting_sum[j] * surface_color[j] for j in range(3)]
            
    # 5. Compute Transparency
    transparency = [math.exp(-kappa * val * dt) for _ in range(3)]
    
    # 6. Compute Output Color
    color = [1000.0 * q[j] * val * dt for j in range(3)]
    
    return OpticsData(color, transparency)

def normalize(v):
    mag = math.sqrt(sum(x*x for x in v))
    return [x / mag for x in v] if mag > 0 else [0.0, 0.0, 0.0]

def dot(a, b):
    return sum(x*y for x, y in zip(a, b))

class OpticsData:
    def __init__(self, color, transparency):
        self.color = color
        self.transparency = transparency
```

**Edge Cases and Pitfalls**
*   **Zero Gradient:** If the volume gradient is zero, the normal cannot be normalized, resulting in undefined behavior or NaNs in lighting calculations [G]. Guard against this by checking gradient magnitude before normalization.
*   **Camera/Light at Sample Position:** If the camera or light position coincides with the sample position, the view or light direction vector becomes zero-length, causing division by zero during normalization [G]. Add a small epsilon to the denominator or check for zero length.
*   **DELTA Threshold:** If `val <= DELTA`, the emission `q` remains zero, making the sample effectively invisible in the final color calculation, although transparency is still computed [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Provides realistic surface-like shading for isosurfaces, enhancing visual perception of shape and depth [S1, unofficial].
*   **Limitations:** Computationally expensive due to per-sample lighting calculations for multiple lights [S1, unofficial]. Sensitive to the choice of `sigma` and `DELTA` [S1, unofficial]. Gradient estimation errors can lead to incorrect normals and shading artifacts [G].
*   **When to Use:** When realistic surface shading is desired for thresholded volume data and computational resources allow for per-sample lighting calculations [S1, unofficial].
*   **When Not to Use:** For real-time applications with many lights or large volumes where performance is critical, or when the volume data has noisy gradients [G].

## Background
The method is implemented in the `IsosurfaceClassifier` class, which inherits from a base `Classifier` class and uses Eigen for vector operations and `tinycolormap` for color mapping [S1, unofficial]. The class fields include `isovalue`, `sigma`, `kappa`, `visualize_target`, and `cm_type` [S1, unofficial]. The technique combines isosurface rendering concepts (thresholding, Gaussian transfer functions) with surface shading techniques (Phong illumination, Blinn-Phong specular highlights) [S1, unofficial].

## Sources
[S1] GitHub - yuehaowang/volume_renderer: CUDA-based interactive volume ..., https://github.com/yuehaowang/volume_renderer, fetched 2026-10-29 (unofficial)
