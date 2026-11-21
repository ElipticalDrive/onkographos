# SDF Smoothness Loss

Also called: Gradient Penalty Loss, SDF Gradient Regularization, Smooth Loss (Ls)

## Overview
SDF Smoothness Loss regularizes the reconstructed surface by penalizing the difference between the spatial gradients of the Signed Distance Function (SDF) at adjacent sampled points, thereby encouraging geometric smoothness and reducing high-frequency artifacts in the predicted field [S1]. It differs from the Eikonal loss, which enforces the gradient magnitude to be unity, by instead enforcing consistency between local gradient directions and magnitudes across small spatial perturbations [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of sampled points $\mathcal{X}$, the SDF network $d_\Omega(\cdot)$, and a small perturbation vector $\varepsilon$.
*   **Output:** A scalar loss value $\mathcal{L}_s$.

**Data Structures:**
*   $\mathcal{X}$: A set of $N_{sm}$ 3D points $\mathbf{p}$.
*   $\varepsilon$: A small perturbation vector (magnitude not specified in [S1]).

**Steps:**
1.  For each point $\mathbf{p} \in \mathcal{X}$, compute the SDF value $s = d_\Omega(\mathbf{p})$.
2.  Compute the spatial gradient of the SDF at $\mathbf{p}$: $\nabla d_\Omega(\mathbf{p})$. This is typically obtained via automatic differentiation of the network output with respect to the input coordinates.
3.  Compute the perturbed point $\mathbf{p}' = \mathbf{p} + \varepsilon$.
4.  Compute the SDF value $s' = d_\Omega(\mathbf{p}')$.
5.  Compute the spatial gradient of the SDF at $\mathbf{p}'$: $\nabla d_\Omega(\mathbf{p} + \varepsilon)$.
6.  Compute the Euclidean distance between the two gradients.
7.  Average these distances over all points in $\mathcal{X}$.

**Formula:**
The smooth loss $\mathcal{L}_s$ is defined as [S1]:

$$
\mathcal{L}_s = \frac{1}{N_{sm}} \sum_{\mathbf{p} \in \mathcal{X}} \left\| \nabla d_\Omega(\mathbf{p}) - \nabla d_\Omega(\mathbf{p} + \varepsilon) \right\|_2
$$

Where:
*   $N_{sm}$ is the number of points used to compute the smooth loss.
*   $\mathbf{p}$ is a query point in the set $\mathcal{X}$.
*   $d_\Omega(\mathbf{p})$ is the SDF output produced by the geometry MLP for point $\mathbf{p}$.
*   $\varepsilon$ is a small perturbation applied to the point.
*   $\nabla d_\Omega(\cdot)$ denotes the gradient of the SDF with respect to the spatial coordinates.

**Parameters:**
*   $\varepsilon$: Small perturbation. Value not given in [S1].
*   $N_{sm}$: Number of points. Not given in [S1].
*   Weighting factor $\lambda_s$: Not explicitly defined in [S1], but standard practice applies a weight to balance with other losses.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_sdf_smoothness_loss(network, points, epsilon):
    """
    Computes the SDF Smoothness Loss.
    
    Args:
        network: Neural network predicting SDF values.
        points: Tensor of shape (N, 3) containing query points p.
        epsilon: Tensor of shape (3,) or (N, 3) containing perturbation vector.
        
    Returns:
        Scalar loss value.
    """
    # Compute gradients at original points
    points.requires_grad_(True)
    sdf_original = network(points)
    grad_original = torch.autograd.grad(
        sdf_original.sum(), points, create_graph=True
    )[0]
    
    # Compute gradients at perturbed points
    points_perturbed = points + epsilon
    points_perturbed.requires_grad_(True)
    sdf_perturbed = network(points_perturbed)
    grad_perturbed = torch.autograd.grad(
        sdf_perturbed.sum(), points_perturbed, create_graph=True
    )[0]
    
    # Compute L2 norm of the difference
    diff = grad_original - grad_perturbed
    loss = torch.mean(torch.norm(diff, dim=1))
    
    return loss
```

## Subtopics

### SDF Field Regularization
A variant for discrete SDF fields (e.g., on tetrahedral grids) that uses normal smoothing via gradient alignment and K-nearest neighbor weighted averaging [S2, unofficial]. It computes the gradient of the SDF within a tetrahedron as a linear function of SDF values at vertices and spatial gradients of interpolation weights [S2, unofficial]. Smoothed SDF values are computed using a weighted average of K-nearest sites, where neighbors are found once per up-sampling step via KD-trees [S2, unofficial]. The regularization loss aligns the gradient of the SDF with the gradient of the smoothed SDF [S2, unofficial]. It also employs a total variation loss on tetrahedral mesh edges [S2, unofficial].

**Formula:**
The regularization loss component is [S2, unofficial]:
$$
\mathcal{L}_{reg} = 0.5 \sum_t \left( 1 - \frac{\nabla s_{df,t} \cdot \nabla s_{df,t}^{smooth}}{\|\nabla s_{df,t}\| \|\nabla s_{df,t}^{smooth}\|} \right)^2
$$
Where $\nabla s_{df,t}$ is the gradient in tetrahedron $t$, and $\nabla s_{df,t}^{smooth}$ is the gradient of the smoothed SDF.

**Parameters:**
*   $w_{reg}$: Weight factor for regularization loss [S2, unofficial].
*   $w_{tv}$: Weight factor for total variation loss [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Requires two forward passes and two gradient computations per point (original and perturbed), making it roughly twice the cost of a single gradient evaluation.
*   **Memory Complexity:** Requires storing gradients for both the original and perturbed points during the backward pass.
*   **Strengths:** Directly penalizes geometric irregularities and high-frequency noise in the SDF field, leading to smoother surfaces [S1].
*   **Limitations:** Depends on the choice of perturbation $\varepsilon$; if $\varepsilon$ is too large, it may not capture local smoothness; if too small, it may be sensitive to numerical noise. Not given in [S1] is the optimal range for $\varepsilon$.
*   **When to use:** When the reconstructed surface exhibits jaggedness or high-frequency artifacts despite Eikonal regularization.

## Background
The SDF Smoothness Loss was introduced in the context of ray-distance volume rendering for neural scene reconstruction [S1]. It addresses potential depth ambiguities and surface irregularities by encouraging the SDF gradients to be consistent over small spatial distances [S1].

## Sources
[S1] Ray-Distance Volume Rendering for Neural Scene Reconstruction, https://doi.org/10.1007/978-3-031-72630-9_22, fetched 2026-10-26
[S2] WACV 2025 Open Access Repository, https://openaccess.thecvf.com/content/WACV2025/html/Thomas_VortSDF_3D_Modeling_with_Centroidal_Voronoi_Tesselation_on_Signed_Distance_WACV_2025_paper.html, fetched 2026-10-30 (unofficial)
