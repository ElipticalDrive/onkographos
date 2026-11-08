# Gradient Magnitude Modulation

Also called: Gradient Magnitude Approximation, Gradient Magnitude Calculation, Gradient Magnitude Illumination Modulation, Gradient Magnitude Opacity Modulation, Maximum Gradient Importance, Maximum Gradient Magnitude Measure

## Overview
Gradient Magnitude Modulation is a technique in volume rendering that uses the computed gradient magnitude of a sample to modulate its opacity and/or illumination values [S1, S3, unofficial]. It solves the problem of internal features being occluded by external structures with higher scalar intensity, a limitation of Maximum Intensity Projection (MIP) [S2]. By treating gradient magnitude as an importance measure, it emphasizes surface boundaries and reduces the visual impact of noise or homogeneous regions [S1, unofficial][S2]. It builds on [Sobel Gradient Magnitude Approximation](1429-sobel-gradient-magnitude-approximation.md) to obtain the necessary gradient data.

## How it works
The method consists of two phases: computing the gradient magnitude and applying a modulation factor to the sample's optical properties.

### Inputs and Outputs
*   **Inputs**:
    *   Scalar field values at the current sample position and its neighbors.
    *   Current sample opacity ($\alpha$) and illumination ($I$).
    *   Accumulated maximum gradient magnitude along the ray ($\text{grad}_{acc}$) [S2].
    *   Gradient Magnitude Lookup Table (GmLUT) [S1, unofficial].
*   **Outputs**:
    *   Modulated opacity and/or illumination values.
    *   Updated $\text{grad}_{acc}$ for the next sample [S2].

### Data Structures
*   **Gradient Magnitude Lookup Table (GmLUT)**: Stores eight table entries specifying points of a piece-wise linear function at equal intervals along the gradient magnitude axis [S1, unofficial]. The table maps the computed gradient magnitude to a modulation value in the range $[0, 1]$ [S1, unofficial].

### Steps

1.  **Compute Gradient Components**: Calculate the partial derivatives in the x, y, and z directions. Using forward differences with step length $d$:
    $$
    \begin{aligned}
    dx &= V(x+d, y, z) - V(x, y, z) \\
    dy &= V(x, y+d, z) - V(x, y, z) \\
    dz &= V(x, y, z+d) - V(x, y, z)
    \end{aligned}
    $$
    where $V$ is the volume texture/sample function [S3, unofficial].

2.  **Compute Squared Magnitude**: Sum the squares of the components:
    $$
    g^2 = dx^2 + dy^2 + dz^2
    $$
    [S1, unofficial].

3.  **Compute Gradient Magnitude**: Calculate $g = \sqrt{g^2}$.
    *   **Hardware Implementation (VolumePro)**: A Newton-Raphson iteration is used to compute the square root, resulting in an 8-bit approximation in the range $[0, 2^{20}]$ [S1, unofficial].
    *   **Software/GPU Implementation**: Direct square root calculation is used [S3, unofficial].

4.  **Map to Modulation Value**:
    *   Map the gradient magnitude $g$ into the range $[0, 255]$ using a user-specified piece-wise linear function defined by the GmLUT [S1, unofficial].
    *   Perform linear interpolation between the table values to compute an 8-bit repeating-fraction modulation value $\beta \in [0, 1]$ [S1, unofficial].
    *   *Note*: In the "Maximum Gradient Importance" variant, $\beta$ is calculated dynamically based on the accumulated maximum gradient [S2]:
        $$
        \beta = B(\Delta_{imp}(\text{grad}_i, \text{grad}_{acc})) = \begin{cases} 1 - (\text{grad}_i - \text{grad}_{acc}) & \text{if } \text{grad}_i > \text{grad}_{acc} \\ 1 & \text{otherwise} \end{cases}
        $$
        where $\text{grad}_i$ is the current sample's gradient magnitude and $\text{grad}_{acc}$ is the maximum gradient magnitude encountered so far along the ray [S2].
    *   Initialize $\text{grad}_{acc}$ to the gradient magnitude of the first sample or zero [S2]. If $\text{grad}_i > \text{grad}_{acc}$, update $\text{grad}_{acc} = \text{grad}_i$ [S2].

5.  **Apply Modulation**:
    *   Multiply the sample's opacity $\alpha$ and/or illumination $I$ by $\beta$ [S1, unofficial].
    *   Registers control this: `Gmom` for opacity, `Gmimd` for diffuse illumination, and `Gmims` for specular illumination [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gradient_magnitude_modulation(position, intensity, step_length, grad_acc, gm_lut):
    """
    Computes gradient magnitude and returns modulation factor and updated grad_acc.
    
    Args:
        position: vec3, current ray position
        intensity: float, scalar value at current position
        step_length: float, distance between samples
        grad_acc: float, accumulated maximum gradient magnitude so far
        gm_lut: list of 8 floats, lookup table for piece-wise linear mapping
        
    Returns:
        beta: float, modulation factor in [0, 1]
        new_grad_acc: float, updated accumulated maximum gradient
    """
    # Step 1: Compute gradient components (forward differences)
    # Note: texture() assumes normalized coordinates or appropriate scaling
    dx = texture(volume, position + vec3(step_length, 0, 0)).r - intensity
    dy = texture(volume, position + vec3(0, step_length, 0)).r - intensity
    dz = texture(volume, position + vec3(0, 0, step_length)).r - intensity
    
    # Step 2 & 3: Compute magnitude
    grad_sq = dx*dx + dy*dy + dz*dz
    grad_mag = sqrt(grad_sq)
    
    # Step 4: Calculate modulation factor beta
    # Using Maximum Gradient Importance logic [S2]
    if grad_mag > grad_acc:
        # Update accumulated max gradient
        new_grad_acc = grad_mag
        # Calculate beta based on difference
        # Note: The formula 1 - (grad_i - grad_acc) can exceed 1 or be negative 
        # if not clamped. The source implies it acts as a transparency increase.
        # We clamp to [0, 1] for valid modulation factor.
        beta = 1.0 - (grad_mag - grad_acc)
        beta = max(0.0, min(1.0, beta))
    else:
        new_grad_acc = grad_acc
        beta = 1.0
        
    # Alternative: If using GmLUT [S1, unofficial], beta would be:
    # mapped_val = map_grad_to_0_255(grad_mag)
    # beta = interpolate_gm_lut(gm_lut, mapped_val)
    
    return beta, new_grad_acc

def apply_modulation(opacity, illumination, beta):
    """
    Modulates opacity and illumination by beta.
    """
    modulated_opacity = opacity * beta
    modulated_illumination = illumination * beta
    return modulated_opacity, modulated_illumination
```

### Parameters
*   **GmLUT**: 8-entry lookup table for piece-wise linear mapping [S1, unofficial].
*   **Registers**: `Gmom`, `Gmimd`, `Gmims` control which channels are modulated [S1, unofficial].
*   **Step Length ($d$)**: Distance between samples, used in finite difference calculations [S3, unofficial].

### Edge Cases and Pitfalls
*   **Precision**: Hardware implementations may use 8-bit signed precision for gradient components, representing values in $[-127, 127]$ [S1, unofficial].
*   **Noise**: Small gradient magnitudes in noisy data can lead to low modulation values, effectively reducing noise impact [S1, unofficial].
*   **Homogeneous Regions**: Regions with small gradient magnitudes (e.g., interior of walls) become more transparent, which is desirable for revealing internal structures [S1, unofficial].

## Subtopics

### [Adjacent Gradient Difference](5787-adjacent-gradient-difference.md)
A variant that calculates importance difference by comparing gradient magnitudes of two adjacent samples, increasing accumulated transparency whenever an increase in gradient magnitude is detected.

### [Maximum Gradient Magnitude Projection](5788-maximum-gradient-magnitude-projection.md)
A projection technique that uses the maximum gradient magnitude along the ray as the importance measure, better revealing sharp features compared to MIP.

## Operating characteristics
*   **Time Complexity**: Computing the gradient magnitude requires 3 texture lookups and basic arithmetic operations per sample [S3, unofficial]. The lookup table interpolation is constant time [S1, unofficial].
*   **Memory Complexity**: Requires storage for the GmLUT (8 entries) [S1, unofficial] and the accumulated gradient magnitude state [S2].
*   **Strengths**:
    *   Emphasizes surface boundaries [S1, unofficial].
    *   Reduces reflection from non-surfaces (noise) [S1, unofficial].
    *   Reveals internal features occluded by high-intensity external structures, outperforming MIP in revealing sharp features [S2].
*   **Limitations**:
    *   Requires additional texture lookups for gradient computation [S3, unofficial].
    *   Synchronization issues may arise if gradient computation is delayed relative to sample processing (e.g., in hardware pipelines) [S1, unofficial].
*   **When to Use**: When surface emphasis is desired, or when internal structures need to be visible despite high-intensity occluders [S1, unofficial][S2].
*   **When Not to Use**: When maximum intensity projection is sufficient, or when computational resources are extremely limited and gradient computation is prohibitive [S2].

## Background
Gradient magnitude modulation was introduced in the VolumePro real-time ray-casting system [S1, unofficial]. It was developed to address the limitations of maximum intensity projection, where internal features of lower scalar intensity are occluded by external structures [S2]. The technique uses the gradient magnitude as an indication of surface presence [S1, unofficial]. It is widely used in transfer functions to emphasize surfaces [S3, unofficial].

## Sources
[S1] The VolumePro Real-Time Ray-Casting System, https://www.merl.com/publications/docs/TR99-19.pdf, fetched 2026-10-26 (unofficial)
[S2] Fast Volumetric Data Exploration with Importance-Based Accumulated Transparency Modulation, https://doi.org/10.2312/vg/vg10/061-068, fetched 2026-10-26
[S3] GPU-accelerated single-pass volumetric raycasting in Qt ..., https://martinopilia.com/posts/2018/09/17/volume-raycasting.html, fetched 2026-10-27 (unofficial)
