# Reflectance and Shading Decomposition Priors

Also called: Intrinsic Priors, Mean Reflectance Regularization, Shading Smoothness

## Overview
Reflectance and Shading Decomposition Priors enforce physical plausibility on the learned radiance field by decoupling material properties (reflectance) from illumination effects (shading). This method solves the ambiguity inherent in intrinsic image decomposition, where scaling reflectance and inversely scaling shading produces identical observed colors, by applying regularization constraints that stabilize the separation. It distinguishes itself from standard NeRF training by explicitly optimizing for intrinsic image components rather than just photometric reconstruction, enabling consistent object insertion and relighting.

## How it works
The method extends the standard NeRF training loop by introducing additional loss terms that constrain the reflectance $\mathbf{R}$ and shading $\mathbf{S}$ components of the predicted color. The total color $\mathbf{C}$ is typically modeled as the product of reflectance and shading: $\mathbf{C} = \mathbf{R} \odot \mathbf{S}$, where $\odot$ denotes element-wise multiplication.

### Inputs and Outputs
*   **Inputs:** Camera poses, ray origins, ray directions, ground truth pixel colors, and the current state of the neural network parameters.
*   **Outputs:** Updated neural network weights, decomposed reflectance $\mathbf{R}(\mathbf{x})$ and shading $\mathbf{S}(\mathbf{x})$ fields for any 3D point $\mathbf{x}$.

### Data Structures
The neural network outputs two additional channels or separate heads for each sampled point along a ray:
1.  **Reflectance $\mathbf{R}(\mathbf{x})$:** A 3-dimensional vector (RGB) representing the intrinsic material color.
2.  **Shading $\mathbf{S}(\mathbf{x})$:** A 3-dimensional vector (RGB) representing the illumination at that point.

### Formulas and Loss Terms
The total training loss $\mathcal{L}_{train}$ is a weighted sum of the photometric reconstruction loss and the intrinsic priors [S1]:

$$
\mathcal{L}_{train} = \lambda_0 \mathcal{L} + \lambda_1 \mathcal{L}_d + \lambda_2 \mathcal{L}_{mean,r} + \lambda_3 \mathcal{L}_{chro} + \lambda_4 \mathcal{L}_{rs} + \lambda_5 \mathcal{L}_{nrs} + \lambda_6 \mathcal{L}_s + \lambda_7 \mathcal{L}_{sn}
$$

Where:
*   $\mathcal{L}$ is the standard photometric loss (e.g., L1 or L2 between rendered and ground truth colors).
*   $\mathcal{L}_d$ is a depth loss (if available).
*   $\lambda_i$ are adjustable hyperparameters.

The specific priors defined in the source are:

1.  **Mean Reflectance Regularization ($\mathcal{L}_{mean,r}$):**
    Constrains the maximum reflectance value across RGB channels to a target average brightness $\beta_{avg,r}$ to prevent unreasonable brightness scaling [S1].
    $$
    \mathcal{L}_{mean,r} = \frac{1}{|\mathcal{R}|} \sum_{\mathbf{r} \in \mathcal{R}} \left| (\mathbf{R}(\mathbf{r}).max) - \beta_{avg,r} \right|
    $$
    Here, $\mathbf{R}(\mathbf{r}).max = \max(R_r, R_g, R_b)$ for the reflectance vector at ray $\mathbf{r}$, and $\beta_{avg,r}$ is a hyperparameter typically set to $0.6$ [S1].

2.  **Shading Smoothness ($\mathcal{L}_s$):**
    Defined as the variance of shading values for all pixels in a single batch, encouraging spatially smooth illumination [S1].
    $$
    \mathcal{L}_s = \text{Var}(\{\mathbf{S}(\mathbf{x}) \mid \mathbf{x} \in \text{Batch}\})
    $$
    *Note: The exact mathematical formulation of the variance over the batch is not explicitly expanded in the source, but it is described as the variance of shading for all pixels in a single batch.*

3.  **Shading-Normal Consistency ($\mathcal{L}_{sn}$):**
    Enforces consistency between shading and surface normals by penalizing large shading differences between nearby points with similar orientations [S1].
    $$
    \mathcal{L}_{sn} = \frac{1}{|\mathcal{R}|} \sum_{\mathbf{r} \in \mathcal{R}} \omega_n(\mathbf{r}, \mathbf{r}') \cdot |\mathbf{S}(\mathbf{r}) - \mathbf{S}(\mathbf{r}')| \cdot (1 - \tanh(\text{Dis}(\mathbf{r}, \mathbf{r}')))
    $$
    Where:
    *   $\mathbf{r}, \mathbf{r}'$ are rays or sampling points.
    *   $\omega_n(\mathbf{r}, \mathbf{r}')$ measures the cosine similarity of the surface normals at $\mathbf{r}$ and $\mathbf{r}'$.
    *   $\text{Dis}(\mathbf{r}, \mathbf{r}')$ is the Euclidean distance between the 3D points associated with $\mathbf{r}$ and $\mathbf{r}'$.

4.  **Other Priors (Mentioned but not detailed in formulas):**
    The source lists $\mathcal{L}_{chro}$ (reflectance chromaticities), $\mathcal{L}_{rs}$ (sparsity), and $\mathcal{L}_{nrs}$ (non-local sparsity) as common intrinsic image decomposition constraints used in the optimization, but does not provide their explicit mathematical definitions in the provided text [S1].

### Pseudocode
The following pseudocode illustrates the integration of these priors into a standard NeRF training step. It assumes a standard NeRF backbone that outputs color, density, and now reflectance and shading.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_intrinsic_priors_loss(rendered_color, gt_color, 
                                  reflectance_field, shading_field, 
                                  normal_field, batch_size, 
                                  beta_avg_r, lambdas):
    """
    Computes the total loss including Reflectance and Shading Decomposition Priors.
    
    Args:
        rendered_color: Tensor of shape (batch_size, 3), predicted RGB color.
        gt_color: Tensor of shape (batch_size, 3), ground truth RGB color.
        reflectance_field: Tensor of shape (batch_size, 3), predicted reflectance R.
        shading_field: Tensor of shape (batch_size, 3), predicted shading S.
        normal_field: Tensor of shape (batch_size, 3), predicted surface normals N.
        batch_size: Integer, number of samples in the batch.
        beta_avg_r: Float, target average max reflectance (default 0.6).
        lambdas: Dict of hyperparameters lambda_0 to lambda_7.
        
    Returns:
        total_loss: Scalar tensor.
    """
    # 1. Standard Photometric Loss (L)
    # Assuming L1 loss as common in NeRF variants, though not explicitly defined in source
    loss_photometric = torch.mean(torch.abs(rendered_color - gt_color))
    
    # 2. Mean Reflectance Regularization (L_mean,r)
    # R(r).max is the max value across RGB channels for each sample
    max_reflectance = torch.max(reflectance_field, dim=1, keepdim=True)[0]
    loss_mean_r = torch.mean(torch.abs(max_reflectance - beta_avg_r))
    
    # 3. Shading Smoothness (L_s)
    # Variance of shading for all pixels in the batch
    # Variance is computed over the batch dimension for each channel, then averaged
    shading_variance = torch.var(shading_field, dim=0)
    loss_shading_smooth = torch.mean(shading_variance)
    
    # 4. Shading-Normal Consistency (L_sn)
    # This requires pairwise comparisons or a local neighborhood. 
    # The source formula sums over rays r, implying a comparison with neighbors r'.
    # In practice, this is often approximated using adjacent samples or a local grid.
    # Since the source does not specify the neighbor selection strategy, 
    # we note this as a complex term requiring implementation details not in source.
    # Placeholder for L_sn calculation based on source formula (5):
    # loss_shading_normal = compute_sn_loss(shading_field, normal_field) 
    # [G] Implementation note: Typically involves computing cosine similarity of normals
    # and distance between 3D points for neighboring samples.
    loss_shading_normal = 0.0 # Placeholder: Not fully specified in source
    
    # 5. Other Priors (L_chro, L_rs, L_nrs)
    # Source mentions these but does not provide formulas.
    loss_chro = 0.0 # Placeholder
    loss_rs = 0.0   # Placeholder
    loss_nrs = 0.0  # Placeholder
    
    # 6. Depth Loss (L_d)
    # Placeholder if depth supervision is used
    loss_depth = 0.0 # Placeholder
    
    # 7. Total Loss
    total_loss = (
        lambdas['lambda_0'] * loss_photometric +
        lambdas['lambda_1'] * loss_depth +
        lambdas['lambda_2'] * loss_mean_r +
        lambdas['lambda_3'] * loss_chro +
        lambdas['lambda_4'] * loss_rs +
        lambdas['lambda_5'] * loss_nrs +
        lambdas['lambda_6'] * loss_shading_smooth +
        lambdas['lambda_7'] * loss_shading_normal
    )
    
    return total_loss
```

### Parameters and Edge Cases
*   **$\beta_{avg,r}$:** Set to $0.6$ for most scenes [S1].
*   **Scale Ambiguity:** A critical pitfall is the scale ambiguity between shading and reflectance. Scaling reflectance by $k$ and shading by $1/k$ yields the same observed color. The Mean Reflectance Regularization helps mitigate unreasonable brightness changes but does not fully resolve the ambiguity without additional constraints or normalization [S1].
*   **Sampling Density:** The source notes that sparse sampling on objects can lead to missing details, which may affect the accuracy of the shading and normal consistency terms [S1].

## Operating characteristics
*   **Complexity:** Adds computational overhead due to the calculation of variance and pairwise normal/shading consistency terms. The shading-normal consistency term $\mathcal{L}_{sn}$ is particularly expensive as it involves pairwise comparisons or local neighborhood searches.
*   **Performance:** The regularization is described as "crucial, particularly for complex scenes," reducing noise in normal maps and improving the smoothness of intrinsic components [S1].
*   **Use Case:** Best suited for applications requiring object insertion, relighting, or physically plausible material decomposition. Not necessary for standard novel view synthesis where only photometric accuracy matters.

## Background
This method originates from the work on "Relighting Scenes With Object Insertions in Neural Radiance Fields" [S1]. It adapts traditional intrinsic image decomposition constraints (chromaticity, sparsity, non-local sparsity) to the NeRF backbone to enable the separation of scene and object intrinsic properties for compositing.

## Sources
[S1] Relighting Scenes With Object Insertions in Neural Radiance Fields, https://doi.org/10.1109/TCSVT.2025.3535599, fetched 2026-10-29
