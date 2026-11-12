# Multiplex Training via S3IM

Also called: S3IM Training, Stochastic Structural SIMilarity Training

## Overview
Multiplex Training via S3IM is a training paradigm that combines a conventional pixel-wise loss (e.g., MSE or L1) with a Stochastic Structural SIMilarity (S3IM) loss to improve neural field reconstruction quality, particularly for complex details and sparse inputs [S1]. It sets itself apart from standard SSIM optimization by using stochastic patch sampling to capture non-local structural information from distant pixels across different images, whereas standard SSIM via local patches only captures structural information from nearby pixels within a single image [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A dataset of rays $\mathcal{D}$, a neural field model with parameters $\Theta$, hyperparameters $M$ (number of stochastic patches), $K$ (kernel size), $s$ (stride), and $\lambda$ (loss weight).
*   **Outputs:** Updated model parameters $\Theta$.

**Data Structures:**
*   $\mathcal{R}$: A minibatch of rays sampled from $\mathcal{D}$.
*   $\mathcal{C}$: Ground-truth pixel colors corresponding to rays in $\mathcal{R}$.
*   $\hat{\mathcal{C}}$: Rendered pixel colors corresponding to rays in $\mathcal{R}$.
*   $\Patch^{(m)}$: A stochastic patch generation function for the $m$-th iteration.

**Numbered Steps:**
1.  Sample a data minibatch of rays $\mathcal{R}$ from dataset $\mathcal{D}$ [S1].
2.  Obtain ground-truth pixels $\mathcal{C} = \{ \mathbf{C}(\mathbf{r}) | \mathbf{r} \in \mathcal{R} \}$ [S1].
3.  Compute rendered pixels $\hat{\mathcal{C}} = \{ \hat{\mathbf{C}}(\mathbf{r}) | \mathbf{r} \in \mathcal{R} \}$ using the current model $\Theta$ [S1].
4.  Initialize the conventional loss $L(\Theta)$ based on the chosen pixel-wise metric (typically L1 or MSE) [S1].
5.  Loop $m$ from 1 to $M$:
    a. Initialize the stochastic patch generation function $\Patch^{(m)}$ [S1].
    b. Transform rendered pixels into the rendered stochastic patch $\Patch^{(m)}(\hat{\mathcal{C}})$ [S1].
    c. Transform ground-truth pixels into the ground-truth stochastic patch $\Patch^{(m)}(\mathcal{C})$ [S1].
    d. Compute the SSIM between $\Patch^{(m)}(\hat{\mathcal{C}})$ and $\Patch^{(m)}(\mathcal{C})$ using kernel size $K \times K$ and stride $s$ [S1].
6.  Calculate the average SSIM over all stochastic patches:
    $$ \SSSIM (\hat{\mathcal{C}}, \mathcal{C}) = \frac{1}{M} \sum_{m=1}^{M} \SSIM (\Patch^{(m)}(\hat{\mathcal{C}}), \Patch^{(m)}(\mathcal{C})) $$
    [S1]
7.  Calculate the S3IM loss:
    $$ L_{\SSSIM}(\Theta, \mathcal{R}) = 1 - \SSSIM (\hat{\mathcal{C}}, \mathcal{C}) $$
    [S1]
8.  Calculate the total multiplex loss:
    $$ L_M(\Theta) = L(\Theta) + \lambda L_{\SSSIM}(\Theta, \mathcal{R}) $$
    [S1]
9.  Compute the gradient $\nabla L_M(\Theta)$ [S1].
10. Update model parameters $\Theta$ using an SGD-like algorithm $A$ [S1].

**Formulas:**
*   **S3IM Loss:**
    $$ L_{\SSSIM}(\Theta, \mathcal{R}) = 1 - \frac{1}{M} \sum_{m=1}^{M} \SSIM (\Patch^{(m)}(\hat{\mathcal{C}}), \Patch^{(m)}(\mathcal{C})) $$
    [S1]
    Where $\SSSIM$ is the average SSIM over $M$ stochastic patches, $\hat{\mathcal{C}}$ are rendered pixels, $\mathcal{C}$ are ground-truth pixels, and $\Patch^{(m)}$ generates the $m$-th stochastic patch.
*   **Base Color Loss (L1 form):**
    $$ L_{\mathrm{color}} = \frac{1}{\|\mathcal{R}\|} \sum_{\mathbf{r} \in \mathcal{R}} \| \hat{\mathbf{C}}(\mathbf{r}) - \mathbf{C}(\mathbf{r}) \|_{1} $$
    [S1]
*   **Multiplex Color Loss (for Neural Surface Representations like NeuS):**
    $$ L_{\mathrm{M},\mathrm{color}} = L_{\mathrm{color}} + \lambda L_{\SSSIM} $$
    [S1]

**Parameters:**
*   $M$: The number of stochastic patches. Default value is 10 to reduce stochasticity, but $M=1$ produces nearly the same good results. Approaching $+\infty$ eliminates stochasticity [S1].
*   $K$: Kernel size for SSIM calculation. Default setting is $K=4$ [S1].
*   $s$: Stride size for SSIM calculation. Default setting is $s=K$ (i.e., stride equals kernel size) [S1].
*   $\lambda$: The weight for the S3IM loss term. Requires fine-tuning in practice [S1].

**Edge Cases and Pitfalls:**
*   Directly optimizing standard SSIM via local patches may not work well and increases coding and computational costs, potentially hurting NeRF performance [S1].
*   S3IM lies in the range $[-1, 1]$ and is positively correlated with image quality, hence the loss is defined as $1 - \SSSIM$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multiplex_training_s3im(model, dataset, optimizer, M=10, K=4, s=4, lambda_s3im=1.0):
    """
    Training loop for Multiplex Training via S3IM.
    
    Args:
        model: Neural field model with parameters theta.
        dataset: Dataset D providing rays and ground truth colors.
        optimizer: SGD-like algorithm A for updating theta.
        M: Number of stochastic patches.
        K: Kernel size for SSIM.
        s: Stride for SSIM.
        lambda_s3im: Weight for S3IM loss.
    """
    while not stopping_criterion_met():
        # 1. Sample minibatch
        rays, gt_colors = dataset.sample_minibatch()
        
        # 2. Render
        rendered_colors = model.render(rays)
        
        # 3. Compute conventional loss (L1 as per source for NeuS/general case)
        # Note: Source specifies L1 for NeuS, MSE is common for NeRF. 
        # We implement L1 as the base L(Theta) as per Eq 13/14 context.
        conventional_loss = mean_l1_loss(rendered_colors, gt_colors)
        
        # 4. Compute S3IM Loss
        ssim_sum = 0.0
        for m in range(M):
            # Initialize stochastic patch generation
            # The source defines P^(m) as a function that transforms pixels into a stochastic patch.
            # In implementation, this typically involves random cropping or sampling 
            # of pixel coordinates to form a patch, or a randomized weighting/filtering.
            # Since the exact mathematical definition of P^(m) is not fully detailed 
            # beyond "stochastic patch generation function", we assume a standard 
            # stochastic sampling of pixel indices or patches.
            
            # Transform rendered pixels into rendered stochastic patch
            # Note: SSIM is computed on the patches. 
            # We assume P^(m) returns the subset of pixels/patches for the m-th sample.
            rendered_patch, gt_patch = generate_stochastic_patches(rendered_colors, gt_colors, m)
            
            # Compute SSIM between patches
            # SSIM function takes two image/patch tensors, kernel size K, stride s
            ssim_value = compute_ssim(rendered_patch, gt_patch, kernel_size=K, stride=s)
            ssim_sum += ssim_value
            
        # Average SSIM
        avg_ssim = ssim_sum / M
        
        # S3IM Loss
        s3im_loss = 1.0 - avg_ssim
        
        # 5. Total Loss
        total_loss = conventional_loss + lambda_s3im * s3im_loss
        
        # 6. Update
        optimizer.zero_grad()
        total_loss.backward()
        optimizer.step()
```

## Subtopics

### S3IM (Stochastic Structural SIMilarity)
The core loss metric used in this method. It differs from standard SSIM by averaging SSIM over $M$ stochastic patches rather than local fixed patches, allowing it to capture non-local structural information. [Link](10895-s3im-stochastic-structural-similarity.md)

### NeuS
A neural surface representation method where the multiplex training via S3IM is specifically adapted by replacing the color loss with the multiplex color loss $L_{\mathrm{M},\mathrm{color}}$. [Link](10895-s3im-stochastic-structural-similarity.md) (Contextual link to application)

## Operating characteristics
*   **Time and Memory Complexity**: The extra computational cost is limited because S3IM requires only one backpropagation per iteration, similar to standard training, despite the loop over $M$ patches [S1].
*   **Measured Performance**:
    *   **Replica Dataset (DVGO)**: Mean PSNR improved from 17.07 to 33.50, SSIM from 0.696 to 0.955, LPIPS from 0.510 to 0.0637 [S1].
    *   **Replica Dataset (TensoRF)**: Mean PSNR improved from 14.30 to 39.05, SSIM from 0.574 to 0.971, LPIPS from 0.689 to 0.0454 [S1].
    *   **T&T Dataset (DVGO)**: Mean PSNR improved from 22.42 to 23.20, SSIM from 0.776 to 0.809, LPIPS from 0.236 to 0.176 [S1].
    *   **Sparse Inputs (20% training images, Truck scene)**: PSNR improved by up to 4.32 compared to standard training [S1].
    *   **Dynamic Scene (Lego, D-NeRF)**: PSNR improved from 21.29 to 23.36, SSIM from 0.821 to 0.900, LPIPS from 0.0634 to 0.0482 [S1].
*   **Scaling**: Performance improvements are more significant with sparser inputs (e.g., Replica vs T&T) [S1].
*   **Strengths**: Improves RGB image quality metrics and geometric quality metrics (e.g., Chamfer Distance) significantly. Robust to image corruption and noise [S1].
*   **Limitations**: Requires fine-tuning of $\lambda$. Standard SSIM optimization via local patches is inferior and more costly [S1].
*   **When to use**: When training neural fields (NeRF, DVGO, TensoRF, D-NeRF, NeuS) with sparse inputs or noisy/corrupted images, or when high-fidelity structural details are required [S1].

## Background
*   **Origin**: Introduced in the paper "S3IM: Stochastic Structural SIMilarity and Its Unreasonable Effectiveness for Neural Fields" [S1].
*   **History**: The method is model-agnostic and orthogonal to existing point-wise training paradigms and NeRF variants like DVGO, TensoRF, D-NeRF, and NeuS [S1]. It was developed to address the limitations of standard SSIM optimization in neural field reconstruction.
*   **Usage**: Applied to NeRF, DVGO, TensoRF, D-NeRF, and NeuS [S1].

## Sources
[S1] S3IM: Stochastic Structural SIMilarity and Its Unreasonable Effectiveness for Neural Fields, https://doi.org/10.1109/iccv51070.2023.01652, fetched 2026-10-26
