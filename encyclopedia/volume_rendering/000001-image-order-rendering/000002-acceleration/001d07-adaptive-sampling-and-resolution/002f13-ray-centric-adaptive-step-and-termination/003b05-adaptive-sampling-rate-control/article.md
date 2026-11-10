# Adaptive Sampling Rate Control

Also called: Viewport Adaptive Resolution, dt_scale control

## Overview
Adaptive Sampling Rate Control modulates the ray step size in volume rendering by mapping a user-defined sampling rate to an inverse scale factor (`dt_scale`) passed to the shader [S1, unofficial]. It differs from error-driven siblings by relying on a discrete, user-selected rate rather than a computed metric, offering a deterministic trade-off between performance and fidelity [S1, unofficial]. It builds on [Ray-Centric Adaptive Step and Termination](5938-ray-centric-adaptive-step-and-termination.md) by providing the specific parameterization for rate-based adaptation.

## How it works
**Inputs:**
- `rate_index`: Integer slider value $0 \dots 5$.
- `is_adaptive`: Boolean toggle for viewport resolution.

**Outputs:**
- `dt_scale`: Uniform float passed to the rendering shader.
- `max_resolution`: Viewport resolution limit.

**Data Structures:**
- `rate_map`: Array mapping indices to sampling rates.
  - Index 0: 0.25
  - Index 1: 0.5
  - Index 2: 1.0
  - Index 3: 2.0
  - Index 4: 4.0
  - Index 5: 8.0

**Steps:**
1. Retrieve the current `rate_index` from the UI slider.
2. Look up the corresponding sampling rate $R$ from `rate_map`.
3. Compute `dt_scale` as the inverse of the sampling rate:
   $$ \text{dt\_scale} = \frac{1}{R} $$
4. Pass `dt_scale` to the volume rendering shader as a uniform float. The shader uses this to modulate the ray step size $\Delta t$, effectively scaling the sampling density [S1, unofficial].
5. If `is_adaptive` is true, enable viewport adaptive resolution and set the maximum resolution limit via `viewport.setMaxResolution()` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Mapping defined in source [S1, unofficial]
RATE_MAP = [0.25, 0.5, 1.0, 2.0, 4.0, 8.0]

def compute_dt_scale(rate_index: int) -> float:
    """
    Computes the dt_scale parameter for the shader based on the UI slider index.
    """
    if rate_index < 0 or rate_index >= len(RATE_MAP):
        raise ValueError("Invalid rate index")
    
    sampling_rate = RATE_MAP[rate_index]
    # dt_scale is the inverse of the sampling rate [S1, unofficial]
    dt_scale = 1.0 / sampling_rate
    return dt_scale

def apply_adaptive_settings(rate_index: int, is_adaptive: bool):
    """
    Applies the sampling rate control and viewport resolution settings.
    """
    dt_scale = compute_dt_scale(rate_index)
    
    # Pass dt_scale to the shader uniform
    # shader.set_uniform("dt_scale", dt_scale)
    
    if is_adaptive:
        # Enable adaptive resolution and set max limit [S1, unofficial]
        # viewport.setAdaptiveResolution(True)
        # viewport.setMaxResolution(...)
        pass
```

**Edge Cases:**
- `rate_index` out of bounds: Must be clamped or rejected.
- Zero sampling rate: The map ensures $R \ge 0.25$, preventing division by zero.

## Subtopics

### Dense Sampling for Multidimensional Transfer Functions
A rendering strategy requiring dense sampling of voxel data to ensure smooth boundaries when using multidimensional transfer functions, as neighboring voxels may differ in additional dimensions [S2]. It sets apart by mandating high sample counts (3-6 per voxel) specifically for frozen high-quality renders to handle complex transfer function boundaries [S2]. Choose this when visual fidelity of multidimensional data is critical and interaction latency is not a concern [S2].

## Operating characteristics
- **Time Complexity:** Frame updates can take nearly a second for a moderately sized volume (256x256x128) with shading and shadows at 3-6 samples per voxel [S2].
- **Memory:** Not given in the sources.
- **Scaling:** Performance degrades linearly with sample rate and volume size [S2].
- **Strengths:** Ensures smooth boundaries for multidimensional transfer functions [S2].
- **Limitations:** High computational cost for interactive frame rates [S2].
- **When to use:** High-quality frozen renderings with multidimensional transfer functions [S2].
- **When not:** Interactive exploration where lower sample rates are acceptable during interaction [S2].

## Background
The concept of adaptive sampling rates in the context of multidimensional transfer functions was discussed in relation to interactive volume rendering, noting the need to lower sample rates during interaction and rerender at higher rates after actions are completed [S2]. The specific implementation of `dt_scale` control via a slider is documented in a WebGL volume rendering repository [S1, unofficial].

## Sources
[S1] termijn/webgl-volumerendering, https://github.com/termijn/webgl-volumerendering, fetched 2026-10-27 (unofficial)
[S2] Multidimensional Transfer Functions for Interactive Volume ..., https://www.academia.edu/5403340/Multidimensional_Transfer_Functions_for_Interactive_Volume_Rendering, fetched 2026-11-01
