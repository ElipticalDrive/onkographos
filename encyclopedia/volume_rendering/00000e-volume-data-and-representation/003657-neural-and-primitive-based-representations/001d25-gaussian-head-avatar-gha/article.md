# Gaussian Head Avatar (GHA)
## Overview
Gaussian Head Avatar (GHA) is a neural primitive-based representation for dynamic head avatars that predicts 3D Gaussian parameters directly from facial expression and head pose inputs using multi-layer perceptrons (MLPs) and a deformation module [S1][S2, unofficial]. It distinguishes itself from methods like Gaussian Avatars, which optimize per-triangle Gaussians on a static FLAME mesh, by employing a continuous, learned deformation field that generalizes to unseen expressions without per-scene optimization [S1][S2, unofficial].
## How it works
**Inputs and Outputs:**
- Input: Facial expression coefficients (e.g., from a parametric face model like FLAME) and head pose parameters.
- Output: 3D Gaussian parameters (position, covariance, opacity) and color for splatting.

**Data Structures:**
- Two MLPs: One for geometry/deformation and one for color.
- Deformation Module: Maps base coordinates to deformed positions based on pose/expression.
- Gaussian Primitives: Defined by mean $\mathbf{\mu}$, covariance $\mathbf{\Sigma}$, opacity $\alpha$, and color $\mathbf{c}$.

**Steps:**
1. **Pose/Expression Encoding:** Input facial expression and head pose vectors are encoded.
2. **Deformation Prediction:** A deformation module (often an MLP) takes a base point (e.g., from a template mesh or canonical space) and the pose/expression features to predict the deformed 3D position $\mathbf{\mu}$ [S1][S2, unofficial].
3. **Color Prediction:** A separate color MLP takes the deformed position and viewing direction to predict color $\mathbf{c}$ [S1][S2, unofficial].
4. **Gaussian Parameterization:** Covariance $\mathbf{\Sigma}$ and opacity $\alpha$ are predicted or derived. The sources note the prediction of 3D Gaussian positions specifically, implying other parameters are either learned jointly or fixed via a prior [S1][S2, unofficial].
5. **Rendering:** The predicted Gaussians are rendered using 3D Gaussian Splatting techniques [S1][S2, unofficial].

**Formulas:**
- Deformed Position: $\mathbf{\mu} = f_{\text{def}}(\mathbf{x}_{\text{base}}, \mathbf{p}_{\text{pose}}, \mathbf{e}_{\text{expr}})$, where $f_{\text{def}}$ is the deformation MLP [S1][S2, unofficial].
- Color: $\mathbf{c} = f_{\text{color}}(\mathbf{\mu}, \mathbf{d})$, where $f_{\text{color}}$ is the color MLP and $\mathbf{d}$ is the view direction [S1][S2, unofficial].

**Parameters:**
- Number of MLP layers and hidden dimensions: Not given in the sources.
- Base geometry: Typically a FLAME mesh template, though GHA uses MLPs instead of mesh-constrained optimization [S1][S2, unofficial].

**Edge Cases and Pitfalls:**
- Generalization to extreme expressions: The MLP must be trained on diverse data to avoid artifacts [S1][S2, unofficial].
- Precision: Standard MLP spectral bias issues may affect high-frequency facial details [S1][S2, unofficial] (inherited from parent principles).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# GHA Rendering Step
def render_gha(pose, expression, view_direction, base_points):
    """
    pose: Head pose parameters
    expression: Facial expression coefficients
    view_direction: Camera view vector
    base_points: Canonical 3D points (e.g., from FLAME mesh)
    """
    # Deformation Module
    deformed_positions = deformation_mlp(base_points, pose, expression)
    
    # Color MLP
    colors = color_mlp(deformed_positions, view_direction)
    
    # Gaussian Parameters
    # Note: Covariance and opacity prediction details are not specified in sources
    # [G] Assuming standard 3DGS parameterization or learned via separate heads
    covariances = predict_covariance(deformed_positions) 
    opacities = predict_opacity(deformed_positions)
    
    # Splatting
    rendered_image = gaussian_splatting(deformed_positions, covariances, opacities, colors)
    return rendered_image
```
## Subtopics

### Gaussian Avatars
Uses a FLAME mesh and optimizes Gaussians constrained to triangles. Choose when mesh-based priors are preferred for stability [S1][S2, unofficial].

### Gaussian Avatar (SMPL)
Extends to full-body using SMPL. Choose for full-body avatars [S1][S2, unofficial].

### 3DGS-Avatar
Combines rigid and non-rigid SMPL transformations. Choose for full-body with complex motion [S1][S2, unofficial].

## Operating characteristics
- **Time Complexity**: Depends on MLP inference and Gaussian splatting. Generally faster than per-scene optimization methods like Gaussian Avatars since no per-frame optimization is needed [S1][S2, unofficial].
- **Memory Complexity**: Stores MLP weights and base mesh. Lower than methods storing large sets of optimized Gaussians per scene [S1][S2, unofficial].
- **Strengths**: Real-time rendering potential, generalization to new expressions without optimization [S1][S2, unofficial].
- **Limitations**: Quality may depend on training data diversity; potential artifacts in extreme poses [S1][S2, unofficial].
## Background
GHA was introduced as an alternative to mesh-constrained Gaussian optimization methods like Gaussian Avatars [S1][S2, unofficial]. It leverages the flexibility of neural networks to model dynamic deformations directly [S1][S2, unofficial].
## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
