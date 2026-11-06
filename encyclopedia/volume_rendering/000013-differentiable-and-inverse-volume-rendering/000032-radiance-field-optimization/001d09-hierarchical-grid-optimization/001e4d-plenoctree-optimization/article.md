# PlenOctree Optimization

Also called: Structure Refinement (Octree Optimization)

## Overview
PlenOctree Optimization is a post-training refinement process that finetunes a converted PlenOctree on original training images using Stochastic Gradient Descent (SGD) to improve image quality [S1, unofficial]. It solves the residual approximation errors introduced during the conversion from a neural network to a sparse voxel grid [S1, unofficial]. Unlike [Plenoxels Optimization Configuration](929-plenoxels-optimization-configuration.md), which configures fixed grid parameters, PlenOctree Optimization actively adjusts the density and spherical harmonic coefficients of the octree leaves [S1, unofficial]. It also differs from [Coarse-to-Fine Optimization with Pruning](1153-coarse-to-fine-with-pruning.md) by operating on a pre-converted sparse structure rather than optimizing the network and sparsity simultaneously [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A converted PlenOctree containing density $\sigma$ and spherical harmonic (SH) coefficients for RGB channels at each leaf node [S1, unofficial].
*   **Input:** Training images and corresponding camera poses [S1, unofficial].
*   **Output:** Optimized PlenOctree attributes (density and SH coefficients) [S1, unofficial].

**Data Structures:**
The PlenOctree is a sparse octree. Each leaf node stores:
*   Density $\sigma$ (scalar).
*   SH coefficients for Red, Green, and Blue channels (vectors).
*   The structure is fixed during optimization; only the attribute values change [S1, unofficial].

**Parameters:**
*   **Learning Rate:** $1 \times 10^7$ for NeRF-synthetic dataset; $1.5 \times 10^6$ for Tanks&Temples dataset [S1, unofficial].
*   **Maximum Epochs:** 80 for NeRF-synthetic; 40 for Tanks&Temples [S1, unofficial].
*   **Precision:** Float32 for optimization stability; Float16 for storage after optimization [S1, unofficial].
*   **Validation Set:** 10% of training set held out for Tanks&Temples [S1, unofficial].

**Loss Function:**
The optimization minimizes the RGB reconstruction loss:
$$
L_{RGB} = \sum_{\mathbf{r} \in R} \| \mathbf{C}(\mathbf{r}) - \hat{\mathbf{C}}(\mathbf{r}) \|_2^2
$$
where $\mathbf{C}(\mathbf{r})$ is the ground truth color and $\hat{\mathbf{C}}(\mathbf{r})$ is the rendered color [S1, unofficial]. Note that sparsity loss is not applied because the octree is already sparse [S1, unofficial].

**Steps:**
1.  **Initialize:** Start with the PlenOctree converted from the NeRF-SH model [S1, unofficial].
2.  **Iterate:** For each epoch up to the maximum limit:
    a. Sample rays from the training set [S1, unofficial].
    b. Render $\hat{\mathbf{C}}(\mathbf{r})$ using the current PlenOctree attributes [S1, unofficial].
    c. Compute $L_{RGB}$ [S1, unofficial].
    d. Compute gradients of $L_{RGB}$ with respect to $\sigma$ and SH coefficients [S1, unofficial].
    e. Update attributes using SGD with the specified learning rate [S1, unofficial].
3.  **Early Stopping:** Monitor PSNR on the validation set (if available). Stop optimization if PSNR does not improve [S1, unofficial].
4.  **Storage:** Convert optimized attributes to Float16 [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def plenoctree_optimization(plenoctree, training_data, dataset_type, max_epochs):
    # Set learning rate based on dataset
    if dataset_type == "NeRF-synthetic":
        lr = 1e7
        max_epochs = 80
    elif dataset_type == "Tanks&Temples":
        lr = 1.5e6
        max_epochs = 40
        # Hold out 10% for validation
        train_set, val_set = split_validation(training_data, 0.1)
    else:
        raise ValueError("Unknown dataset type")

    # Optimization loop
    best_psnr = -1
    for epoch in range(max_epochs):
        # Sample rays from training set
        rays = sample_rays(train_set)
        
        # Render using current PlenOctree attributes
        # Uses analytic derivatives implemented in custom CUDA kernels [S1, unofficial]
        rendered_colors = render_plenoctree(plenoctree, rays)
        
        # Compute loss
        ground_truth_colors = get_ground_truth(rays)
        loss = sum((gt - pred)**2 for gt, pred in zip(ground_truth_colors, rendered_colors))
        
        # Compute gradients and update attributes (SGD)
        gradients = compute_gradients(loss, plenoctree.attributes)
        update_attributes(plenoctree.attributes, gradients, lr)
        
        # Early stopping check
        if dataset_type == "Tanks&Temples":
            val_psnr = evaluate_psnr(plenoctree, val_set)
            if val_psnr > best_psnr:
                best_psnr = val_psnr
                save_best_model(plenoctree)
            else:
                break

    # Convert to Float16 for storage
    plenoctree.attributes = convert_to_float16(plenoctree.attributes)
    return plenoctree
```

**Edge Cases and Pitfalls:**
*   **Precision:** Optimization must be done in Float32 to ensure stability; using lower precision may lead to convergence issues [S1, unofficial].
*   **Sparsity Loss:** Do not apply sparsity loss during this phase as the structure is already sparse [S1, unofficial].

## Subtopics

### [PlenOctree Conversion](1359-plenoctree-conversion.md)
PlenOctree Conversion is a pipeline that transforms a trained NeRF-SH model into a sparse octree representation for real-time volume rendering. It solves the performance bottleneck of conventional NeRFs, which require dense sampling and neural network inference for every ray step dependent on viewing direction. Unlike [Analytic Derivatives of PlenOctree Rendering](1360-analytic-derivatives-of-plenoctree-rendering.md), which focuses on differentiable optimization of the octree structure, this method focuses on the construction and tabulation of the view-dependent volume from a pre-trained network.

### [Analytic Derivatives of PlenOctree Rendering](1360-analytic-derivatives-of-plenoctree-rendering.md)
This method provides the closed-form gradients of the NeRF piecewise constant volume rendering equation with respect to the density ($\boldsymbol{\sigma}$) and color ($\mathbf{c}$) parameters. It enables direct optimization of PlenOctree attributes using gradient-based methods (e.g., SGD) to refine image quality.

### PlenOctree Fine-tuning
PlenOctree Fine-tuning is a variant where training of the initial NeRF model is stopped early, converted into a PlenOctree, and then trained significantly faster as it no longer involves neural networks [S1, unofficial]. This approach accelerates NeRF-based rendering by 5 orders of magnitude without loss in image quality [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   **Speed:** PlenOctree optimization operates at about 3 million rays per second, compared to about 9,000 for NeRF training [S1, unofficial].
*   **Optimization Time:** Around 10 minutes to finish PlenOctree optimization for each scene on a single NVIDIA V100 GPU [S1, unofficial].
*   **Memory:** The sparse octree structure reduces memory usage compared to dense grids, but intermediate memory consumption during training can be higher than some explicit representations because all feature volumes need to be decoded to trace rays [S2].

**Measured Performance:**
*   **NeRF-synthetic:** 80 epochs, LR $1 \times 10^7$ [S1, unofficial].
*   **Tanks&Temples:** 40 epochs, LR $1.5 \times 10^6$ [S1, unofficial].
*   **Quality:** State-of-the-art level performance for image quality, comparable to original NeRF [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Significant speedup in rendering (3500x+ faster than NeRF) [S1, unofficial]; fast optimization time (10 mins) [S1, unofficial]; no loss in image quality [S1, unofficial].
*   **Limitations:** Requires a pre-trained NeRF model; optimization is only applicable after conversion [S1, unofficial].

**When to Use:**
Use when real-time rendering is required and a pre-trained NeRF model is available. Use when computational resources for full NeRF training are limited.

**When Not to Use:**
Do not use if the initial NeRF model has not converged or if the scene requires dynamic geometry that cannot be captured by a static octree.

## Background
PlenOctree Optimization was introduced in the PlenOctrees paper [S1, unofficial]. It was developed to address the slow rendering speed of Neural Radiance Fields (NeRFs) by converting them into a sparse octree representation [S1, unofficial]. The method leverages the differentiability of the volume rendering equation to refine the octree attributes using standard gradient descent [S1, unofficial].

## Sources
[S1] PlenOctrees For Real-time Rendering of Neural Radiance Fields (NeRFs), http://alexyu.net/plenoctrees/, fetched 2026-10-28 (unofficial)
[S2] NeAT: Neural Adaptive Tomography, https://doi.org/10.48550/arxiv.2202.02171, fetched 2026-10-26
