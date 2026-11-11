# Predicted Normal Loss
Also called: Predicted Normal Consistency Loss

## Overview
Predicted Normal Loss is a regularization term in differentiable volume rendering that enforces geometric consistency by aligning explicitly predicted surface normals with normals derived from the scene's density field [S2]. It addresses normal estimation ambiguity in radiance fields by ensuring the learned geometry and appearance features describe the same surface structure [S2]. Unlike sibling methods that enforce smoothness or ray alignment, this loss couples the explicit normal prediction head to the implicit density gradient [S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   $\mathbf{n}_k^{\text{pred}}$: The predicted normal vector at sample point $k$, output by the network's normal prediction head [S2].
    *   $\sigma(\mathbf{x})$: The scalar density field evaluated at position $\mathbf{x}$ [S2].
    *   $w_k$: The accumulated transmittance weight for sample $k$ along the ray [S2].
    *   $C_{\text{normals}}$: A scalar hyperparameter controlling the loss weight [S2].
*   **Outputs:**
    *   $\mathcal{L}_{\text{normals}}$: A scalar loss value [S2].

**Data Structures:**
*   The density field $\sigma$ is typically represented as a neural network (e.g., MLP or Hash Grid) mapping 3D coordinates to a scalar [G].
*   Predicted normals $\mathbf{n}^{\text{pred}}$ are vectors of shape $(3)$ per sample [G].

**Steps:**
1.  **Compute Derived Normal:** Calculate the negative gradient of the density field with respect to position $\mathbf{x}_{\text{pos}}_k$ at the sample location. Normalize this gradient to obtain the derived normal $\mathbf{n}_k^{\text{derived}}$ [S2]:
    $$
    \mathbf{n}_k^{\text{derived}} = -\frac{\nabla \sigma(\mathbf{x}_{\text{pos}}_k)}{\| \nabla \sigma(\mathbf{x}_{\text{pos}}_k) \|}
    $$
2.  **Compute Weighted L2 Error:** Calculate the squared Euclidean distance between the predicted normal and the derived normal, weighted by the rendering weight $w_k$ [S2]:
    $$
    \mathcal{L}_{\text{normals}} = C_{\text{normals}} \sum_{k} w_k \| \mathbf{n}_k^{\text{pred}} - \mathbf{n}_k^{\text{derived}} \|^2
    $$
3.  **Accumulate:** Sum the contributions over all samples $k$ along all rays in the batch [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def predicted_normal_loss(
    predicted_normals: list,      # List of (x, y, z) tuples
    density_field: callable,      # Function sigma(x, y, z)
    sample_positions: list,       # List of (x, y, z) tuples
    render_weights: list,         # List of floats
    loss_weight: float            # C_normals
) -> float:
    """
    Computes the Predicted Normal Loss.
    """
    total_loss = 0.0
    for i in range(len(sample_positions)):
        pos = sample_positions[i]
        # 1. Compute the gradient of the density field at sample positions
        grad = gradient(density_field, pos)  # Standard automatic differentiation [G]
        
        # 2. Normalize the gradient to get derived normals
        norm = math.sqrt(sum(g**2 for g in grad))
        if norm < 1e-6:
            norm = 1e-6
        derived_normal = tuple(-g / norm for g in grad)
        
        # 3. Compute L2 distance between predicted and derived normals
        pred = predicted_normals[i]
        diff = tuple(p - d for p, d in zip(pred, derived_normal))
        l2_error = sum(d**2 for d in diff)
        
        # 4. Weight by render weights and sum
        total_loss += render_weights[i] * l2_error
        
    # 5. Apply global loss weight
    return loss_weight * total_loss
```

**Parameters:**
*   `pred_normal_loss_mult` (or $C_{\text{normals}}$): Multiplier for the loss. Typical values vary by dataset. For Open Illumination, $C_{\text{normals}} = 1.0$. For TensoIR-synthetic, it linearly interpolates from $0.0001$ to $1.0$ between iterations $20,000$ and $40,000$ [S2].

**Edge Cases and Pitfalls:**
*   **Zero Gradient:** In regions where $\nabla \sigma(\mathbf{x}) \approx 0$ (e.g., inside empty space or uniform density), the derived normal is undefined. A small epsilon must be added to the norm denominator to prevent NaN gradients [G].
*   **Non-normalized Predictions:** If the network does not explicitly normalize its normal output, the L2 loss may penalize magnitude differences rather than just orientation. It is recommended to normalize $\mathbf{n}^{\text{pred}}$ before computing the loss [G].

## Subtopics

### Density Gradient Normal Loss
This is the specific implementation of Predicted Normal Loss where the "derived" normal is strictly defined as the normalized negative gradient of the density field $\sigma$ [S2]. It is chosen when the scene representation relies on a continuous density field (like NeRF or NGP) and explicit normal prediction heads are used to disambiguate surface orientation [G].

## Operating characteristics
**Time and Memory Complexity:**
*   **Time:** Dominated by the forward pass of the density network and the backward pass for the gradient computation $\nabla \sigma$ [G].
*   **Memory:** Requires storing the computational graph for the density field to compute $\nabla \sigma$ [G].

**Strengths and Limitations:**
*   **Strengths:** Enforces strong geometric consistency between the appearance model (predicted normals) and the geometry model (density) [S2].
*   **Limitations:** Relies on the differentiability and smoothness of the density field [G].

**When to Use:**
*   Use when optimizing radiance fields with explicit normal prediction heads (e.g., in inverse rendering or high-fidelity novel view synthesis) [S2].

## Background
The Predicted Normal Loss was introduced in the context of differentiable volume rendering to improve the geometric quality of learned scenes [G]. It supervises predicted normals using rendered normals to ensure consistency between the two [S1, unofficial]. The specific formulation using the density gradient is detailed in works such as *Flash Cache: Reducing Bias in Radiance Cache Based Inverse Rendering* [S2] and is referenced in implementations like *GANerf* [S1, unofficial]. It is part of a broader family of geometric regularization losses including Curvature Loss and Normal Ray Alignment Loss [Parent].

## Sources
[S1] barbararoessle/ganerf, https://github.com/barbararoessle/ganerf, fetched 2026-10-27 (unofficial)
[S2] Flash Cache: Reducing Bias in Radiance Cache Based Inverse Rendering, https://doi.org/10.1007/978-3-031-73390-1_2, fetched 2026-10-27
