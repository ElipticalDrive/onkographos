# Alpha-to-Extinction Conversion

Also called: Alpha to Extinction, Extinction Coefficient Computation from Alpha, convertToExtinctionCoeff

## Overview
Alpha-to-Extinction Conversion transforms opacity (alpha) values into raw extinction coefficients, resolving the incompatibility between alpha values and the extinction-based rendering pipeline. Unlike [Alpha Computation with Alpha Mask](10549-alpha-computation-with-alpha-mask.md), which handles masking, this method specifically bridges the gap between transfer-function-derived alpha and the physical scattering model. It enables the combination of physically calculated cloud extinction (from CIWC/CLWC) with transfer-function alpha values within a single renderer.

## How it works
The method converts scalar alpha values into extinction coefficients ($\tau$) to allow consistent accumulation during ray casting. The renderer uses extinction coefficients instead of simple alpha values to combine physically calculated ice and water cloud extinction with transfer function provided alpha values [S1].

**Inputs:**
- $\alpha$: The alpha (opacity) value derived from a transfer function or other source.
- $d$: The step size (distance) of the current ray segment.
- Mode: `Visual` or `Physical`.

**Outputs:**
- $\tau$: The raw extinction coefficient.

**Data Structures:**
- The conversion is implemented as a helper function `convertToExtinctionCoeff()` within `volume_sample_cloud_utils.glsl` [S1].

**Steps:**
1. Determine the conversion mode based on shader defines or configuration.
2. If `Visual` mode is active:
   - Apply a parameterized conversion to achieve the same visual look as if the algorithm were based directly on alpha values.
   - Use user-provided parameters to control the perceived optical thickness [S1].
3. If `Physical` mode is active:
   - Invert Beer's Law to derive the extinction coefficient.
   - Beer's Law states that the amount of light reaching the observer is $1 - e^{-\tau \cdot d}$. This value is treated as the alpha value.
   - Solve for $\tau$:
     $$ \alpha = 1 - e^{-\tau \cdot d} $$
     $$ e^{-\tau \cdot d} = 1 - \alpha $$
     $$ -\tau \cdot d = \ln(1 - \alpha) $$
     $$ \tau = -\frac{\ln(1 - \alpha)}{d} $$
   - Where $\ln$ is the natural logarithm [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def convert_to_extinction_coeff(alpha, d, mode, visual_params=None):
    """
    Convert alpha to extinction coefficient.
    
    Args:
        alpha: Float, opacity value in [0, 1).
        d: Float, step size (distance) of the ray segment.
        mode: String, 'Visual' or 'Physical'.
        visual_params: Dict, parameters for visual mode (e.g., thickness_factor).
        
    Returns:
        Float, extinction coefficient tau.
    """
    # Guard against division by zero or invalid alpha
    if d <= 0:
        return 0.0
    
    # Clamp alpha to prevent log(0) or negative log arguments
    # Alpha of 1.0 implies infinite extinction, so clamp slightly below 1.0
    alpha_clamped = min(alpha, 0.999999)
    
    if mode == 'Physical':
        # Invert Beer's Law: alpha = 1 - exp(-tau * d)
        # tau = -ln(1 - alpha) / d
        tau = -math.log(1.0 - alpha_clamped) / d
        return tau
    
    elif mode == 'Visual':
        # Visual conversion: achieve same visual look as direct alpha
        # Provides parameters to control perceived optical thickness.
        # Exact formula depends on visual_params, but generally scales alpha
        # to mimic extinction behavior without strict physical inversion.
        # [G] Since the source does not provide the exact visual formula,
        # we assume a linear or scaled mapping controlled by parameters.
        # A common approach is tau = alpha / d, but adjusted by thickness_factor.
        thickness_factor = visual_params.get('thickness_factor', 1.0) if visual_params else 1.0
        tau = (alpha_clamped * thickness_factor) / d
        return tau
    
    else:
        raise ValueError("Unknown conversion mode")
```

**Edge Cases:**
- **Alpha = 1.0:** $\ln(1 - 1) = \ln(0)$ is undefined. Clamp alpha to a value slightly less than 1.0 (e.g., 0.999999) to avoid numerical instability [G].
- **Step Size = 0:** Division by zero. Return 0 or handle as degenerate case [G].

## Subtopics

### Extinction Coefficient Calculation from Physical Properties
Calculation of extinction coefficients directly from scalar values for Cloud Ice Water Content (CIWC) and Cloud Liquid Water Content (CLWC). This allows physically based extinction to be combined with transfer-function alpha.
- **CLWC Calculation:** Performed in `calculateLwcExtCoeff()` based on Hu and Stamnes (1993) [S1].
- **CIWC Calculation:** Performed in `calculateIwcExtCoeff()` based on Fu (1996) [S1].
- **History:** Hu and Stamnes (1993) described the method for water clouds: "An accurate parameterization of the radiative properties of water clouds suitable for use in climate models." Fu (1996) described the method for cirrus clouds: "An accurate parameterization of the solar radiative properties of cirrus clouds for climate models." [S1]

## Operating characteristics
- **Time Complexity:** O(1) per sample, as it involves a single logarithmic or arithmetic operation.
- **Memory Complexity:** O(1), no additional data structures required.
- **Strengths:** Enables hybrid rendering where physically derived cloud extinction (CIWC/CLWC) can be combined with artist-defined transfer function alpha values.
- **Limitations:** Visual mode requires tuning parameters to match expected appearance. Physical mode assumes Beer's Law holds strictly for the medium.
- **When to use:** When using a renderer that relies on extinction coefficients for accumulation but needs to incorporate non-physical alpha values from transfer functions.

## Background
The method originates from the Met3D direct volume rendering implementation, which uses extinction coefficients instead of simple alpha values to allow combining physically calculated ice and water cloud extinction with transfer function provided alpha values [S1]. The physical conversion inverts Beer's Law, arguing that applying Beer's law using an extinction coefficient yields the amount of light still able to reach the observer, which can be treated as an alpha value [S1].

## Sources
[S1] Direct-volume raycasting, https://met3d.readthedocs.io/en/latest/05_developer_manual/03_actors/algorithms/direct_volume_rendering.html, fetched 2026-10-27
