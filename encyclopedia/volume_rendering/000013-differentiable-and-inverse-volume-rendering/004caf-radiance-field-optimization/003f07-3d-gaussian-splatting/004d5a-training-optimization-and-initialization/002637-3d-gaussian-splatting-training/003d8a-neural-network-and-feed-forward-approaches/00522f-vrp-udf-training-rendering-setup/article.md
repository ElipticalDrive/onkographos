# VRP-UDF Training & Rendering Setup

Also called: UDF-Gaussian Joint Training, Volume Rendering Prior Network

## Overview
This method integrates a neural network-based volume rendering prior that learns Unsigned Distance Functions (UDFs) to guide the optimization of 3D Gaussian Splatting [Parent](9526-neural-network-and-feed-forward-approaches.md). It solves the problem of inaccurate surface reconstruction in thin structures and depth errors caused by ill-defined UDF fields near surfaces, which are common in standard volume rendering frameworks. Unlike [MVSplat](9213-mvsplat.md), which focuses on differentiable rendering from multi-view stereo, this approach explicitly decouples the geometric prior (UDF) from the radiance field, using the UDF to constrain Gaussian placement and refinement.

## How it works
The method employs a joint training pipeline where a UDF network and a color network are trained alongside 3D Gaussians. The UDF provides a dense, unbiased geometric prior, while the Gaussians handle high-frequency appearance details.

### Inputs and Outputs
*   **Inputs:** Multi-view images, camera poses, and ground truth depth maps (or UDF values) for initialization.
*   **Outputs:** Optimized 3D Gaussian parameters (position, scale, rotation, opacity, spherical harmonics coefficients) and the trained UDF/Color networks.

### Data Structures
The core data structures are Multi-Layer Perceptrons (MLPs) and 3D Gaussian primitives.

**MLP Architectures:**
All MLPs use 256 hidden units.
1.  **$f_{w1}$ (Weight Network):** 3 layers.
2.  **$f_{w2}$ (Weight Network):** 6 layers, includes skip connections.
3.  **$f_u$ (UDF Network):** 8 layers, includes skip connections.
4.  **$f_c$ (Color Network):** 2 layers.

**3D Gaussians:**
Standard 3D Gaussian Splatting representation: mean position $\mu$, covariance $\Sigma$ (parameterized by scale $s$ and rotation $q$), opacity $\alpha$, and spherical harmonics coefficients $c_{lm}$.

### Numbered Steps

1.  **Network Initialization:**
    Initialize $f_{w1}$, $f_{w2}$, $f_u$, and $f_c$ with the specified layer counts and hidden units. Initialize 3D Gaussians from the input point cloud or sparse reconstruction.

2.  **Training Schedule:**
    *   **Phase 1 (Iterations 0–9,000):** Train the Gaussian Splatting method only. Do not activate the UDF prior.
    *   **Phase 2 (Iterations 9,000–20,000):** Begin training the UDFs ($f_u$) and Color network ($f_c$) alongside the Gaussians.
    *   **Phase 3 (Iterations 20,000–30,000):** Refine Gaussians using the UDF prior.

3.  **UDF Constraint and Activation:**
    The UDF network $f_u(\mathbf{x})$ outputs the unsigned distance to the surface. To enforce non-negativity, apply the Softplus activation function after the final layer of $f_u$:
    $$ f_u(\mathbf{x}) = \text{Softplus}(MLP_{f_u}(\mathbf{x})) $$
    where $\text{Softplus}(z) = \ln(1 + e^z)$.

4.  **Color Network Input Modification:**
    When querying the color network $f_c$, do **not** include normal vectors in the input. The input to $f_c$ consists only of the spatial coordinates $\mathbf{x}$ and potentially view direction $\mathbf{d}$.
    *   *Reason:* Normals are ambiguous at the zero-level set of a UDF due to non-differentiability. Including them introduces noise during optimization [S1].

5.  **Ray Casting and Sampling (Refinement Phase):**
    During the final 10,000 iterations:
    *   Cast rays through the camera frustum.
    *   Use a batch size of 2,048 rays per batch.
    *   Sample 64 points along each ray.
    *   Evaluate the UDF $f_u$ at these points to guide the density and position of Gaussians.

6.  **Loss Functions:**
    Minimize rendering errors between predicted images and ground truth. Metrics include:
    *   Depth L1 distance.
    *   Mask errors (Cross Entropy and L1 distance).
    *   Peak Diff-L1: The L1 error between the sampled point with maximum rendering weight and the true surface along each ray.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_vrp_udf_gaussian(num_iterations=30000, start_udf_iter=9000, 
                           refine_start_iter=20000, batch_size=2048, 
                           samples_per_ray=64):
    # Initialize networks
    f_w1 = MLP(layers=3, hidden=256)
    f_w2 = MLP(layers=6, hidden=256, skip_connections=True)
    f_u  = MLP(layers=8, hidden=256, skip_connections=True)
    f_c  = MLP(layers=2, hidden=256)
    
    # Initialize Gaussians from sparse reconstruction
    gaussians = initialize_gaussians_from_point_cloud()
    
    # Optimizers
    opt_gaussians = Adam(gaussians.parameters())
    opt_f_u = Adam(f_u.parameters())
    opt_f_c = Adam(f_c.parameters())
    
    # Learning rate schedulers for f_u and f_c follow previous works [1], [24]
    # Exact values not specified in source [S1]
    
    for iteration in range(1, num_iterations + 1):
        # Sample rays
        rays = sample_rays(batch_size=batch_size)
        
        # Sample points along rays
        points = sample_points_along_rays(rays, num_samples=samples_per_ray)
        
        # Forward pass through UDF
        # Apply Softplus to enforce non-negative UDF output
        udf_values = f_u(points)
        udf_values = softplus(udf_values)
        
        # Forward pass through Color Network
        # Note: Normals are NOT included in input to f_c
        # Inputs: spatial coordinates, view direction
        colors = f_c(points, view_directions=rays.directions)
        
        # Render image using Gaussian Splatting and UDF prior
        # The UDF guides the density/opacity of Gaussians
        rendered_image = render_gaussians_with_udf_prior(
            gaussians=gaussians, 
            rays=rays, 
            udf_values=udf_values,
            colors=colors
        )
        
        # Compute Loss
        # Ground truth images and depths are available
        loss = compute_rendering_loss(rendered_image, ground_truth_image)
        loss += compute_depth_loss(rendered_depth, ground_truth_depth)
        
        # Backpropagation
        if iteration < start_udf_iter:
            # Phase 1: Train only Gaussians
            opt_gaussians.zero_grad()
            loss.backward()
            opt_gaussians.step()
        else:
            # Phase 2 & 3: Train Gaussians, UDF, and Color Network
            opt_gaussians.zero_grad()
            opt_f_u.zero_grad()
            opt_f_c.zero_grad()
            
            loss.backward()
            
            opt_gaussians.step()
            opt_f_u.step()
            opt_f_c.step()
            
            # Refine Gaussians using UDF constraints in final phase
            if iteration >= refine_start_iter:
                refine_gaussians_with_udf(gaussians, f_u, points)

def softplus(x):
    return log(1 + exp(x))
```

### Parameters
*   **$f_{w1}$:** 3 layers, 256 hidden units.
*   **$f_{w2}$:** 6 layers, 256 hidden units, skip connections.
*   **$f_u$:** 8 layers, 256 hidden units, skip connections.
*   **$f_c$:** 2 layers, 256 hidden units.
*   **Total Iterations:** 30,000.
*   **UDF Start Iteration:** 9,000.
*   **Refinement Batch Size:** 2,048 rays.
*   **Samples per Ray:** 64.
*   **Learning Rate:** Follows previous works [1], [24] (exact values not given in [S1]).

### Edge Cases and Pitfalls
1.  **Ill-defined UDF near surfaces:** Random hierarchical sampling can cause rendering inaccuracies near surfaces. This is a common issue in volume rendering frameworks, even theoretically unbiased ones like NeuS [S1].
2.  **Premature Flipping:** Excessively large convergence parameters (e.g., NeuralUDF's flip factor) can cause the UDF to flip into an SDF before the true surface, leading to depth rendering errors [S1].
3.  **Normal Ambiguity:** Including normal vectors in the color network input causes instability because normals are undefined/ambiguous at the UDF zero-level set [S1].

## Operating characteristics
*   **Time Complexity:** Training is intensive due to the joint optimization of Gaussians and multiple MLPs. The refinement phase uses a fixed batch size of 2,048 rays with 64 samples, which is computationally heavier than standard Gaussian Splatting but necessary for UDF guidance.
*   **Memory Complexity:** Requires storing parameters for four MLPs (total ~1.5M parameters given 256 hidden units) plus the Gaussian attributes.
*   **Strengths:** Achieves best accuracy among renderers for UDFs and SDFs on ShapeNet. Outperforms other methods on complex indoor scenes (Replica) and thin structures (eggshells, sails) where other UDF-based methods struggle [S1].
*   **Limitations:** Inaccuracies still occur in real rendering due to sampling and ill-defined UDF fields near surfaces. Thin structures may not be captured completely [S1].
*   **When to use:** When high-fidelity reconstruction of thin structures and accurate depth is required, and multi-view images with ground truth depth/UDF are available.
*   **When not to use:** For real-time rendering where the overhead of UDF evaluation and joint training is prohibitive, or when ground truth depth is unavailable.

## Background
The VRP-UDF method originates from the paper "VRP-UDF: Toward Unbiased Learning of Unsigned Distance Functions From Multi-View Images With Volume Rendering Priors" [S1]. It builds upon previous works on volume rendering priors [1, 24] and NeuralUDF [2]. The method addresses the limitations of standard SDF-based approaches by using UDFs, which are more robust to noise and topology changes, while mitigating the ambiguity of normals at the zero-level set. It is particularly effective for reconstructing thin surfaces and complex indoor scenes.

## Sources
[S1] VRP-UDF: Toward Unbiased Learning of Unsigned Distance Functions From Multi-View Images With Volume Rendering Priors., https://doi.org/10.1109/tpami.2026.3653901, fetched 2026-10-27
