# Explicit Neural Feature Volume

Also called: Explicit Neural Feature Tensor

## Overview
Explicit Neural Feature Volume stores neural features as a large, directly addressable tensor, serving as the primary data structure for neural adaptive tomography [S1]. It solves the problem of balancing reconstruction fidelity with memory efficiency in limited-angle tomography, where hybrid explicit-implicit models suffer from increased blur and training costs [S1]. It differs from hybrid approaches like ACORN by avoiding implicit compression of feature volumes, which otherwise worsens PSNR in the missing wedge direction [S1].

## How it works
**Inputs and Outputs:**
*   Input: A set of 2D projection images (sinograms) and their corresponding camera poses.
*   Output: A 3D density volume reconstructed via volume rendering.

**Data Structures:**
The core data structure is an explicit tensor storing feature vectors.
*   **Feature Tensor:** A 3D grid where each cell contains a fixed-size feature vector.
*   **Feature Vector Size:** 8 elements (floats) [S1].
*   **Grid Resolution:** Varies by configuration; a typical high-quality configuration uses a resolution of $8 \times 173 \times 173$ (or similar anisotropic scaling) [S1].
*   **Sparse Representation:** The tensor is often organized hierarchically (e.g., octree) to allow empty space skipping and structural refinement (merging/splitting/culling leaf nodes) [S1].

**Steps:**
1.  **Initialization:** Initialize the explicit feature tensor with random or zero values. The grid resolution is chosen to balance variable count with detail [S1].
2.  **Ray Casting:** For each pixel in a target view, cast a ray through the 3D space.
3.  **Feature Sampling:** Sample the feature tensor at points along the ray. If using a sparse/octree structure, traverse the tree to find active nodes [S1].
4.  **Decoding (Density Activation):** Pass the sampled 8-element feature vector through a small decoder network to produce a single scalar density value $\sigma$ [S1].
    *   *Note:* The decoder acts as a regularizer. Without it, blurry artifacts appear similar to iterative reconstruction methods [S1].
5.  **Volume Rendering:** Compute the accumulated color and opacity along the ray using standard volume rendering integrals, weighted by the decoded density $\sigma$ [S1].
6.  **Loss Computation:** Compare the rendered image against the ground truth projection.
7.  **Optimization:** Update the feature tensor values and decoder network weights using gradient descent.
8.  **Structural Refinement:** Periodically (every few epochs), perform octree optimization: merge, split, and cull leaf nodes to optimize reconstruction time and memory [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Simplified forward pass for a single ray
def render_ray(ray_origin, ray_direction, feature_grid, decoder_network):
    # 1. Traverse the sparse grid (octree) along the ray
    # Returns a list of (position, feature_vector) tuples
    samples = feature_grid.sample_along_ray(ray_origin, ray_direction)
    
    accumulated_color = 0.0
    accumulated_alpha = 0.0
    T = 1.0  # Transmittance
    
    for pos, feature_vec in samples:
        # 2. Decode feature vector to density
        # feature_vec is size 8
        density = decoder_network(feature_vec)
        
        # 3. Volume rendering step (simplified)
        # alpha = 1 - exp(-density * step_size)
        alpha = 1.0 - exp(-density * 0.1) # step_size assumed 0.1
        
        color = compute_color(pos, feature_vec) # Color might be separate or derived
        
        # 4. Composite
        weight = T * alpha
        accumulated_color += weight * color
        accumulated_alpha += weight
        T *= (1.0 - alpha)
        
        if T < 1e-5:
            break
            
    return accumulated_color
```

**Parameters:**
*   **Feature Vector Dimension:** 8 [S1].
*   **Grid Resolution:** $8 \times 173 \times 173$ (with decoder) vs $1 \times 333 \times 333$ (without decoder) [S1]. Note: Both configurations use the same total number of variables [S1].
*   **Refinement Frequency:** Every few epochs [S1].

**Edge Cases and Pitfalls:**
*   **Missing Wedge Blur:** In limited-angle tomography, reconstructions are naturally blurred in the direction orthogonal to the missing viewing direction [S1].
*   **Hybrid Model Degradation:** Using a hybrid explicit-implicit model (compressing features into an implicit network) worsens PSNR in limited-angle scenarios because the implicit network's regularization encourages additional blur instead of repairing the missing wedge artifacts [S1].
*   **Memory Consumption:** Hybrid models have higher *intermediate* memory consumption during training because all feature volumes must be decoded to trace rays for a single view, despite potentially smaller final model size [S1].
*   **Training Time:** Hybrid models dramatically increase training times [S1].
*   **Blur without Decoder:** Omitting the decoder network leads to blurry artifacts [S1].

## Subtopics

### [Density Activation](2421-density-activation.md)
Density Activation is the specific mechanism within the Explicit Neural Feature Volume pipeline that converts the raw 8-element feature vector into a valid, non-negative density value. It differs from the parent method by focusing solely on the final non-linear mapping step rather than the storage and retrieval of the feature tensor. It is chosen when regularization is needed to prevent blurry artifacts in difficult reconstruction conditions (e.g., limited angles). The decoder network typically transforms the 8-element feature vector into a single density value. A key implementation detail is that the variant *with* the decoder uses a grid resolution of $8 \times 173$, while the variant *without* the decoder uses $1 \times 333$, maintaining the same total variable count. Without this activation/decoder step, blurry artifacts similar to those in iterative reconstruction methods appear.

## Operating characteristics
*   **Time Complexity:** Training time is lower than hybrid explicit-implicit models because it avoids the overhead of decoding an implicit network for every ray [S1]. Structural refinement (octree operations) adds overhead but reduces reconstruction time via empty space skipping [S1].
*   **Memory Complexity:** Intermediate memory consumption is lower than hybrid models because features are stored explicitly and do not require decoding an implicit network for ray tracing [S1]. Final model size is larger than hybrid models, but this is a trade-off for better reconstruction quality [S1].
*   **Performance:** Achieves higher PSNR than hybrid explicit-implicit models, particularly in limited-angle tomography [S1].
*   **Strengths:** Better reconstruction quality in limited-angle scenarios; avoids the "blur encouragement" of implicit regularizers; lower intermediate memory usage during training compared to hybrids.
*   **Limitations:** Higher final model size compared to hybrid approaches; requires careful tuning of grid resolution and decoder architecture.
*   **When to Use:** Limited-angle tomography where hybrid models fail to repair missing wedge blur; scenarios where intermediate training memory is a constraint.
*   **When Not to Use:** When final model compactness is the absolute priority and limited-angle artifacts are not a concern; when training time is extremely constrained (though hybrids are slower).

## Background
The Explicit Neural Feature Volume method was introduced in the context of Neural Adaptive Tomography (NeAT) [S1]. It addresses limitations in hybrid explicit-implicit representations like ACORN, which were found to degrade performance in limited-angle tomography [S1]. The method emphasizes the benefits of explicit storage for feature tensors to maintain high-frequency details and avoid over-regularization [S1].

## Sources
[S1] NeAT: Neural Adaptive Tomography, https://doi.org/10.48550/arxiv.2202.02171, fetched 2026-10-26
