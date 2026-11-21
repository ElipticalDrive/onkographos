# Hybrid Training for Explicit Primitives

Also called: Hybrid training of convexes, Two-stage hybrid training scheme

## Overview
Hybrid training is a two-stage optimization procedure for explicit primitive-based scene representations that combines vanilla geometric primitives with appearance embeddings to balance geometric fidelity, photometric quality, and parameter efficiency [S1, unofficial]. It sets itself apart from single-stage training by decoupling geometry initialization from appearance modeling, allowing the model to first establish a robust structural baseline before adapting to lighting variations [S1, unofficial]. This approach is specifically designed for space robotics applications where compactness and safety-critical robustness are prioritized over pure photometric fidelity [S1, unofficial].

## How it works
The method trains explicit convex primitives (CS) using a staged approach to mitigate clutter and improve geometric accuracy [S1, unofficial].

**Inputs and Outputs**
*   **Input:** A set of $N$ training images with known camera poses.
*   **Output:** A set of convex primitives defined by their geometric parameters (e.g., position, orientation, scale) and, in Stage 2, appearance embeddings.

**Data Structures**
*   **Convex Primitives:** Each primitive $i$ is defined by a set of geometric parameters $\theta_i$. In the vanilla stage, these parameters define the shape and position. In the hybrid stage, an additional appearance embedding vector $e_i$ is associated with each primitive to model lighting and material variations [S1, unofficial].
*   **Appearance Embeddings:** A learnable vector per primitive that substitutes for missing data in low-data regimes by explaining lighting variations [S1, unofficial].

**Numbered Steps**
1.  **Stage 1: Vanilla Geometry Training**
    *   Initialize a set of convex primitives with geometric parameters only.
    *   Optimize these parameters for 30,000 iterations using a photometric loss (e.g., L1 or SSIM) between rendered images and ground truth images [S1, unofficial].
    *   No appearance embeddings are used during this stage. The goal is to learn efficient geometry without the interference of appearance modeling [S1, unofficial].

2.  **Stage 2: Hybrid Appearance Training**
    *   Introduce appearance embeddings $e_i$ for each primitive.
    *   Continue training for an additional 30,000 iterations [S1, unofficial].
    *   Optimize both the geometric parameters (from Stage 1) and the new appearance embeddings to minimize the photometric loss [S1, unofficial].
    *   The appearance embeddings help explain lighting variations that the vanilla geometry cannot capture [S1, unofficial].

**Parameters**
*   **Stage 1 Iterations:** 30,000 [S1, unofficial].
*   **Stage 2 Iterations:** 30,000 [S1, unofficial].
*   **Total Iterations:** 60,000 [S1, unofficial].
*   **Appearance Embeddings:** Their utility is highest in low-data regimes (e.g., 20 images) where they act as a substitute for missing data [S1, unofficial]. As the training set size grows (e.g., 500 images), the dataset itself absorbs much of the variation, and the utility of embeddings for geometry learning diminishes [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hybrid_training(primitives, images, poses, num_stage1_iters=30000, num_stage2_iters=30000):
    """
    Two-stage hybrid training for explicit convex primitives.
    
    Args:
        primitives: List of convex primitive objects with geometric parameters.
        images: List of training images.
        poses: List of camera poses corresponding to images.
        num_stage1_iters: Number of iterations for vanilla geometry training.
        num_stage2_iters: Number of iterations for hybrid appearance training.
        
    Returns:
        primitives: Optimized primitives with geometry and appearance embeddings.
    """
    # Stage 1: Train vanilla convexes for geometry
    for step in range(num_stage1_iters):
        # Render images using current primitive geometry
        rendered_images = render(primitives, images, poses)
        
        # Compute photometric loss (e.g., L1 or SSIM)
        loss = compute_photometric_loss(rendered_images, images)
        
        # Backpropagate to update geometric parameters only
        loss.backward()
        optimizer.step()
        optimizer.zero_grad()
        
    # Stage 2: Introduce appearance embeddings and continue training
    for prim in primitives:
        prim.add_appearance_embedding()  # [G] Initialize embedding vector, e.g., with zeros or small random values
        
    for step in range(num_stage2_iters):
        # Render images using geometry and appearance embeddings
        rendered_images = render_with_appearance(primitives, images, poses)
        
        # Compute photometric loss
        loss = compute_photometric_loss(rendered_images, images)
        
        # Backpropagate to update both geometric parameters and appearance embeddings
        loss.backward()
        optimizer.step()
        optimizer.zero_grad()
        
    return primitives
```

**Edge Cases and Pitfalls**
*   **Low-Data Regimes:** Appearance embeddings are critical when training data is scarce (e.g., 20 images) to handle lighting variations [S1, unofficial].
*   **High-Data Regimes:** With abundant data (e.g., 500 images), the benefit of appearance embeddings diminishes as the dataset absorbs variation, and their effect becomes less predictable [S1, unofficial].
*   **Clutter:** Vanilla convexes may repurpose noise primitives for geometry, while wild convexes (with embeddings) may reduce clutter but require careful tuning [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The method requires 60,000 iterations total (30k for Stage 1, 30k for Stage 2) [S1, unofficial].
*   **Memory Complexity:** The parameter count is compact. On the SPEED+ dataset, the hybrid model uses 137,034 parameters (±1,986) [S1, unofficial].
*   **Measured Performance (SPEED+ Dataset):**
    *   PSNR: 23.12
    *   SSIM: 0.85
    *   LPIPS: 0.49
    *   IoU: 0.60
    *   TPR: 1.00
    *   FPR: 0.11
    *   FDR: 0.40
    *   Parameters: 137,034 (±1,986) [S1, unofficial].
*   **Data Efficiency:**
    *   With 20 training images: IoU 0.44, PSNR 18.69, Params 197,961 [S1, unofficial].
    *   With 500 training images: IoU 0.78, PSNR 24.95, Params 232,185 [S1, unofficial].
*   **Strengths:** High geometric fidelity (IoU 0.60, TPR 1.00), parameter efficiency (137k params), and robustness in low-data regimes [S1, unofficial].
*   **Limitations:** Potential drop in image quality compared to Gaussian Splatting (GS) in some metrics, which may affect downstream interaction tasks [S1, unofficial].
*   **When to Use:** Space robotics applications requiring compact, safe, and efficient 3D reconstructions [S1, unofficial].
*   **When Not to Use:** When pure photometric fidelity is the only priority and computational resources are not constrained [S1, unofficial].

## Background
Hybrid training for explicit primitives was introduced in the context of comparing implicit and explicit neural scene representations for space robotics [S1, unofficial]. The work highlights that explicit methods, particularly convex splatting with hybrid training, offer a flexible trade-off between efficiency, fidelity, and robustness [S1, unofficial]. The method was evaluated on the SPEED+ dataset, demonstrating its suitability for onboard deployment in space applications [S1, unofficial].

## Sources
[S1] On the Geometric Accuracy of Implicit and Primitive-based Representations Derived from View Rendering Constraints, https://doi.org/10.48550/arXiv.2509.10241, fetched 2026-10-30 (unofficial)
