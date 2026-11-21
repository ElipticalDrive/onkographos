# Joint Fine-tuning
Also called: Progressive Joint Optimization
## Overview
Joint Fine-tuning is a training strategy within the GPS-Gaussian framework that progressively co-optimizes a 3D Gaussian Splatting model and an Unsigned Distance Function (UDF) model to refine scene geometry [S1]. It addresses the ambiguity in pulling Gaussians to a surface by using dense queries around the UDF zero-level set as geometric cues, preventing error accumulation [S1]. Unlike methods that regress Gaussian parameters directly from sparse views without optimization [S2, unofficial], this approach iteratively improves both representations [S1].
## How it works
### Inputs and Outputs
**Inputs:**
*   Sparse source views (images).
*   Initial 3D Gaussian distribution (positions, scales, rotations, opacities, colors).
*   Initial UDF field (parameterized by a neural network).
*   Camera poses for source views.

**Outputs:**
*   Refined 3D Gaussian parameters.
*   Refined UDF field.

### Data Structures
*   **Gaussian Model:** Set of $N$ Gaussians. Each Gaussian $k$ has position $\mathbf{p}_k$, covariance $\mathbf{\Sigma}_k$, opacity $\alpha_k$, and color $\mathbf{c}_k$.
*   **UDF Model:** Neural network $f_{\theta}(\mathbf{x})$ predicting unsigned distance at point $\mathbf{x}$.
*   **Ray $V_k$:** Ray passing through Gaussian $k$'s center.

### Numbered Steps
1.  **Initialization:** Initialize the Gaussian model and UDF model. The UDF is not perfectly accurate initially [S1].
2.  **Progressive Joint Optimization Loop:**
    a.  **Ray Casting:** For each Gaussian $k$, define a ray $V_k$ passing through its center [S1].
    b.  **Depth Projection:** Calculate $d_t$, the depth of the projection point from Gaussian $k$ onto $V_k$ [S1].
    c.  **UDF Query:** Query the UDF at the Gaussian's position to get $d_n$, the depth of the surface point $\{q_n\}$ along the ray direction [S1].
    d.  **Loss Calculation:** Compute a geometric consistency loss. The source mentions a stop-gradient operation $sg(\cdot)$ applied to the UDF prediction to stabilize training [S1]. The loss pulls the Gaussian towards the UDF surface using dense queries around the zero-level set rather than a direct pull to the zero-level set itself [S1].
    e.  **Gradient Update:** Update both the Gaussian parameters and the UDF network parameters $\theta$ using the combined loss. The stop-gradient $sg(\cdot)$ is applied to the UDF term to prevent conflicting gradients [S1].
    f.  **Iteration:** Repeat steps 2a-2e for multiple epochs. The process is "progressive," meaning the refinement happens over time, continuously improving each model [S1].

### Formulas
The geometric loss leverages the depth difference between the Gaussian projection and the UDF surface. Let $d_t$ be the depth of the Gaussian's projection onto the ray $V_k$ and $d_n$ be the depth of the surface point $\{q_n\}$ predicted by the UDF. The update involves a stop-gradient operation $sg(\cdot)$ on the UDF term [S1]:
$$ \mathcal{L}_{geo} = \| d_t - sg(d_n) \|^2 $$
(Note: The exact loss function formula is not fully provided in the source text, but the components $d_t$, $d_n$, and $sg(\cdot)$ are explicitly defined [S1].)

### Parameters
*   **Progressive Schedule:** Not given in the sources.
*   **Learning Rate:** Not given in the sources.

### Edge Cases and Pitfalls
*   **Ambiguous Pulling Directions:** Directly pulling Gaussians to the UDF zero-level set can cause error accumulation due to ambiguous directions [S1]. This method avoids this by using dense queries around the zero-level set [S1].
*   **UDF Accuracy:** The UDF is not perfectly accurate, but provides reliable optimization directions [S1]. Poor initial UDF quality may slow convergence.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Joint Fine-tuning for GPS-Gaussian
# Inputs: gaussians (list of Gaussian objects), udf_net (neural network), 
#         source_views (list of images), cameras (list of camera poses)
# Outputs: Refined gaussians, Refined udf_net

def joint_fine_tuning(gaussians, udf_net, source_views, cameras, epochs):
    for epoch in range(epochs):
        for gaussian in gaussians:
            # Step 1: Define ray Vk through Gaussian center
            ray = get_ray_through_gaussian_center(gaussian, cameras)
            
            # Step 2: Get depth of Gaussian projection dt
            dt = gaussian.depth_along_ray(ray)
            
            # Step 3: Query UDF for surface depth dn
            # The UDF predicts unsigned distance. We assume dn is derived from 
            # the UDF zero-level set intersection along the ray.
            dn = udf_net.get_surface_depth(ray)
            
            # Step 4: Compute geometric loss with stop-gradient on UDF term
            # sg() denotes stop-gradient operation
            loss = (dt - stop_gradient(dn)) ** 2
            
            # Step 5: Backpropagate to update both Gaussian and UDF
            loss.backward()
            
        # Step 6: Optimizer step for both models
        optimizer.step()
        optimizer.zero_grad()
        
    return gaussians, udf_net

def stop_gradient(tensor):
    # Returns tensor detached from computation graph
    return tensor.detach()
```

## Subtopics

### [FaCT-GS](10199-fact-gs.md)
A method for compressing 3D Gaussian Splatting models. It focuses on reducing memory footprint through factorization, unlike Joint Fine-tuning which focuses on geometric refinement via UDFs.

### [SegAnySplatting Feature Optimization](12209-seganysplatting-feature-optimization.md)
Optimizes features for segmentation-aware Gaussian Splatting. It targets semantic feature learning rather than geometric refinement through joint UDF optimization.

## Operating characteristics
*   **Time Complexity:** Higher than non-optimizing methods like the initial GPS-Gaussian regression [S2, unofficial] because it involves iterative joint optimization of Gaussians and a UDF network [S1].
*   **Memory Complexity:** Requires storing both the Gaussian parameters and the UDF network parameters [S1].
*   **Strengths:** Prevents error accumulation from ambiguous pulling directions by using dense queries around the zero-level set [S1]. Continuously improves geometry over time [S1].
*   **Limitations:** Computationally expensive due to joint optimization [S1]. Relies on the UDF providing reliable directions despite not being perfectly accurate [S1].
*   **When to Use:** When high-fidelity geometry is required and sparse views are available, and computational resources allow for iterative optimization [S1].
*   **When Not to Use:** When real-time inference without fine-tuning is required, as this method is a training-time optimization [S2, unofficial].

## Background
Joint Fine-tuning, also known as Progressive Joint Optimization, was introduced in the VRP-UDF paper [S1]. It builds on the GPS-Gaussian framework [S2, unofficial], which initially introduced Gaussian parameter maps on sparse source views to regress parameters jointly with a depth estimation module without fine-tuning [S2, unofficial]. The Joint Fine-tuning method extends this by adding a progressive joint optimization step to refine the geometry further [S1]. It addresses limitations in existing methods that directly pull Gaussians to the UDF zero-level set [S1].

## Sources
[S1] VRP-UDF: Toward Unbiased Learning of Unsigned Distance Functions From Multi-View Images With Volume Rendering Priors., https://doi.org/10.1109/tpami.2026.3653901, fetched 2026-10-27
[S2] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
