# Shutter Miscalibration Correction

Also called: Camera Shutter Speed Miscalibration Correction, Multi-exposure training, Per-color-channel scaling, Shutter miscalibration compensation, Variable Exposure Training

## Overview
Shutter Miscalibration Correction is a training-time radiometric self-calibration technique that compensates for sensor-specific color channel mismatches when processing images with varying shutter speeds [S1, S2]. It addresses the problem where the relationship between exposure time and recorded intensity decays at different rates per color channel, a phenomenon distinct from simple black-level offsets [S2]. Unlike [Exposure Correction Strategy](8760-exposure-correction-strategy.md), which typically handles global exposure scaling, this method applies learned, per-channel scaling factors to correct for non-linear sensor response artifacts inherent in raw data capture [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Input:** Rendered color values $\hat{y}$ in linear RGB space and the corresponding shutter speed $t_i$ for each training image $i$.
*   **Output:** Corrected color values $\tilde{y}$ ready for loss computation or display.

**Data Structures:**
*   **Learned Parameters ($\beta$):** A set of scaling factors indexed by color channel $c \in \{R, G, B\}$ and unique shutter speed $t_i$. The structure is effectively a lookup table or embedding where $\beta^c_{t_i}$ is the scalar factor for channel $c$ at speed $t_i$.

**Steps:**
1.  For each training sample $i$ with shutter speed $t_i$:
2.  For each color channel $c$:
    a. Retrieve the rendered linear RGB value $\hat{y}^c_i$.
    b. Retrieve the learned scaling factor $\beta^c_{t_i}$.
    c. Compute the scaled value: $v^c_i = \hat{y}^c_i \cdot t_i \cdot \beta^c_{t_i}$.
    d. Clamp the result to the maximum representable value (1.0 for normalized linear space): $\tilde{y}^c_i = \min(v^c_i, 1)$.
3.  The vector $\tilde{y}_i$ is used for loss calculation against the reference image.

**Formulas:**
The final output color for the $c$-th channel of the $i$-th image is defined as [S1]:
$$ \tilde{y}^c_i = \min(\hat{y}^c_i \cdot t_i \cdot \beta^c_{t_i}, 1) $$

Where:
*   $\hat{y}^c_i$: Rendered color intensity for channel $c$ of image $i$ in linear space.
*   $t_i$: Shutter speed (exposure time) of image $i$.
*   $\beta^c_{t_i}$: Learned scaling factor for channel $c$ and shutter speed $t_i$.
*   $1$: The upper bound for normalized linear RGB values.

**Parameters:**
*   $\beta^c_{t_i}$: Initialized as a learnable parameter during optimization. Typical values depend on the sensor; for example, relative shifts between slow (1/1104s) and fast (1/181s) shutters can result in factors like (0.89, 0.93, 0.75) for Red, Green, and Blue respectively in the raw domain [S2].

**Edge Cases and Pitfalls:**
*   **Black Level:** The miscalibration is not caused by black level offsets. Shifting the black level higher or lower worsens the problem [S2]. Black level is an integer on the scale of 0 to 4095 for a 12-bit sensor [S2].
*   **Saturation:** The clamping operation handles saturation, but inputs should ideally avoid saturated pixels during training to ensure accurate gradient flow [S2].
*   **Cause:** The exact cause is not fully understood but may involve sensor temperature changes, imprecise shutter timing, or low-level sensor characteristics [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def shutter_miscalibration_correction(rendered_color, shutter_speed, beta_factors):
    """
    Applies per-channel scaling to correct shutter miscalibration.
    
    Args:
        rendered_color: Tensor of shape (H, W, 3) or (N, H, W, 3), linear RGB values.
        shutter_speed: Float or Tensor of shape (N,) or scalar, exposure time in seconds.
        beta_factors: Dictionary or Tensor mapping (channel, shutter_speed_bin) to scalar factors.
                      For simplicity, assume beta_factors is a tensor of shape (3, N_shutter_bins)
                      or a lookup function.
    
    Returns:
        corrected_color: Tensor of same shape as rendered_color, clamped to [0, 1].
    """
    # Ensure shutter_speed is broadcastable to (H, W, 1) or (N, 1, 1, 1)
    # In a batched setting, shutter_speed might vary per image.
    
    corrected_color = torch.zeros_like(rendered_color)
    
    for c in range(3): # R, G, B
        # Retrieve beta for this channel and the specific shutter speed of the image
        # Note: In practice, beta_factors might be interpolated if shutter speeds are continuous,
        # but the source defines it per unique shutter speed.
        beta = beta_factors[c, shutter_speed] 
        
        # Apply scaling: y_hat * t * beta
        scaled = rendered_color[..., c] * shutter_speed * beta
        
        # Clamp to 1.0
        clamped = torch.clamp(scaled, max=1.0)
        
        corrected_color[..., c] = clamped
        
    return corrected_color
```

## Subtopics

### Radiometric Self-Calibration
The parent method that optimizes exposure and sensor bias. Shutter Miscalibration Correction is a specific extension handling temporal exposure variations [Parent].

### Exposure Correction Strategy
A sibling method focusing on general exposure handling. Shutter Miscalibration Correction is distinct in its per-channel, shutter-speed-dependent scaling [Sibling].

## Operating characteristics
*   **Time Complexity:** Adds negligible overhead during training, consisting of a few multiplications and a clamp per pixel per channel.
*   **Memory Complexity:** Requires storing $\beta^c_{t_i}$ for each unique shutter speed and channel. If $N_t$ unique shutter speeds are used, memory is $O(3 \cdot N_t)$.
*   **Strengths:** Corrects significant color shifts (e.g., ~10-25% deviation) between slow and fast exposures that simple global scaling cannot fix [S2]. Enables training on datasets with highly variable exposure without manual pre-processing.
*   **Limitations:** Requires raw or linear RGB data. Does not correct for black-level miscalibration, which must be handled separately or ignored as it exacerbates the issue [S2].
*   **When to use:** When training on datasets with varying shutter speeds (e.g., night scenes, HDR synthesis) and observing color channel drift.
*   **When not to use:** If all images have identical shutter speeds, or if the sensor is perfectly calibrated (rare in practice).

## Background
The method was introduced in the context of "NeRF in the Dark" and "Lighting Every Darkness with 3DGS" to handle high dynamic range view synthesis from noisy raw images [S1, S2]. It addresses empirical observations from iPhone X sensors where shutter speed miscalibration causes non-uniform decay across color channels [S2]. The technique is part of a broader effort to make neural radiance fields robust to real-world sensor imperfections [S1, S2].

## Sources
[S1] Lighting Every Darkness with 3DGS: Fast Training and Real-Time Rendering for HDR View Synthesis, https://doi.org/10.52202/079017-2549, fetched 2026-10-26
[S2] NeRF in the Dark: High Dynamic Range View Synthesis from Noisy Raw Images, https://doi.org/10.1109/cvpr52688.2022.01571, fetched 2026-10-26
