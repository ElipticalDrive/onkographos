# Training Objective for Fine Representation

Also called: Fine Representation Training Objective, Fine-stage Training Configuration

## Overview
The training objective for the fine representation defines the loss function used to optimize the high-resolution radiance field or voxel grid after the coarse stage has converged. It solves the problem of refining geometric details and texture fidelity without over-regularizing the solution, which can lead to blurry or artifact-prone results. It differs from the coarse stage objective primarily by reducing the weight of regularization terms, a modification empirically shown to yield slightly better visual quality [S2]. It builds on [Training Configuration](2332-training-configuration.md) by specifying the parameter adjustments for the second optimization phase.

## How it works
**Inputs and Outputs:**
*   **Inputs:** The optimized coarse representation (density/alpha fields), the current fine representation parameters (e.g., fine voxel grid values or fine MLP weights), and the training dataset (images, camera poses).
*   **Outputs:** Updated fine representation parameters and the computed training loss scalar.

**Data Structures:**
The fine representation typically mirrors the coarse structure but with higher resolution or different encoding. For direct voxel optimization, this is a high-resolution 3D grid of density and color values [S1, S2]. For MLP-based NeRFs, it is a separate network or a higher-frequency encoding branch.

**Steps:**
1.  **Ray Sampling:** Sample points along rays using the same sampling strategy as the coarse stage, potentially biased by the coarse density field to focus on occupied space [S1, S2].
2.  **Rendering:** Compute the rendered image color $C(r)$ for each training ray $r$ by volume rendering the fine representation's density $\sigma_f$ and color $c_f$.
3.  **Loss Computation:** Calculate the total loss $\mathcal{L}_{fine}$ as a weighted sum of reconstruction loss and regularization losses.
4.  **Optimization:** Update the fine representation parameters using an optimizer (e.g., Adam) based on $\mathcal{L}_{fine}$.

**Formulas:**
The total loss is defined as:
$$ \mathcal{L}_{fine} = \mathcal{L}_{rec} + \lambda_{reg} \mathcal{L}_{reg} $$

Where:
*   $\mathcal{L}_{rec}$ is the reconstruction loss (e.g., Mean Squared Error or L1 loss) between the rendered image and the ground truth image. This term is identical to the coarse stage [S1, S2].
*   $\mathcal{L}_{reg}$ represents regularization losses (e.g., density smoothness, sparsity constraints).
*   $\lambda_{reg}$ is the regularization weight.

**Parameters:**
*   **Regularization Weight ($\lambda_{reg}$):** The key difference from the coarse stage is that $\lambda_{reg}$ is set to a **smaller value** in the fine stage. The sources state that using smaller regularization weights leads to slightly better visual quality empirically [S2]. The exact numerical value is not given in the sources; it is a hyperparameter tuned relative to the coarse stage [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_fine_stage(coarse_repr, fine_repr, dataset, optimizer, lambda_reg_coarse):
    """
    Optimizes the fine representation using a reduced regularization weight.
    
    Args:
        coarse_repr: The optimized coarse representation (used for sampling bias if applicable).
        fine_repr: The fine representation parameters (voxels or MLP weights).
        dataset: Iterable of (image, camera_pose) pairs.
        optimizer: Optimizer instance for fine_repr.
        lambda_reg_coarse: The regularization weight used in the coarse stage.
        
    Returns:
        fine_repr: The updated fine representation.
    """
    # Define the reduced regularization weight
    # The sources specify a smaller weight but do not give a specific ratio or value [S2]
    lambda_reg_fine = lambda_reg_coarse * 0.1  # [G] Placeholder ratio; implementer must tune this based on empirical results
    
    for image, camera_pose in dataset:
        # 1. Sample points along rays
        # Use sampling strategy consistent with coarse stage, potentially using coarse density for biasing [S1, S2]
        ray_origins, ray_directions, sample_points = sample_rays(camera_pose, coarse_repr)
        
        # 2. Query fine representation
        densities, colors = fine_repr.query(sample_points)
        
        # 3. Volume render the image
        rendered_image = volume_render(ray_origins, ray_directions, densities, colors)
        
        # 4. Compute reconstruction loss (same as coarse stage)
        loss_rec = mse_loss(rendered_image, image)
        
        # 5. Compute regularization loss
        # The specific form of L_reg depends on the representation (e.g., density smoothness)
        loss_reg = compute_regularization_loss(fine_repr)
        
        # 6. Compute total loss with reduced regularization weight
        loss = loss_rec + lambda_reg_fine * loss_reg
        
        # 7. Backpropagate and update
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
    return fine_repr
```

**Edge Cases and Pitfalls:**
*   **Over-smoothing:** If the regularization weight is not reduced sufficiently, the fine stage may fail to capture high-frequency details, resulting in blurry outputs [S2].
*   **Instability:** Reducing regularization too much may lead to noisy or unstable optimization if the reconstruction loss alone is insufficient to constrain the high-dimensional fine representation [S2].

## Subtopics

### [StyleGAN2-based Discriminator Integration](8938-stylegan2-based-discriminator-integration.md)
This method integrates a StyleGAN2-based discriminator into the NeRF training loop to evaluate the perceptual quality of rendered images, addressing the limitation of pixel-wise losses like MSE which often produce blurry results. It differs from sibling methods such as [Test-Time Finetuning](12983-test-time-finetuning.md) by actively participating in the optimization loop via adversarial gradients rather than post-processing. The discriminator is configured to accept 64x64 RGB images and outputs a real/fake probability score, providing a training signal that encourages sharp, photorealistic textures.

### [Test-Time Finetuning](12983-test-time-finetuning.md)
Test-Time Finetuning is a post-sampling optimization step that refines the sampled scene code by minimizing a combined loss of rendering fidelity and a diffusion prior, while keeping the diffusion model and decoder parameters frozen. It addresses the limitation where reconstruction guidance alone cannot strictly enforce rendering constraints towards faithful reconstruction, distinguishing it from two-stage methods that ignore the prior term during initial training, which results in biased and noisy scene codes.

### Geometry-Aware Attention
A learnable module combining spatial (pixel) and geometric (camera) attention to regulate contributions from neighbours and reduce ghosting artifacts from remaining misalignments [S3]. It computes pixel attention weights $\psi_{pix}^i \in \mathbb{R}^{w \times h}$ using a module $\mathcal{A}_{pix}$ with two convolutional layers and sigmoid activation, and camera attention weights $\psi_{cam}^i \in \mathbb{R}^1$ using a module $\mathcal{A}_{cam}$ with two fully connected layers and sigmoid activation [S3]. The final attention-regulated features are computed as $H_{i \rightarrow k}^{f^a} = \psi_{cam}^i \times \psi_{pix}^i \times H_{i \rightarrow k}^{f'}$, where $H_{i \rightarrow k}^{f'}$ are the input neighbour features [S3].

## Operating characteristics
*   **Time Complexity:** Similar to the coarse stage, dominated by volume rendering and backpropagation through the fine representation. The fine representation is larger, so per-step computation is higher [S1, S2].
*   **Memory Complexity:** Higher than the coarse stage due to the increased resolution or capacity of the fine representation [S1, S2].
*   **Performance:** Empirically leads to slightly better visual quality compared to using the same regularization weights as the coarse stage [S2].
*   **Strengths:** Allows for the capture of high-frequency details and textures that the coarse stage cannot represent.
*   **Limitations:** Requires careful tuning of the regularization weight to balance detail capture and noise/artifact suppression. Not given in the sources: specific convergence rates or memory usage numbers.

## Background
The concept of multi-stage training with adjusted objectives is central to modern radiance field optimization, particularly in methods like Instant-NGP and Direct Voxel Grid Optimization. The specific observation that reducing regularization weights in the fine stage improves quality was highlighted in the Direct Voxel Grid Optimization paper [S2]. The integration of advanced perceptual losses (like StyleGAN2 discriminators) and post-training refinement (like Test-Time Finetuning) represents ongoing efforts to overcome the limitations of standard pixel-wise losses [S1, S3].

## Sources
[S1] Direct Voxel Grid Optimization: Super-fast Convergence for Radiance Fields Reconstruction, https://doi.org/10.1109/cvpr52688.2022.00538, fetched 2026-10-26
[S2] Direct Voxel Grid Optimization: Super-Fast Convergence ..., https://openaccess.thecvf.com/content/CVPR2022/papers/Sun_Direct_Voxel_Grid_Optimization_Super-Fast_Convergence_for_Radiance_Fields_Reconstruction_CVPR_2022_paper.pdf, fetched 2026-10-29
[S3] RoGUENeRF: A Robust Geometry-Consistent Universal Enhancer for NeRF, https://doi.org/10.1007/978-3-031-73254-6_4, fetched 2026-10-29
