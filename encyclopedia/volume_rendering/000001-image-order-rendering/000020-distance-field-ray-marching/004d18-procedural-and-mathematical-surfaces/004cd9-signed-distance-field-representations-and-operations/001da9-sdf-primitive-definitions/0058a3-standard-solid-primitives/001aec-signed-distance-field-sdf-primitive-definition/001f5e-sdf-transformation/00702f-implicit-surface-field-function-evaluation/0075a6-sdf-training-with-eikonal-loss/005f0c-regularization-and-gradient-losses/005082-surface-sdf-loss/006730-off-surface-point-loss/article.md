# Off-surface Point Loss

Also called: Off-surface Loss

## Overview
The off-surface point loss is a regularization term used in training Signed Distance Field (SDF) networks to enforce that the predicted distance values are positive and large for points located far outside the target geometry. It complements the surface SDF loss by penalizing the network for predicting negative or small positive values in the exterior region, thereby preventing the SDF from collapsing or becoming ambiguous in empty space. It is distinct from the [SDF Loss](17752-sdf-loss.md) which primarily targets points on or near the surface.

## How it works
The method operates in the canonical space of the target mesh (e.g., SMPL). It involves sampling points in the exterior volume and applying an exponential penalty based on the network's output.

**Inputs:**
- $f_{\sigma_f}(\mathbf{x})$: The SDF network function parameterized by $\sigma_f$, returning the signed distance at point $\mathbf{x}$.
- $\hat{\mathbf{X}}_{\text{off}}$: A set of sampled points in canonical space.
- $|P|$: The number of points in $\hat{\mathbf{X}}_{\text{off}}$.

**Outputs:**
- $\mathcal{L}_O$: The scalar off-surface point loss value.

**Data Structures:**
- $\hat{\mathbf{X}}_{\text{off}}$: A set of 3D coordinates. In the source implementation, $|P| = 1024$. Points are sampled such that their Euclidean distance to the canonical mesh surface is greater than 20cm [S1].

**Steps:**
1. Sample 1024 points $\hat{\mathbf{x}}$ in canonical space such that the distance from $\hat{\mathbf{x}}$ to the canonical mesh surface is $> 20$cm [S1]. Let this set be $\hat{\mathbf{X}}_{\text{off}}$.
2. For each point $\hat{\mathbf{x}} \in \hat{\mathbf{X}}_{\text{off}}$, evaluate the SDF network $f_{\sigma_f}(\hat{\mathbf{x}})$.
3. Compute the loss using the formula:
   $$ \mathcal{L}_O = \frac{1}{|P|} \sum_{\hat{\mathbf{x}} \in \hat{\mathbf{X}}_{\text{off}}} \exp \left( -100 \cdot f_{\sigma_f}(\hat{\mathbf{x}}) \right) $$
   where the coefficient $100$ (written as $1e^2$ in the source) scales the penalty [S1].

**Parameters:**
- Number of points $|P|$: 1024 [S1].
- Distance threshold for sampling: $> 20$cm [S1].
- Exponential coefficient: $100$ ($1e^2$) [S1].
- Weight $\lambda_O$: $100$ (used in total loss combination, though the formula itself contains the internal coefficient) [S1].

**Edge Cases and Pitfalls:**
- If the network predicts negative values for off-surface points, the exponent becomes positive, leading to an exponentially large loss, which can cause training instability if not balanced [S1].
- The threshold of 20cm is specific to the scale of the SMPL mesh; scaling the mesh or changing the domain requires re-evaluating this threshold [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def off_surface_point_loss(sdf_network, points_off, num_points=1024):
    """
    Computes the off-surface point loss.
    
    Args:
        sdf_network: Function f_sigma_f(x) returning SDF value.
        points_off: Array of shape (num_points, 3) containing points 
                    > 20cm from the mesh surface in canonical space.
        num_points: Number of points (default 1024).
        
    Returns:
        Scalar loss value.
    """
    # Evaluate SDF at all off-surface points
    # Shape: (num_points,)
    sdf_values = sdf_network(points_off)
    
    # Compute exponential penalty
    # Coefficient is 100 (1e2) as per source [S1]
    # exp(-100 * f(x))
    penalties = torch.exp(-100.0 * sdf_values)
    
    # Average over all points
    loss = torch.mean(penalties)
    
    return loss
```

## Subtopics

### Inside Point Loss
The inside point loss is a variant that regularizes points located inside the mesh. It samples 1024 points in canonical space that are inside the mesh and whose distance to the surface is greater than 1cm [S1]. The loss is defined as:
$$ \mathcal{L}_I = \frac{1}{|P|} \sum_{\hat{\mathbf{x}} \in \hat{\mathbf{X}}_{\text{in}}} \text{sigmoid} \left( 5000 \cdot f_{\sigma_f}(\hat{\mathbf{x}}) \right) $$
where $\hat{\mathbf{X}}_{\text{in}}$ is the set of inside points, and the coefficient is $5000$ ($5e^3$) [S1]. This encourages the SDF to be negative inside the volume.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of sampled points $|P|$. Each point requires one forward pass through the SDF network.
- **Memory Complexity:** Linear with respect to $|P|$ to store the sampled points and intermediate SDF values.
- **Strengths:** Effectively pushes the SDF values positive in the exterior, preventing "floating" artifacts or incorrect zero-level sets far from the surface.
- **Limitations:** The choice of sampling distance (20cm) and coefficient (100) is heuristic and tied to the specific mesh scale (SMPL) [S1]. It does not enforce the Eikonal equation (gradient magnitude = 1), which is handled by separate regularization [S1].

## Background
The off-surface point loss was introduced in the ARAH (Animatable Volume Rendering of Articulated Human SDFs) method [S1]. It is part of a set of regularization losses including Eikonal loss, inside point loss, and skinning loss, designed to train robust SDF representations for articulated human bodies [S1].

## Sources
[S1] ARAH: Animatable Volume Rendering of Articulated Human SDFs, https://doi.org/10.1007/978-3-031-19824-3_1, fetched 2026-10-26
