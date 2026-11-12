# PlenOctree Optimization

Also called: Structure Refinement (Octree Optimization)

## Overview
PlenOctree Optimization is a post-processing stage that refines a pre-converted PlenOctree representation by optimizing its leaf node parameters (density and spherical harmonics coefficients) against original training images using stochastic gradient descent (SGD) [S1, unofficial]. It solves the issue of approximation errors introduced during the initial conversion from a neural network to a sparse voxel grid, thereby improving final image quality without altering the tree structure [S1, unofficial]. Unlike [PlenOctree Conversion](1359-plenoctree-conversion.md), which focuses on the construction and tabulation of the view-dependent volume, this method focuses on the differentiable optimization of the already-sparse structure [S1, unofficial]. It is distinct from [Structure Refinement](929-plenoxels-optimization-configuration.md) in that it operates on a fixed tree topology, whereas the latter dynamically merges, splits, and culls nodes [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A converted PlenOctree containing density $\sigma$ and spherical harmonics (SH) coefficients for color $\mathbf{c}$ at each leaf node; the original training dataset (images and camera poses); validation subset (optional, for early stopping).
*   **Output:** Optimized PlenOctree with updated leaf node parameters.

**Data Structures:**
The PlenOctree is a sparse octree. Each leaf node stores:
*   Density $\sigma$ (scalar).
*   Color coefficients: A vector of SH coefficients for each RGB channel. The dimension depends on the SH degree used during conversion (typically degree 3 or 4 in NeRF-SH).
*   [G] The tree structure (parent pointers, child indices) remains fixed during this optimization phase.

**Parameters:**
*   **Learning Rate:**
    *   NeRF-synthetic dataset: $1 \times 10^{-7}$ [S1, unofficial].
    *   Tanks&Temples dataset: $1.5 \times 10^{-6}$ [S1, unofficial].
*   **Maximum Epochs:**
    *   NeRF-synthetic: 80 [S1, unofficial].
    *   Tanks&Temples: 40 [S1, unofficial].
*   **Optimization Precision:** Float32 [S1, unofficial].
*   **Storage Precision:** Float16 (applied after optimization) [S1, unofficial].
*   **Validation Set:** For Tanks&Temples, 10% of the training set is held out [S1, unofficial].

**Loss Function:**
The optimization minimizes the mean squared error between the rendered color $\hat{C}(\mathbf{r})$ and the ground truth color $C(\mathbf{r})$ [S1, unofficial]:

$$
\mathcal{L} = \sum_{\mathbf{r} \in \mathcal{R}} \| C(\mathbf{r}) - \hat{C}(\mathbf{r}) \|_2^2
$$

where $\mathcal{R}$ is the set of sampled rays. The rendering equation $\hat{C}(\mathbf{r})$ is computed using the standard emission-absorption model defined in the parent article. Note that no sparsity loss is applied because the octree is already sparse [S1, unofficial].

**Steps:**
1.  **Initialize:** Load the converted PlenOctree and the training dataset.
2.  **Setup Optimizer:** Configure an SGD optimizer with the specified learning rate for the dataset type.
3.  **Iterate:** For each epoch up to the maximum limit:
    a.  Sample a batch of rays from the training views.
    b.  Render the rays using the current PlenOctree parameters. This involves traversing the octree and evaluating the SH coefficients for the viewing direction.
    c.  Compute the loss $\mathcal{L}$ using the formula above.
    d.  Compute gradients. The analytic derivatives of the volume rendering equation with respect to $\sigma$ and $\mathbf{c}$ are used [S1, unofficial]. These are typically implemented in custom CUDA kernels for efficiency [S1, unofficial].
    e.  Update the leaf node parameters ($\sigma$ and SH coefficients) using the SGD step.
4.  **Early Stopping:** Monitor the PSNR on the validation set (if available). Stop optimization if the validation PSNR does not improve [S1, unofficial].
5.  **Finalize:** Convert the optimized parameters to Float16 for storage to reduce model size [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def plenoctree_optimization(plenoctree, training_data, validation_data, dataset_type):
    # 1. Setup parameters
    if dataset_type == "nerf_synthetic":
        lr = 1e-7
        max_epochs = 80
        use_validation = False
    elif dataset_type == "tanks_temples":
        lr = 1.5e-6
        max_epochs = 40
        use_validation = True
        # Hold out 10% of training_data for validation
        validation_data = split_validation(training_data, ratio=0.1)
    
    # 2. Initialize optimizer
    # Parameters are the leaf node sigma and SH coefficients
    optimizer = SGD(parameters=plenoctree.leaf_params, lr=lr)
    
    best_val_psnr = -infinity
    patience_counter = 0
    
    # 3. Optimization loop
    for epoch in range(max_epochs):
        # Sample batch of rays from training data
        rays = sample_rays(training_data)
        
        # Forward pass: Render rays
        # Uses analytic derivatives for gradient computation
        rendered_colors = render_plenoctree(plenoctree, rays)
        ground_truth_colors = get_ground_truth(rays)
        
        # Compute Loss
        loss = mean_squared_error(rendered_colors, ground_truth_colors)
        
        # Backward pass
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
        # Early stopping check
        if use_validation:
            val_rays = sample_rays(validation_data)
            val_rendered = render_plenoctree(plenoctree, val_rays)
            val_gt = get_ground_truth(val_rays)
            val_psnr = compute_psnr(val_rendered, val_gt)
            
            if val_psnr > best_val_psnr:
                best_val_psnr = val_psnr
                patience_counter = 0
                save_best_model(plenoctree)
            else:
                patience_counter += 1
                # Note: Specific patience value not given in sources [G]
                if patience_counter > 10: 
                    break
        else:
            # For NeRF-synthetic, just run for max epochs
            pass
            
    # 4. Finalize
    # Convert to float16 for storage
    plenoctree.convert_to_float16()
    return plenoctree
```

**Edge Cases and Pitfalls:**
*   **Precision:** Optimization must be performed in Float32 to ensure stability [S1, unofficial].
*   **Sparsity Loss:** Do not apply sparsity regularization (e.g., $L_1$ on density) as the tree structure is already sparse and fixed [S1, unofficial].
*   **Validation Set:** For Tanks&Temples, a validation set is required for early stopping; for NeRF-synthetic, it is not used [S1, unofficial].

## Subtopics

### [PlenOctree Conversion](1359-plenoctree-conversion.md)
The preceding step that creates the initial sparse octree from a NeRF-SH model. It focuses on construction and tabulation rather than optimization.

### [Analytic Derivatives of PlenOctree Rendering](1360-analytic-derivatives-of-plenoctree-rendering.md)
The mathematical foundation for the gradients used in this optimization. It provides closed-form derivatives for the volume rendering equation.

### [PlenOctree Fine-tuning](1465-plenoctree-fine-tuning.md)
A related approach that also optimizes leaf parameters but is distinguished by its reliance on numerical gradient descent rather than analytic derivatives and its focus on accelerating training by 5 orders of magnitude.

## Operating characteristics
*   **Time Complexity:** The optimization is significantly faster than training a standard NeRF. It operates at approximately 3 million rays per second, compared to ~9,000 rays per second for NeRF training [S1, unofficial].
*   **Memory Complexity:** Memory usage is dominated by the storage of the PlenOctree parameters and the gradients during backpropagation. Since the tree is sparse, memory usage is proportional to the number of leaf nodes.
*   **Performance:** On a single NVIDIA V100 GPU, optimization takes approximately 10 minutes per scene [S1, unofficial].
*   **Strengths:** Improves image quality over the initial conversion; very fast convergence compared to full NeRF training; reduces model size via Float16 storage [S1, unofficial].
*   **Limitations:** Requires a pre-converted PlenOctree; does not adapt the tree structure (no splitting/merging); performance depends on the quality of the initial conversion [S1, unofficial].
*   **When to use:** When a high-quality real-time rendering representation is needed from a pre-trained NeRF-SH model and the initial conversion quality is insufficient [S1, unofficial].
*   **When not to use:** When the scene geometry is unknown or requires structural adaptation (use [Structure Refinement](929-plenoxels-optimization-configuration.md) instead); when computational resources for initial conversion are unavailable.

## Background
PlenOctree Optimization was introduced as part of the PlenOctrees method for real-time rendering of Neural Radiance Fields [S1, unofficial]. The method addresses the bottleneck of conventional NeRFs, which require dense sampling and neural network inference for every ray step [S1, unofficial]. By converting the NeRF to a sparse octree and then fine-tuning it, the method achieves rendering speeds over 3500x faster than the original NeRF with no significant loss in image quality [S1, unofficial]. The optimization process leverages analytic derivatives to enable efficient gradient-based refinement of the spherical harmonics coefficients and density values [S1, unofficial].

## Sources
[S1] PlenOctrees For Real-time Rendering of Neural Radiance Fields (NeRFs), http://alexyu.net/plenoctrees/, fetched 2026-10-28 (unofficial)
[S2] NeAT: Neural Adaptive Tomography, https://doi.org/10.48550/arxiv.2202.02171, fetched 2026-10-26
