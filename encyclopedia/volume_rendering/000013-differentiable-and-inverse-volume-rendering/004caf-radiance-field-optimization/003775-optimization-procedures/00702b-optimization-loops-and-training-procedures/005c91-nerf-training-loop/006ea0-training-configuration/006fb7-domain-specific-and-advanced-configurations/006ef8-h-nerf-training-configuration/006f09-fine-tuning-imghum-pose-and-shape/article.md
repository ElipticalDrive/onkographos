# Fine-tuning imGHUM Pose and Shape

Also called: Incremental Correction, Pose and Shape Fine-tuning

## Overview
This method improves the geometric fit of the imGHUM model during H-NeRF training by optimizing time-consistent shape corrections and per-frame pose corrections, rather than keeping the initial imGHUM fit parameters fixed [S1, unofficial]. It addresses errors in the initial imGHUM fitting that propagate into the learned radiance field $F_\omega$ and displacement field $\Delta H_\omega$ [S1, unofficial]. Unlike methods that rely on fixed priors or indirect pose modeling, this approach explicitly refines the parametric model's alignment with the input video sequence through a dedicated loss function [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Initial imGHUM shape parameters $\boldsymbol{\beta}$, initial pose parameters $\boldsymbol{\theta}$, human foreground segmentation masks $\mathbf{M}(\mathbf{r})$ for rays $\mathbf{r}$, and the set of rays $\mathcal{R}$ sampled from the video frames.
*   **Outputs:** Optimized incremental shape correction $\Delta\boldsymbol{\beta}$ and per-frame pose corrections $\Delta\boldsymbol{\theta}(t)$ for each time frame $t$.

**Data Structures:**
*   $\Delta\boldsymbol{\beta}$: A vector of shape correction parameters. The dimensionality matches the shape space of imGHUM.
*   $\Delta\boldsymbol{\theta}(t)$: A vector of pose correction parameters for frame $t$. The dimensionality matches the pose space of imGHUM.
*   $\mathbf{M}(\mathbf{r})$: Binary mask value (0 or 1) indicating if ray $\mathbf{r}$ intersects the human foreground.
*   $d_{\min}(\cdot)$: The minimum distance from the ray origin to the SDF surface defined by the corrected imGHUM parameters.

**Numbered Steps:**
1.  Initialize the incremental corrections $\Delta\boldsymbol{\beta}$ and $\Delta\boldsymbol{\theta}(t)$ to zero vectors.
2.  For each training iteration:
    a. Sample a batch of rays $\mathcal{R}$ from the video frames.
    b. For each ray $\mathbf{r}$, compute the minimum distance $d_{\min}$ to the SDF surface parameterized by the corrected shape $\boldsymbol{\beta} + \Delta\boldsymbol{\beta}$ and pose $\boldsymbol{\theta} + \Delta\boldsymbol{\theta}(t)$.
    c. Compute the fitting loss $\mathcal{L}_{\text{fit}}$ using the Binary Cross-Entropy (BCE) between the segmentation mask $\mathbf{M}(\mathbf{r})$ and the distance $d_{\min}$.
    d. Compute the regularization loss $\mathcal{L}_{\text{inc}}$ based on the L2 norms of $\Delta\boldsymbol{\beta}$ and $\Delta\boldsymbol{\theta}(t)$.
    e. Combine the losses: $\mathcal{L} = \mathcal{L}_{\text{fit}} + \mathcal{L}_{\text{inc}}$.
    f. Update $\Delta\boldsymbol{\beta}$ and $\Delta\boldsymbol{\theta}(t)$ using gradient descent.

**Formulas:**
The fitting loss aligns the imGHUM fit with the human foreground segmentation [S1, unofficial]:
$$
\mathcal{L}_{\text{fit}} = \sum_{\mathbf{r} \in \mathcal{R}} \text{BCE}(\mathbf{M}(\mathbf{r}), d_{\min}(\mathbf{r}|\boldsymbol{\beta} + \Delta\boldsymbol{\beta}, \boldsymbol{\theta} + \Delta\boldsymbol{\theta}(t)))
$$
where:
*   $\mathbf{M}(\mathbf{r})$ is the human foreground segmentation mask for ray $\mathbf{r}$.
*   $d_{\min}(\mathbf{r}|\boldsymbol{\beta} + \Delta\boldsymbol{\beta}, \boldsymbol{\theta} + \Delta\boldsymbol{\theta}(t))$ is the minimum distance to the SDF surface parameterized by the corrected shape $\boldsymbol{\beta} + \Delta\boldsymbol{\beta}$ and pose $\boldsymbol{\theta} + \Delta\boldsymbol{\theta}(t)$.
*   $\mathcal{R}$ is the set of rays.

The regularization loss ensures time consistency and small increments [S1, unofficial]:
$$
\mathcal{L}_{\text{inc}} = \|\Delta\boldsymbol{\beta}\|_2 + \frac{1}{N_t} \sum_{t=1}^{N_t} \|\Delta\boldsymbol{\theta}\|_2
$$
where:
*   $N_t$ is the total number of time frames.
*   $\|\cdot\|_2$ denotes the L2 norm.

**Parameters:**
*   **Optimizer:** Adam [S1, unofficial].
*   **Learning Rate:** 0.001, exponentially decayed by a factor of 0.1 [S1, unofficial].
*   **Maximum Iterations:** 10,000 [S1, unofficial].
*   **Ray Batch Size:** 4,000 [S1, unofficial].
*   **Noise:** Gaussian noise with standard deviation $\sigma = 0.1$ is added to the NeRF conditioning code (pose $\theta$, transformation $T$, viewing direction $v$) to prevent overfitting [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The distance function $d_{\min}$ requires accurate SDF evaluation. Degenerate poses or extreme shapes may lead to unstable gradients if the SDF is not well-defined.
*   **Boundaries:** The method assumes valid segmentation masks $\mathbf{M}(\mathbf{r})$. Poor segmentation quality will directly impact the fitting loss.
*   **Overfitting:** Without the regularization term $\mathcal{L}_{\text{inc}}$ and noise injection, the corrections may overfit to noise in the segmentation or initial fit errors [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

def compute_sdf_distance(rays, beta, delta_beta, theta, delta_theta_t, imghum_model):
    """
    Computes the minimum distance from rays to the imGHUM SDF surface.
    rays: Tensor of shape (N, 3) representing ray origins.
    beta: Tensor of shape (D_beta) initial shape parameters.
    delta_beta: Tensor of shape (D_beta) shape correction.
    theta: Tensor of shape (D_theta) initial pose parameters.
    delta_theta_t: Tensor of shape (D_theta) pose correction for current frame.
    imghum_model: Function/object that computes SDF given shape and pose.
    Returns: Tensor of shape (N,) with minimum distances.
    """
    # Combine initial and corrected parameters
    current_beta = beta + delta_beta
    current_theta = theta + delta_theta_t
    
    # Compute SDF distances for all rays
    # Note: imghum_model should handle batched ray queries
    distances = imghum_model.compute_min_distance(rays, current_beta, current_theta)
    return distances

def fine_tune_imghum_pose_shape(
    rays_batch, 
    masks_batch, 
    beta, 
    theta, 
    delta_beta, 
    delta_theta_t, 
    imghum_model, 
    optimizer, 
    N_t
):
    """
    Performs one step of fine-tuning imGHUM pose and shape.
    """
    optimizer.zero_grad()
    
    # Compute distances
    distances = compute_sdf_distance(
        rays_batch, beta, delta_beta, theta, delta_theta_t, imghum_model
    )
    
    # Compute fitting loss (BCE)
    # Note: BCE typically expects probabilities, so distances might need sigmoid or normalization
    # The source uses BCE(M, d_min), implying d_min is treated as a probability-like value
    # or the SDF output is scaled/transformed to [0,1]. Assuming standard BCE implementation.
    fitting_loss = nn.BCELoss()(masks_batch, distances)
    
    # Compute regularization loss
    shape_reg = torch.norm(delta_beta, p=2)
    pose_reg = torch.norm(delta_theta_t, p=2)
    regularization_loss = shape_reg + (1.0 / N_t) * pose_reg
    
    # Total loss
    total_loss = fitting_loss + regularization_loss
    
    # Backpropagate
    total_loss.backward()
    optimizer.step()
    
    return total_loss.item()

# Training loop setup
# beta_init, theta_init: Initial imGHUM parameters
# delta_beta = torch.zeros_like(beta_init, requires_grad=True)
# delta_theta_t = torch.zeros_like(theta_init, requires_grad=True)
# optimizer = optim.Adam([delta_beta, delta_theta_t], lr=0.001)
# scheduler = optim.lr_scheduler.ExponentialLR(optimizer, gamma=0.1)

# for iteration in range(10000):
#     rays_batch, masks_batch = sample_rays_and_masks(batch_size=4000)
#     # Add noise to conditioning code if applicable (not directly in this sub-step's loss)
#     loss = fine_tune_imghum_pose_shape(...)
#     scheduler.step()
```

## Subtopics

### H-NeRF Training Configuration
The parent configuration that includes this fine-tuning step as part of the overall training pipeline for dynamic human scenes. [Link](21746-h-nerf-training-configuration.md)

## Operating characteristics
*   **Time Complexity:** The method adds an optimization loop over the imGHUM parameters ($\Delta\boldsymbol{\beta}$ and $\Delta\boldsymbol{\theta}(t)$) to the standard NeRF training loop. The complexity is dominated by the SDF evaluation for each ray in the batch. With 4,000 rays per batch and 10,000 iterations, the computational cost is significant but manageable on modern GPUs.
*   **Memory Complexity:** Requires storing the gradients for $\Delta\boldsymbol{\beta}$ and $\Delta\boldsymbol{\theta}(t)$, which is minimal compared to the NeRF weights.
*   **Measured Performance:** H-NeRF with this fine-tuning outperforms NeuralBody across datasets (RenderPeople, GHS3D, PeopleSnapshot, Human3.6M) and metrics (PSNR, SSIM, LPIPS, Chamfer Distance, Normal Consistency), especially for geometric reconstruction [S1, unofficial].
*   **Strengths:** Improves geometric accuracy by correcting initial imGHUM fitting errors. Time-consistent regularization prevents jitter in the shape and pose over time.
*   **Limitations:** Relies on the quality of the initial imGHUM fit and segmentation masks. May not converge if the initial fit is severely incorrect.
*   **When to Use:** Use when high-quality geometric reconstruction of dynamic humans is required and the initial imGHUM fit is not sufficient.
*   **When Not to Use:** Avoid if computational resources are extremely limited or if the initial imGHUM fit is already very accurate.

## Background
This method was introduced in the H-NeRF paper by [S1, unofficial]. It builds on the observation that initial imGHUM fitting can be a source of error in learning the radiance and displacement fields for dynamic human sequences. The technique is part of the broader H-NeRF framework for neural radiance fields for rendering and temporal reconstruction of humans in motion.

## Sources
[S1] H-NeRF: Neural Radiance Fields for Rendering and Temporal Reconstruction of Humans in Motion, https://doi.org/10.48550/arxiv.2110.13746, fetched 2026-10-26 (unofficial)
