# VRP-UDF Training & Rendering Setup

Also called: UDF-Gaussian Joint Training, Volume Rendering Prior Network

## Overview
This method integrates a neural network-based volume rendering prior (VRP) that learns Unsigned Distance Functions (UDFs) to guide the optimization of 3D Gaussian Splatting. It solves the problem of inaccurate surface reconstruction and depth estimation in Gaussian splatting by providing dense, geometrically consistent queries around the UDF zero-level set. Unlike sibling methods like [MVSplat](9213-mvsplat.md) which rely on explicit multi-view stereo consistency or pure geometric priors, this approach uses a differentiable UDF field to regularize the Gaussian primitives, ensuring unbiased learning from ground truth UDFs and depths while mitigating the ambiguities inherent in standard SDF-based normal estimation.

## How it works
The method employs a joint training pipeline where a set of Multi-Layer Perceptrons (MLPs) model the UDF and color fields, which are then used to refine Gaussian Splatting parameters.

### Inputs and Outputs
*   **Inputs:** Multi-view images, camera poses, and ground truth unsigned distance values (or depth maps) for the scene.
*   **Outputs:** Optimized 3D Gaussian primitives (position, scale, rotation, opacity, spherical harmonics coefficients) and the learned UDF/Color fields.

### Data Structures and Network Layout
The volume rendering prior consists of four distinct MLPs, all utilizing 256 hidden units. The architecture is defined as follows:

1.  **Feature Network ($f_{w1}$):**
    *   Structure: 3 layers.
    *   Input: 3D coordinates.
    *   Output: Feature vectors.
2.  **Weight Network ($f_{w2}$):**
    *   Structure: 6 layers with skip connections.
    *   Input: Feature vectors from $f_{w1}$ and 3D coordinates.
    *   Output: Weights for the UDF and color prediction.
3.  **UDF Network ($f_u$):**
    *   Structure: 8 layers with skip connections.
    *   Input: Feature vectors and 3D coordinates.
    *   Output: Unsigned distance value.
    *   **Activation:** A Softplus activation function is applied to the final output layer to enforce non-negative outputs ($d \ge 0$).
4.  **Color Network ($f_c$):**
    *   Structure: 2 layers.
    *   Input: Feature vectors and 3D coordinates. **Note:** Normal vectors are explicitly excluded from the input.
    *   Output: Color values.

### Training Schedule and Parameters
The training process is divided into specific phases with a total duration of 30,000 iterations.

1.  **Initial Phase (Iterations 0–9,000):** Standard Gaussian Splatting optimization is performed without the UDF prior.
2.  **Prior Introduction (Iteration 9,000):** Training of the UDF networks ($f_u$) and color networks ($f_c$) begins. The learning rate and decay scheduler for these networks follow established conventions from prior works [1, 24] (exact numerical values are not specified in the source).
3.  **Refinement Phase (Iterations 20,000–30,000):** During the final 10,000 iterations, the Gaussians are refined using the learned prior.
    *   **Ray Casting:** Rays are cast with a batch size of 2,048.
    *   **Sampling:** 64 points are sampled along each ray.
    *   **Rendering:** The UDF and color networks are queried at these sample points to compute volume rendering losses, which are backpropagated to update both the Gaussian parameters and the network weights.

### Pseudocode
The following pseudocode outlines the refinement step during the final training phase. It assumes standard differentiable rendering primitives are available.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_vrp_udf_gaussian(gaussians, cameras, images, total_iterations=30000, 
                           udf_start_iter=9000, refinement_start_iter=20000):
    """
    Joint training loop for VRP-UDF and 3D Gaussian Splatting.
    """
    # Initialize networks
    fw1 = MLP(layers=3, hidden=256)
    fw2 = MLP(layers=6, hidden=256, skip_connections=True)
    fu = MLP(layers=8, hidden=256, skip_connections=True)
    fc = MLP(layers=2, hidden=256)
    
    # Optimizers
    opt_gaussians = Optimizer(gaussians.parameters())
    opt_networks = Optimizer([fw1, fw2, fu, fc].parameters())
    
    for iteration in range(1, total_iterations + 1):
        # Sample a batch of camera views
        batch_cameras, batch_images = sample_batch(cameras, images)
        
        # Forward pass: Render current Gaussians
        rendered_images, rendered_depths = render_gaussians(gaussians, batch_cameras)
        
        # Compute standard Gaussian Splatting losses (color, depth, etc.)
        loss_gs = compute_gaussian_losses(rendered_images, batch_images, rendered_depths)
        
        if iteration >= udf_start_iter:
            # Query UDF and Color networks for sample points
            # In practice, sample points are derived from ray casting through Gaussians
            # or random hierarchical sampling near the zero-level set.
            sample_points = get_sample_points(gaussians, batch_cameras, num_samples=64)
            
            # Get features
            features = fw1(sample_points)
            weights = fw2(features, sample_points)
            
            # Predict UDF (apply Softplus to ensure non-negativity)
            udf_values = fu(features, sample_points)
            udf_values = softplus(udf_values)
            
            # Predict Color (no normals in input)
            colors = fc(features, sample_points)
            
            # Compute UDF/Color losses against ground truth UDFs/Depths
            # Note: Ground truth UDFs are assumed to be available or derived from depth
            loss_udf = compute_udf_loss(udf_values, ground_truth_udfs)
            loss_color = compute_color_loss(colors, ground_truth_colors)
            
            # Combine losses
            if iteration >= refinement_start_iter:
                # Refinement phase: Higher weight on UDF consistency to guide Gaussians
                total_loss = loss_gs + lambda_udf * loss_udf + lambda_color * loss_color
            else:
                # Initial prior training phase
                total_loss = loss_gs + lambda_udf * loss_udf + lambda_color * loss_color
                
            # Backpropagation
            opt_networks.zero_grad()
            opt_gaussians.zero_grad()
            
            total_loss.backward()
            
            opt_networks.step()
            opt_gaussians.step()
        else:
            # Standard Gaussian Splatting update
            opt_gaussians.zero_grad()
            loss_gs.backward()
            opt_gaussians.step()

def softplus(x):
    """Enforce non-negative output."""
    return torch.log(1 + torch.exp(x))
```

### Edge Cases and Pitfalls
*   **Normal Ambiguity:** Normal vectors must not be included in the input to the color network ($f_c$). At the zero-level set of a UDF, the gradient is non-differentiable and normals are ambiguous, which can degrade color prediction [S1].
*   **Sampling Inaccuracies:** Random hierarchical sampling and the ill-defined nature of UDF fields near surfaces can cause rendering inaccuracies. This is a common issue in volume rendering frameworks, leading to depth discrepancies even with theoretically unbiased mappings [S1].
*   **Convergence Parameters:** Excessively large convergence parameters (such as the "flip factor" in NeuralUDF) can cause the UDF to flip into an SDF prematurely, before the true surface is reached, resulting in depth rendering errors [S1].

## Operating characteristics
*   **Time Complexity:** The method adds significant computational overhead during the refinement phase due to the forward passes through four MLPs ($f_{w1}, f_{w2}, f_u, f_c$) for every sample point. The training duration is fixed at 30,000 iterations, with the prior active for the latter 21,000 iterations.
*   **Memory Complexity:** Memory usage increases due to the storage of the four MLPs (totaling 19 layers with 256 hidden units each) and the gradients for both the Gaussians and the networks.
*   **Performance:** The method achieves high accuracy in surface reconstruction and depth estimation, outperforming baselines like NeUDF and VRPrior on complex indoor scenes and thin structures (e.g., eggshells, plant branches) [S1]. It provides the best accuracy among tested renderers for both UDFs and SDFs.
*   **Limitations:** Despite being unbiased in theory, practical inaccuracies persist due to sampling noise and UDF ill-definition near surfaces. It may struggle to capture extremely thin structures completely without advanced extraction techniques [S1].

## Background
The VRP-UDF method originates from the paper "VRP-UDF: Toward Unbiased Learning of Unsigned Distance Functions From Multi-View Images With Volume Rendering Priors" [S1]. It builds upon the principles of differentiable volume rendering and unsigned distance function learning, extending them to the context of 3D Gaussian Splatting. The approach addresses limitations in previous UDF-based methods (like NeuralUDF) where premature flipping and normal ambiguity hindered high-fidelity reconstruction. It is particularly relevant for applications requiring precise geometric details from multi-view images, such as digital heritage (e.g., potted flowers, paper boats) and complex indoor scenes.

## Sources
[S1] VRP-UDF: Toward Unbiased Learning of Unsigned Distance Functions From Multi-View Images With Volume Rendering Priors., https://doi.org/10.1109/tpami.2026.3653901, fetched 2026-10-27
