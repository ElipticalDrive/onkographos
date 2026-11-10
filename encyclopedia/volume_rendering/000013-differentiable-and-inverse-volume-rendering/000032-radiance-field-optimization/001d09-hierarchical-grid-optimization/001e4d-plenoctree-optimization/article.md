# PlenOctree Optimization

Also called: Structure Refinement (Octree Optimization)

## Overview
PlenOctree Optimization is a post-training refinement process that finetunes a converted PlenOctree on original training images using Stochastic Gradient Descent (SGD) to improve image quality [S1, unofficial]. It solves the residual approximation errors introduced during the conversion from a neural network to a sparse voxel grid [S1, unofficial]. It builds on the differentiable volume rendering operator and analytic derivatives of the rendering equation to update leaf attributes directly [S1360].

## How it works
**Inputs and Outputs:**
*   **Input:** A converted PlenOctree containing density $\sigma$ and spherical harmonic (SH) coefficients for Red, Green, and Blue channels at each leaf node [S1, unofficial].
*   **Input:** Training images and corresponding camera poses [S1, unofficial].
*   **Output:** Optimized PlenOctree attributes (density and SH coefficients) [S1, unofficial].

**Data Structures:**
The PlenOctree is a sparse octree. Each leaf node stores:
*   Density $\sigma$ (scalar).
*   SH coefficients for Red, Green, and Blue channels (vectors) [S1, unofficial].
*   The tree structure is fixed during optimization; only the attribute values change [S1, unofficial].

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
where $\mathbf{C}(\mathbf{r})$ is the ground truth color and $\hat{\mathbf{C}}(\mathbf{r})$ is the rendered color [S1, unofficial]. Sparsity loss is not applied because the octree is already sparse [S1, unofficial].

**Steps:**
1.  **Initialize:** Start with the PlenOctree converted from the NeRF-SH model [S1, unofficial].
2.  **Iterate:** For each epoch up to the maximum limit:
    a. Sample rays from the training set [S1, unofficial].
    b. Render $\hat{\mathbf{C}}(\mathbf{r})$ using the current PlenOctree attributes [S1, unofficial].
    c. Compute $L_{RGB}$ [S1, unofficial].
    d. Compute gradients of $L_{RGB}$ with respect to the leaf attributes using analytic derivatives [S1, unofficial].
    e. Update attributes using SGD with the specified learning rate [S1, unofficial].
3.  **Early Stopping:** Monitor PSNR on the validation set for Tanks&Temples. Stop optimization if PSNR does not improve [S1, unofficial].
4.  **Storage:** Convert optimized attributes to Float16 [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def plenoctree_optimization(plenoctree, training_data, dataset_type, max_epochs):
    # Set learning rate and epochs based on dataset
    if dataset_type == "NeRF-synthetic":
        lr = 1e7
        max_epochs = 80
        val_set = None
    elif dataset_type == "Tanks&Temples":
        lr = 1.5e6
        max_epochs = 40
        # Hold out 10% for validation
        split_idx = int(len(training_data) * 0.9)
        train_set = training_data[:split_idx]
        val_set = training_data[split_idx:]
    else:
        raise ValueError("Unknown dataset type")

    best_psnr = -1.0
    for epoch in range(max_epochs):
        # Sample rays from training set
        rays = [(ray, gt_color) for ray, gt_color in train_set]
        
        # Render using current PlenOctree attributes
        # Uses analytic derivatives implemented in custom CUDA kernels [S1, unofficial]
        rendered_colors = [render_ray(plenoctree, ray) for ray, _ in rays]
        
        # Compute loss
        loss = sum((gt - pred)**2 for gt, pred in zip([gt for _, gt in rays], rendered_colors))
        
        # Compute gradients and update attributes (SGD)
        gradients = compute_gradients(loss, plenoctree.attributes)
        for attr in plenoctree.attributes:
            attr -= lr * gradients[attr]
        
        # Early stopping check
        if val_set is not None:
            val_rays = [(ray, gt_color) for ray, gt_color in val_set]
            val_rendered = [render_ray(plenoctree, ray) for ray, _ in val_rays]
            val_loss = sum((gt - pred)**2 for gt, pred in zip([gt for _, gt in val_rays], val_rendered))
            val_psnr = compute_psnr(val_loss, len(val_rays))
            if val_psnr > best_psnr:
                best_psnr = val_psnr
                save_checkpoint(plenoctree)
            else:
                break

    # Convert to Float16 for storage
    plenoctree.attributes = convert_to_float16(plenoctree.attributes)
    return plenoctree

def render_ray(plenoctree, ray):
    # Standard volume rendering integration along ray
    samples = sample_along_ray(ray, plenoctree)
    return volume_render(samples)

def compute_gradients(loss, attributes):
    # Standard backpropagation through the rendering graph
    return {attr: grad(loss, attr) for attr in attributes}

def compute_psnr(mse, num_samples):
    return 10 * (1.0 / mse) ** 0.5

def save_checkpoint(plenoctree):
    # Store current attributes
    pass

def convert_to_float16(attributes):
    # Cast attribute tensors to float16
    return {attr: attr.float16() for attr in attributes}
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

## Background
PlenOctree Optimization was introduced in the PlenOctrees paper [S1, unofficial]. It was developed to address the slow rendering speed of Neural Radiance Fields (NeRFs) by converting them into a sparse octree representation [S1, unofficial]. The method leverages the differentiability of the volume rendering equation to refine the octree attributes using standard gradient descent [S1, unofficial].

## Sources
[S1] PlenOctrees For Real-time Rendering of Neural Radiance Fields (NeRFs), http://alexyu.net/plenoctrees/, fetched 2026-10-28 (unofficial)
[S2] NeAT: Neural Adaptive Tomography, https://doi.org/10.48550/arxiv.2202.02171, fetched 2026-10-26
