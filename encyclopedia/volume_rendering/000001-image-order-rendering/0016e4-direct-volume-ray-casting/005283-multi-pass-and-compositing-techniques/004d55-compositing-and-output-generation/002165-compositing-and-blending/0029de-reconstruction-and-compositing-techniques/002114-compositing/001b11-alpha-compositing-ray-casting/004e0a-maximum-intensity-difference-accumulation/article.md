# Maximum Intensity Difference Accumulation

Also called: MIDA, Monotonically Increasing Density Accumulation

## Overview
Maximum Intensity Difference Accumulation (MIDA) is a volume rendering compositing technique that modifies standard Direct Volume Rendering (DVR) to prevent high-intensity structures from being occluded by irrelevant foreground material [S4]. Unlike standard DVR, where accumulated opacity grows monotonically and causes "fog" that hides distant features, MIDA modulates previously accumulated color and opacity based on the difference between the current sample's intensity and the current maximum intensity along the ray [S4]. This allows MIDA to combine the spatial context and shading compatibility of DVR with the parameterless, high-visibility nature of Maximum Intensity Projection (MIP) [S2, S4].

## How it works
### Inputs and Outputs
**Inputs:**
*   A scalar volumetric dataset $V$.
*   A viewing ray defined by origin and direction.
*   A transfer function mapping scalar values to color $c(s)$ and opacity $\alpha(s)$. MIDA typically uses simple linear mappings [S4].
*   A modulation parameter $\gamma \in [-1, 1]$ to control the transition between DVR, MIDA, and MIP [S4].
*   (Optional) A user-defined intensity range $[s_{min}, s_{max}]$ to filter features [S2].
*   (Optional) Brightness compensation parameters $K$, $Range_{total}$, $Range_{sub}$ [S3, unofficial].

**Outputs:**
*   Final accumulated color $C_{out}$ and opacity $\alpha_{out}$ for the pixel.

### Data Structures
*   **Accumulated Color ($c_{acc}$):** RGB vector, initialized to $(0, 0, 0)$.
*   **Accumulated Opacity ($\alpha_{acc}$):** Scalar, initialized to $0.0$.
*   **Current Maximum Intensity ($s_{max}$):** Scalar, initialized to $-\infty$ (or minimum possible data value).
*   **Sample Color ($c_{src}$):** RGB vector, derived from transfer function $c(f(P_i))$.
*   **Sample Opacity ($\alpha_{src}$):** Scalar, derived from transfer function $\alpha(f(P_i))$.
*   **Sample Intensity ($s_i$):** Scalar, $f(P_i)$.

### Steps
1.  **Initialization:** Initialize $c_{acc} = (0, 0, 0)$, $\alpha_{acc} = 0.0$, and $s_{max} = -\infty$.
2.  **Ray Traversal:** Traverse the volume from front to back (eye to object). For each sample $i$ at position $P_i$:
    a.  **Sample Data:** Retrieve scalar value $s_i = f(P_i)$.
    b.  **Classify:** Compute sample color $c_{src} = c(s_i)$ and sample opacity $\alpha_{src} = \alpha(s_i)$ using the transfer function.
    c.  **Update Maximum:** If $s_i > s_{max}$, calculate the delta $\delta_i = s_i - s_{max}$ and update $s_{max} = s_i$. Otherwise, $\delta_i = 0$.
    d.  **Calculate Modulation Factor ($\beta_i$):**
        *   If $\gamma \ge 0$ (MIDA to MIP transition): $\beta_i = 1 - \delta_i$.
        *   If $\gamma < 0$ (MIDA to DVR transition): $\beta_i = 1 - \delta_i (1 + \gamma)$.
        *   *Note:* $\delta_i$ is clamped to $[0, 1]$ if data is normalized to $[0, 1]$. If data is not normalized, $\delta_i$ must be scaled or clamped to prevent $\beta_i < 0$ [S4, S5].
    e.  **Modulate Accumulation:** Apply $\beta_i$ to the previously accumulated values:
        $$c_{acc} \leftarrow \beta_i \cdot c_{acc}$$
        $$\alpha_{acc} \leftarrow \beta_i \cdot \alpha_{acc}$$
    f.  **Composite:** Perform standard front-to-back compositing with the current sample:
        $$c_{acc} \leftarrow c_{acc} + (1 - \alpha_{acc}) \cdot c_{src}$$
        $$\alpha_{acc} \leftarrow \alpha_{acc} + (1 - \alpha_{acc}) \cdot \alpha_{src}$$
3.  **MIP Transition (Image Space):** If $\gamma > 0$, after the ray traversal is complete, linearly interpolate between the accumulated MIDA result $(c_{acc}, \alpha_{acc})$ and the MIP result $(c_{mip}, \alpha_{mip})$ using $\gamma$ as the weight. Interpolation is performed on opacity-weighted colors [S4].
    $$c_{out} = \text{lerp\_weighted}(c_{acc}, c_{mip}, \gamma)$$
    $$\alpha_{out} = \text{lerp}(\alpha_{acc}, \alpha_{mip}, \gamma)$$
    If $\gamma \le 0$, $c_{out} = c_{acc}$ and $\alpha_{out} = \alpha_{acc}$.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_mida_ray(ray_origin, ray_direction, volume, transfer_function, gamma):
    """
    Renders a single ray using MIDA compositing.
    
    Args:
        ray_origin: vec3
        ray_direction: vec3
        volume: Volume object providing sample(P) -> scalar
        transfer_function: Function mapping scalar -> (color: vec3, alpha: float)
        gamma: float in [-1, 1]. -1=DVR, 0=MIDA, 1=MIP.
        
    Returns:
        (color: vec3, alpha: float)
    """
    # Initialization
    acc_color = vec3(0.0, 0.0, 0.0)
    acc_alpha = 0.0
    max_intensity = -float('inf')
    
    # MIP tracking
    mip_color = vec3(0.0, 0.0, 0.0)
    mip_alpha = 0.0
    mip_max_intensity = -float('inf')
    
    # Ray traversal parameters
    step_size = 0.01 # Example step size
    num_samples = 100 # Example count
    
    for i in range(num_samples):
        # Sample position
        pos = ray_origin + ray_direction * (i * step_size)
        
        # Get scalar value
        s_i = volume.sample(pos)
        
        # Classify
        c_src, a_src = transfer_function(s_i)
        
        # Update Maximum Intensity
        delta = 0.0
        if s_i > max_intensity:
            delta = s_i - max_intensity
            max_intensity = s_i
            
        # Calculate Modulation Factor beta
        # Clamp delta to [0, 1] assuming normalized data [0,1]
        # If data is not normalized, delta should be scaled or clamped to prevent beta < 0
        delta = max(0.0, min(1.0, delta))
        
        if gamma < 0:
            beta = 1.0 - delta * (1.0 + gamma)
        else:
            beta = 1.0 - delta
            
        # Modulate accumulated values
        acc_color = beta * acc_color
        acc_alpha = beta * acc_alpha
        
        # Standard Compositing
        # c_dst <- c_dst + (1 - alpha_dst) * c_src
        # alpha_dst <- alpha_dst + (1 - alpha_dst) * alpha_src
        acc_color = acc_color + (1.0 - acc_alpha) * c_src
        acc_alpha = acc_alpha + (1.0 - acc_alpha) * a_src
        
        # Track MIP result for gamma > 0 transition
        if s_i > mip_max_intensity:
            mip_max_intensity = s_i
            mip_color, mip_alpha = transfer_function(s_i)
            
    # Handle gamma > 0: Image space interpolation to MIP
    if gamma > 0:
        # Interpolate between MIDA and MIP results
        # Using opacity-weighted colors for interpolation
        # c_mip_weighted = c_mip * alpha_mip
        # c_mida_weighted = c_mida * alpha_mida
        # c_out = (c_mip_weighted * gamma + c_mida_weighted * (1-gamma)) / (alpha_out * gamma + alpha_mida * (1-gamma))
        
        # Simplified linear interpolation of color and alpha as per source description
        # Note: Source [S4] says "Interpolation is performed using opacity-weighted colors"
        # but does not give the exact formula. Standard weighted lerp:
        
        total_alpha = acc_alpha * (1.0 - gamma) + mip_alpha * gamma
        if total_alpha > 0:
            c_out = (acc_color * acc_alpha * (1.0 - gamma) + mip_color * mip_alpha * gamma) / total_alpha
            alpha_out = total_alpha
        else:
            c_out = acc_color
            alpha_out = 0.0
    else:
        c_out = acc_color
        alpha_out = acc_alpha
        
    return c_out, alpha_out
```

### Parameters
*   **$\gamma$ (Gamma):** Interpolation parameter. $\gamma = -1$ yields standard DVR. $\gamma = 0$ yields pure MIDA. $\gamma = 1$ yields MIP [S4].
*   **Transfer Function:** MIDA relies on simple linear mappings of intensity to opacity and color. Complex TFs are not required [S4].
*   **Intensity Range:** Users can specify a sub-range $[s_{min}, s_{max}]$ to filter noise or irrelevant low-intensity features [S2][S3, unofficial].
*   **Brightness Compensation:** To counteract dimming when the sub-range is large, brightness can be adjusted: $C_{comp} = C_{base} \times \frac{Range_{total}}{Range_{sub}} \times K$, where $K$ is a constant amplification factor (e.g., $K=2$) [S3, unofficial].
*   **Shading Thresholds:** For gradient-based shading, lower threshold $g_l = 0.125$ and upper threshold $g_h = 0.25$ are used to interpolate between unshaded and shaded colors [S4].

### Edge Cases and Pitfalls
*   **Early Ray Termination:** Standard early ray termination (stopping when $\alpha_{acc} \approx 1$) is **not possible** because opacity is no longer monotonically increasing due to modulation [S4].
*   **Max Value Termination:** Ray traversal cannot be terminated early even if the current maximum is the highest value in the dataset, because subsequent samples might modulate the accumulation differently [S4].
*   **Brightness Inconsistency:** Images may appear too dim if the MIDA sub-range is large (front structures are heavily modulated) or too bright if the sub-range is small [S3, unofficial]. Brightness compensation is recommended [S3, unofficial].
*   **Delta Clamping:** If data values are not normalized to $[0, 1]$, $\delta_i$ can exceed 1, causing $\beta_i < 0$, which inverts colors. $\delta_i$ must be clamped or scaled [S4, S5].
*   **Implementation Uncertainty:** Some implementations express uncertainty about the exact definition of "current maximum" and its impact on correctness [S1, unofficial].

## Operating characteristics
*   **Performance:** MIDA achieves real-time performance on modern GPUs. On an NVIDIA GeForce 8800 GTX, MIDA rendered a $256 \times 256 \times 224$ dataset at 9.1 fps (viewport 512x512), compared to 9.9 fps for DVR and 18.2 fps for MIP [S4].
*   **Complexity:** Time complexity is linear with respect to the number of samples per ray, same as DVR. Memory complexity is constant per ray (accumulators).
*   **Strengths:**
    *   No complex transfer function specification required [S4].
    *   Preserves spatial context and shading cues unlike MIP [S4].
    *   Prevents occlusion of high-intensity structures by foreground "fog" [S4].
    *   Smooth transition between DVR and MIP via $\gamma$ [S4].
*   **Limitations:**
    *   Cannot use early ray termination, leading to slightly slower performance than DVR/MIP [S4].
    *   Brightness can be inconsistent without compensation [S3, unofficial].
    *   Modulation may cause prominent features to still be obscured if the local maximum does not exceed the previous global maximum significantly [S3, unofficial].

## Background
MIDA was introduced by Bruckner et al. in 2009 [S4, S5]. It was developed to address the limitations of standard DVR (occlusion by fog) and MIP (loss of spatial context) by exploiting inherent data characteristics (high intensity = important) to modulate opacity accumulation [S2, S4]. It has been extended for multichannel data and applied in neurobiology research [S5].

## Sources
[S1] fargiolas/qvrc, https://github.com/fargiolas/qvrc, fetched 2026-10-26 (unofficial)
[S2] Context-Preserving Volumetric Data Set Exploration Using a 3D Painting Metaphor, https://doi.org/10.1007/978-3-642-33179-4_33, fetched 2026-10-27
[S3] Context-Preserving Volume Image Exploration Using A 3D Painting Metaphor, https://doi.org/10.32920/ryerson.14646888, fetched 2026-10-26 (unofficial)
[S4] Instant Volume Visualization using Maximum Intensity Difference Accumulation, https://doi.org/10.1111/j.1467-8659.2009.01474.x, fetched 2026-10-27
[S5] Fast Volumetric Data Exploration with Importance-Based Accumulated Transparency Modulation, https://doi.org/10.2312/vg/vg10/061-068, fetched 2026-10-26
