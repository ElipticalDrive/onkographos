# Activation Function Selection for Gaussian Parameters

Also called: Choice of activation function applied to Gaussian intensity parameters, Gaussian Parameter Activation Functions, Intensity Activation Functions, Sigmoid for Intensity, Exponential for Scaling, Softmax vs Sigmoid Activation Ablation

## Overview
This method defines the specific non-linear transformations applied to raw neural network outputs (pre-activations) to produce valid Gaussian intensity and scaling parameters. It solves the problem of ensuring physical validity (non-negative scales, bounded intensities) while optimizing reconstruction quality. It is distinguished from sibling methods by its specific ablation results comparing sigmoid, tanh, exponential, and softmax activations, highlighting that sigmoid is superior for intensity and exponential is slightly preferred for scaling.

## How it works
**Inputs and Outputs:**
*   **Input:** Raw pre-activation values $z$ from the network for intensity and scaling parameters.
*   **Output:** Validated intensity values $I \in [0, 1]$ and scaling values $s > 0$.

**Data Structures:**
*   The raw outputs are typically stored as floating-point tensors. No specific bit-packing is mandated by the activation choice itself, but the output values must be interpreted as continuous real numbers.

**Steps:**
1.  **Intensity Activation:** Apply the sigmoid function to the raw intensity pre-activation $z_I$.
    $$ I = \sigma(z_I) = \frac{1}{1 + e^{-z_I}} $$
    This ensures $I \in (0, 1)$, suitable for opacity/intensity representation.
2.  **Scaling Activation:** Apply the exponential function to the raw scaling pre-activation $z_S$.
    $$ s = \exp(z_S) $$
    This ensures $s > 0$, which is required for valid Gaussian covariance scales.

**Parameters:**
*   None specific to the activation function itself.

**Edge Cases and Pitfalls:**
*   **Tanh for Intensity:** Using the hyperbolic tangent (tanh) activation for intensity parameters leads to significantly worse reconstruction performance compared to sigmoid [S1]. Tanh outputs values in $(-1, 1)$, which may require additional shifting/scaling and introduces optimization difficulties for bounded positive quantities like intensity/opacity.
*   **Softmax for Scaling/Intensity:** Softmax forces the network to softly select a specific resolution or distribute values across a set, resulting in higher Chamfer Distance (0.74) compared to Sigmoid (0.65) in surface reconstruction contexts [S2].
*   **Numerical Stability:** While not explicitly detailed for the activation functions themselves, large learning rates in conjunction with exponential scaling can lead to numerical instability (NaN values) during optimization [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def activate_intensity(z_I: float) -> float:
    """
    Applies sigmoid activation to intensity pre-activation.
    Returns value in (0, 1).
    """
    # Clip to prevent overflow in exp
    z_clipped = max(min(z_I, 500), -500)
    return 1.0 / (1.0 + math.exp(-z_clipped))

def activate_scaling(z_S: float) -> float:
    """
    Applies exponential activation to scaling pre-activation.
    Returns value in (0, inf).
    """
    # Clip to prevent overflow
    z_clipped = max(min(z_S, 50), -50)
    return math.exp(z_clipped)
```

## Subtopics

### Intensity Activation
Uses sigmoid. Chosen because tanh yields significantly worse reconstruction quality [S1].

### Scaling Activation
Uses exponential. Chosen because it performs slightly better than sigmoid, though both are comparable [S1].

### Softmax Activation
Avoided for general parameter activation as it forces soft selection of resolution, increasing Chamfer Distance [S2].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per parameter. Sigmoid and exponential are standard mathematical operations with negligible overhead.
*   **Memory Complexity:** $O(1)$ additional memory per parameter.
*   **Measured Performance:**
    *   **Intensity:** Sigmoid outperforms tanh significantly [S1].
    *   **Scaling:** Exponential performs slightly better than sigmoid, but the difference is small [S1].
    *   **Surface Reconstruction:** Sigmoid yields a Chamfer Distance of 0.65, while Softmax yields 0.74 [S2].
*   **Strengths:** Ensures physical validity (positivity for scales, boundedness for intensity).
*   **Limitations:** Tanh is a poor choice for intensity. Softmax is suboptimal for single-value parameter regression.
*   **When to use:** Standard practice for 3D Gaussian Splatting and related explicit neural feature volume methods.
*   **When not to use:** Do not use tanh for intensity or softmax for single-value scaling/intensity unless specific multi-class or multi-resolution selection is required.

## Background
This method originates from ablation studies in 3D Gaussian Splatting variants for CT reconstruction and neural surface reconstruction. The specific comparison of sigmoid vs. tanh for intensity and sigmoid vs. exponential for scaling is detailed in "3DGR-CT: Sparse-view CT reconstruction with a 3D Gaussian representation" [S1]. The comparison of sigmoid vs. softmax in the context of hash-encoded neural surface reconstruction is detailed in "Spatially-Adaptive Hash Encodings for Neural Surface Reconstruction" [S2].

## Sources
[S1] 3DGR-CT: Sparse-view CT reconstruction with a 3D Gaussian representation, https://doi.org/10.1016/j.media.2025.103585, fetched 2026-10-26
[S2] Spatially-Adaptive Hash Encodings for Neural Surface Reconstruction, https://doi.org/10.1109/wacv61041.2025.00293, fetched 2026-10-26
