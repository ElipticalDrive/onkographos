# VRP-UDF Training & Rendering Setup

Also called: UDF-Gaussian Joint Training, Volume Rendering Prior Network

## Overview
VRP-UDF is a method that integrates a neural network-based volume rendering prior into the 3D Gaussian Splatting pipeline to learn Unsigned Distance Functions (UDFs) and guide optimization by providing dense queries around the UDF zero-level set [S1]. It distinguishes itself from standard per-scene optimization by using a learned prior to resolve ambiguities in thin structures and surface reconstruction, addressing the inaccuracies caused by random hierarchical sampling and ill-defined UDF fields near surfaces common in volume rendering frameworks [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Multi-view images, camera parameters, and ground truth UDFs/depths for training [S1].
*   **Outputs:** Optimized 3D Gaussian primitives $\{\mathbf{\mu}_i, \mathbf{R}_i, \mathbf{s}_i, \alpha_i, \mathbf{c}_i\}$ and a learned UDF field $f_u$ [S1].

**Data Structures:**
The method employs four Multi-Layer Perceptrons (MLPs) with 256 hidden units:
1.  **$f_{w1}$:** 3 layers.
2.  **$f_{w2}$:** 6 layers with skip connections.
3.  **$f_u$ (UDF function):** 8 layers with skip connections.
4.  **$f_c$ (Color function):** 2 layers.

**Numbered Steps:**
1.  **Network Initialization:** Initialize $f_{w1}, f_{w2}, f_u,$ and $f_c$ with 256 hidden units as specified above [S1].
2.  **UDF Activation:** Apply the Softplus activation function after the final layer of $f_u$ to enforce non-negative outputs, ensuring the output represents a valid unsigned distance [S1].
3.  **Color Network Input:** Construct the input for the color network $f_c$ using spatial coordinates and features, but explicitly **exclude** normal vectors. Normals are omitted because they are ambiguous at the zero-level set due to the non-differentiability of UDFs [S1].
4.  **Training Schedule:**
    *   Train the Gaussian Splatting method for a total of 30,000 iterations [S1].
    *   Begin training the UDF networks ($f_u$) starting from iteration 9,000 [S1].
5.  **Refinement Phase:** During the final 10,000 iterations (iterations 20,000–30,000), refine the Gaussians by casting rays. Use a batch size of 2,048 rays, sampling 64 points along each ray [S1].
6.  **Loss Computation:** Compute rendering errors using depth L1 distance, mask errors with cross-entropy and L1 distance, and "Peak Diff-L1" (depth discrepancy between the sampled point with maximum rendering weight and the true surface) [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_vrp_udf_gaussian_splatting(images, cameras, gt_udfs, gt_depths):
    # Initialize MLPs with 256 hidden units
    fw1 = MLP(layers=3, hidden=256)
    fw2 = MLP(layers=6, hidden=256, skip_connections=True)
    fu = MLP(layers=8, hidden=256, skip_connections=True)
    fc = MLP(layers=2, hidden=256)
    
    # Initialize Gaussian primitives
    gaussians = initialize_gaussians(images)
    
    # Training loop
    for iteration in range(1, 30001):
        
        # Update learning rates for fu and fc based on schedulers from [1, 24]
        update_learning_rate(fu, fc, iteration)
        
        if iteration >= 9000:
            # Train UDF network
            # Sample points and compute UDF loss using Softplus activation on fu output
            udf_loss = compute_udf_loss(fu, fw1, fw2, gt_udfs)
            loss += udf_loss
            
        # Render Gaussians
        rendered_images, rendered_depths = render_gaussians(gaussians, cameras)
        
        # Compute rendering losses
        depth_loss = l1_loss(rendered_depths, gt_depths)
        mask_loss = cross_entropy_loss(rendered_images, images) + l1_loss(rendered_images, images)
        loss += depth_loss + mask_loss
        
        # Optimization step
        optimize(gaussians, fu, fc, fw1, fw2, loss)
        
        # Refinement phase: Final 10,000 iterations
        if iteration > 20000:
            # Cast rays with specific batch size and sampling
            rays = cast_rays(batch_size=2048)
            sampled_points = sample_along_rays(rays, num_points=64)
            
            # Query UDF and Color for refinement
            udf_values = fu(sampled_points)
            # Note: Normals are NOT passed to fc
            colors = fc(sampled_points, features) 
            
            # Compute refinement loss (e.g., Peak Diff-L1)
            refinement_loss = compute_peak_diff_l1(sampled_points, udf_values, gt_depths)
            loss += refinement_loss
            optimize(gaussians, fu, fc, fw1, fw2, loss)

def compute_udf_loss(fu, fw1, fw2, gt_udfs):
    # Forward pass through UDF network
    # Apply Softplus to ensure non-negative output
    udf_pred = fu(fw1(...), fw2(...))
    udf_pred = softplus(udf_pred)
    return l1_loss(udf_pred, gt_udfs)
```

**Parameters:**
*   **MLP Hidden Units:** 256 for all networks [S1].
*   **MLP Layers:** $f_{w1}$ (3), $f_{w2}$ (6), $f_u$ (8), $f_c$ (2) [S1].
*   **Total Iterations:** 30,000 [S1].
*   **UDF Start Iteration:** 9,000 [S1].
*   **Refinement Batch Size:** 2,048 rays [S1].
*   **Points per Ray:** 64 [S1].
*   **Learning Rate/Scheduler:** Follows previous works [1, 24] (exact values not specified in the source section) [S1].

**Edge Cases and Pitfalls:**
*   **Ill-defined UDF Fields:** Random hierarchical sampling and ill-defined UDF fields near surfaces cause rendering inaccuracies. This is a common issue across volume rendering frameworks, even in theoretically unbiased mappings like NeuS [S1].
*   **Convergence Parameters:** Excessively large convergence parameters (e.g., NeuralUDF's flip factor) can cause premature flipping before the true surface, leading to depth rendering errors. A larger factor does not guarantee higher accuracy [S1].
*   **Normal Ambiguity:** Using normal vectors as input to the color network $f_c$ is detrimental because normals are ambiguous at the zero-level set due to UDF non-differentiability [S1].

## Operating characteristics
*   **Accuracy:** Achieves the best accuracy among compared renderers for UDFs and SDFs on ShapeNet and real scans, particularly excelling in complex indoor scenes and thin structures (e.g., eggshells, sails) where other UDF-based methods struggle [S1].
*   **Limitations:** Despite being unbiased in training, practical rendering inaccuracies persist due to sampling issues near surfaces. It may not fully capture extremely thin structures in a complete manner without advanced UDF extraction techniques [S1].
*   **Performance:** The method involves joint training of Gaussian Splatting and neural networks, requiring significant iterations (30k) and specific refinement phases [S1].

## Background
The VRP-UDF method was introduced to address limitations in learning Unsigned Distance Functions from multi-view images. It builds upon volume rendering priors and extends them to Gaussian Splatting reconstruction. The work is cited as [S1] VRP-UDF: Toward Unbiased Learning of Unsigned Distance Functions From Multi-View Images With Volume Rendering Priors. It compares against methods like NeUDF, VRPrior, and NeuS, highlighting improvements in recovering complete and smooth surfaces [S1].

## Sources
[S1] VRP-UDF: Toward Unbiased Learning of Unsigned Distance Functions From Multi-View Images With Volume Rendering Priors., https://doi.org/10.1109/tpami.2026.3653901, fetched 2026-10-27
