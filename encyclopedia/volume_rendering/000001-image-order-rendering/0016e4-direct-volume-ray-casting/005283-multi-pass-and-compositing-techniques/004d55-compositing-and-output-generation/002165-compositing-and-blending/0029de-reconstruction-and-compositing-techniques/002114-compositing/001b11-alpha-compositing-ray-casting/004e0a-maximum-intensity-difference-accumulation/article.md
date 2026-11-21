# Maximum Intensity Difference Accumulation

Also called: MIDA, Monotonically Increasing Density Accumulation

## Overview
Maximum Intensity Difference Accumulation (MIDA) is a volume rendering compositing technique that modifies standard Direct Volume Rendering (DVR) to prevent high-intensity structures from being occluded by irrelevant foreground material [S4, unofficial]. Unlike standard DVR, where accumulated opacity grows monotonically and causes "fog" that hides distant features, MIDA modulates previously accumulated color and opacity based on the difference between the current sample's intensity and the current maximum intensity along the ray [S4, unofficial]. This allows MIDA to combine the spatial context and shading compatibility of DVR with the parameterless, high-visibility nature of Maximum Intensity Projection (MIP) [S5].

## How it works
### Inputs and Outputs
**Inputs:**
*   A scalar volumetric dataset $V$.
*   A viewing ray defined by origin and direction [S4, unofficial].
*   A transfer function mapping scalar values to color $c(s)$ and opacity $\alpha(s)$ [S4, unofficial].
*   A modulation parameter $\gamma \in [-1, 1]$ to control the transition between DVR, MIDA, and MIP [S5].
*   (Optional) A user-defined intensity range $[s_{min}, s_{max}]$ to filter features [S4, unofficial].
*   (Optional) Brightness compensation parameters $K$, $Range_{total}$, $Range_{sub}$ [S4, unofficial].

**Outputs:**
*   Final accumulated color $C_{out}$ and opacity $\alpha_{out}$ for the pixel [S5].

### Data Structures
*   **Accumulated Color ($c_{acc}$):** RGB vector, initialized to $(0, 0, 0)$ [S5].
*   **Accumulated Opacity ($\alpha_{acc}$):** Scalar, initialized to $0.0$ [S5].
*   **Current Maximum Intensity ($s_{max}$):** Scalar, initialized to $-\infty$ (or minimum possible data value) [S5].
*   **Sample Color ($c_{src}$):** RGB vector, derived from transfer function $c(f(P_i))$ [S5].
*   **Sample Opacity ($\alpha_{src}$):** Scalar, derived from transfer function $\alpha(f(P_i))$ [S5].
*   **Sample Intensity ($s_i$):** Scalar, $f(P_i)$ [S5].

### Steps
1.  **Initialization:** Initialize $c_{acc} = (0, 0, 0)$, $\alpha_{acc} = 0.0$, and $s_{max} = -\infty$ [S5].
2.  **Ray Traversal:** Traverse the volume from front to back (eye to object) [S5]. For each sample $i$ at position $P_i$:
    a.  **Sample Data:** Retrieve scalar value $s_i = f(P_i)$ [S5].
    b.  **Classify:** Compute sample color $c_{src} = c(s_i)$ and sample opacity $\alpha_{src} = \alpha(s_i)$ using the transfer function [S5].
    c.  **Update Maximum:** If $s_i > s_{max}$, calculate the delta $\delta_i = s_i - s_{max}$ and update $s_{max} = s_i$. Otherwise, $\delta_i = 0$ [S5].
    d.  **Calculate Modulation Factor ($\beta_i$):**
        *   If $\gamma \ge 0$ (MIDA to MIP transition): $\beta_i = 1 - \delta_i$ [S5].
        *   If $\gamma < 0$ (MIDA to DVR transition): $\beta_i = 1 - \delta_i (1 + \gamma)$ [S5].
        *   *Note:* $\delta_i$ is clamped to $[0, 1]$ if data is normalized to $[0, 1]$. If data is not normalized, $\delta_i$ must be scaled or clamped to prevent $\beta_i < 0$ [S4, unofficial][S5].
    e.  **Modulate and Composite:** Update accumulated color and opacity in a single step using the modulation factor [S5]:
        $$c_{acc} \leftarrow \beta_i \cdot c_{acc} + (1 - \beta_i \cdot \alpha_{acc}) \cdot \alpha_{src} \cdot c_{src}$$
        $$\alpha_{acc} \leftarrow \beta_i \cdot \alpha_{acc} + (1 - \beta_i \cdot \alpha_{acc}) \cdot \alpha_{src}$$
3.  **MIP Transition (Image Space):** If $\gamma > 0$, after the ray traversal is complete, linearly interpolate between the accumulated MIDA result $(c_{acc}, \alpha_{acc})$ and the MIP result $(c_{mip}, \alpha_{mip})$ using $\gamma$ as the weight. Interpolation is performed using opacity-weighted colors [S5].
    $$c_{out} = \text{lerp\_weighted}(c_{acc}, c_{mip}, \gamma)$$
    $$\alpha_{out} = \text{lerp}(\alpha_{acc}, \alpha_{mip}, \gamma)$$
    If $\gamma \le 0$, $c_{out} = c_{acc}$ and $\alpha_{out} = \alpha_{acc}$ [S5].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_mida_ray(ray_origin, ray_direction, volume, transfer_function, gamma):
    """
    Renders a single ray using MIDA compositing.
    
    Args:
        ray_origin: tuple of 3 floats
        ray_direction: tuple of 3 floats
        volume: object with method sample(pos) -> float
        transfer_function: function mapping scalar -> (color: tuple of 3 floats, alpha: float)
        gamma: float in [-1, 1]. -1=DVR, 0=MIDA, 1=MIP.
        
    Returns:
        (color: tuple of 3 floats, alpha: float)
    """
    # Initialization
    acc_color = (0.0, 0.0, 0.0)
    acc_alpha = 0.0
    max_intensity = float('-inf')
    
    # MIP tracking
    mip_color = (0.0, 0.0, 0.0)
    mip_alpha = 0.0
    mip_max_intensity = float('-inf')
    
    # Ray traversal parameters
    step_size = 0.01 
    num_samples = 100 
    
    for i in range(num_samples):
        # Sample position
        pos = (ray_origin[0] + ray_direction[0] * (i * step_size),
               ray_origin[1] + ray_direction[1] * (i * step_size),
               ray_origin[2] + ray_direction[2] * (i * step_size))
        
        # Get scalar value
        s_i = volume.sample(pos)
        
        # Classify
        c_src, a_src = transfer_function(s_i)
        
        # Update Maximum Intensity
        delta = 0.0
        if s_i > max_intensity:
            delta = s_i - max_intensity
            max_intensity = s_i
            
        # Clamp delta to [0, 1] assuming normalized data [0,1]
        delta = max(0.0, min(1.0, delta))
        
        # Calculate Modulation Factor beta
        if gamma < 0:
            beta = 1.0 - delta * (1.0 + gamma)
        else:
            beta = 1.0 - delta
            
        # Modulate and Composite in one step per [S5]
        # c_acc = beta * c_acc + (1 - beta * alpha_acc) * a_src * c_src
        # alpha_acc = beta * alpha_acc + (1 - beta * alpha_acc) * a_src
        acc_color = tuple(beta * acc_color[k] + (1.0 - beta * acc_alpha) * a_src * c_src[k] for k in range(3))
        acc_alpha = beta * acc_alpha + (1.0 - beta * acc_alpha) * a_src
        
        # Track MIP result for gamma > 0 transition
        if s_i > mip_max_intensity:
            mip_max_intensity = s_i
            mip_color, mip_alpha = transfer_function(s_i)
            
    # Handle gamma > 0: Image space interpolation to MIP
    if gamma > 0:
        # Interpolate using opacity-weighted colors
        total_alpha = acc_alpha * (1.0 - gamma) + mip_alpha * gamma
        if total_alpha > 0:
            c_out = tuple((acc_color[k] * acc_alpha * (1.0 - gamma) + mip_color[k] * mip_alpha * gamma) / total_alpha for k in range(3))
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
*   **$\gamma$ (Gamma):** Interpolation parameter. $\gamma = -1$ yields standard DVR. $\gamma = 0$ yields pure MIDA. $\gamma = 1$ yields MIP [S5].
*   **Transfer Function:** MIDA relies on simple linear mappings of intensity to opacity and color. Complex TFs are not required [S4, unofficial].
*   **Intensity Range:** Users can specify a sub-range $[s_{min}, s_{max}]$ to filter noise or irrelevant low-intensity features [S4, unofficial].
*   **Brightness Compensation:** To counteract dimming when the sub-range is large, brightness can be adjusted: $C_{comp} = C_{base} \times \frac{Range_{total}}{Range_{sub}} \times K$, where $K$ is a constant amplification factor (e.g., $K=2$) [S4, unofficial].
*   **Shading Thresholds:** For gradient-based shading, lower threshold $g_l = 0.125$ and upper threshold $g_h = 0.25$ are used to interpolate between unshaded and shaded colors [S5].

### Edge Cases and Pitfalls
*   **Early Ray Termination:** Standard early ray termination (stopping when $\alpha_{acc} \approx 1$) is **not possible** because opacity is no longer monotonically increasing due to modulation [S5].
*   **Max Value Termination:** Ray traversal cannot be terminated early even if the current maximum is the highest value in the dataset, because subsequent samples might modulate the accumulation differently [S5].
*   **Brightness Inconsistency:** Images may appear too dim if the MIDA sub-range is large (front structures are heavily modulated) or too bright if the sub-range is small [S4, unofficial]. Brightness compensation is recommended [S4, unofficial].
*   **Delta Clamping:** If data values are not normalized to $[0, 1]$, $\delta_i$ can exceed 1, causing $\beta_i < 0$, which inverts colors. $\delta_i$ must be clamped or scaled [S4, unofficial][S5].
*   **Implementation Uncertainty:** Some implementations express uncertainty about the exact definition of "current maximum" and its impact on correctness [S1, unofficial].

## Operating characteristics
*   **Performance:** MIDA achieves real-time performance on modern GPUs. On an NVIDIA GeForce 8800 GTX, MIDA rendered a $256 \times 256 \times 224$ dataset at 9.1 fps (viewport 512x512), compared to 9.9 fps for DVR and 18.2 fps for MIP [S5].
*   **Complexity:** Time complexity is linear with respect to the number of samples per ray, same as DVR. Memory complexity is constant per ray (accumulators).
*   **Strengths:**
  - No complex transfer function specification required [S4, unofficial].
  - Preserves spatial context and shading cues unlike MIP [S5].
  - Prevents occlusion of high-intensity structures by foreground "fog" [S4, unofficial].
  - Smooth transition between DVR and MIP via $\gamma$ [S5].
*   **Limitations:**
  - Cannot use early ray termination, leading to slightly slower performance than DVR/MIP [S5].
  - Brightness can be inconsistent without compensation [S4, unofficial].
  - Modulation may cause prominent features to still be obscured if the local maximum does not exceed the previous global maximum significantly [S4, unofficial].

## Background
MIDA was introduced by Bruckner et al. in 2009 [S5]. It was developed to address the limitations of standard DVR (occlusion by fog) and MIP (loss of spatial context) by exploiting inherent data characteristics (high intensity = important) to modulate opacity accumulation [S4, unofficial]. It has been extended for multichannel data and applied in neurobiology research [S5].

## Sources
[S1] Enhanced perception in volume visualization, https://doi.org/10.5821/dissertation-2117-94929, fetched 2026-10-26 (unofficial)
[S2] fargiolas/qvrc, https://github.com/fargiolas/qvrc, fetched 2026-10-26 (unofficial)
[S3] Context-Preserving Volumetric Data Set Exploration Using a 3D Painting Metaphor, https://doi.org/10.1007/978-3-642-33179-4_33, fetched 2026-10-27
[S4] Context-Preserving Volume Image Exploration Using A 3D Painting Metaphor, https://doi.org/10.32920/ryerson.14646888, fetched 2026-10-26 (unofficial)
[S5] Instant Volume Visualization using Maximum Intensity Difference Accumulation, https://doi.org/10.1111/j.1467-8659.2009.01474.x, fetched 2026-10-27
[S6] Fast Volumetric Data Exploration with Importance-Based Accumulated Transparency Modulation, https://doi.org/10.2312/vg/vg10/061-068, fetched 2026-10-26
