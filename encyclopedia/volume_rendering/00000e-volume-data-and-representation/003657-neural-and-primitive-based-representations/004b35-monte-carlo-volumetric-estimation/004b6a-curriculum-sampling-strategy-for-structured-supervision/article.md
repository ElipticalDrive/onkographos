# Curriculum Sampling Strategy for Structured Supervision
Also called: Planar sampling, Slice-based supervision

## Overview
This method provides structured spatial supervision for neural volumetric representations by blending sparse voxel-based Monte Carlo estimation with planar slice-based constraints. It solves the issue of isolated voxel samples lacking spatial coherence by enforcing consistency along 2D planes, which directly improves the fidelity of slices used in shear-warp rendering. It sets itself apart from pure voxel supervision by introducing geometric structure through random plane orientations, and from dense image supervision by maintaining sparsity and memory efficiency through selective pixel sampling on those planes.

## How it works
The method optimizes a set of Gaussian kernels $\Theta = \{\mu_i, \Sigma_i, C_i, \alpha_i\}_{i=1}^m$ representing a volumetric field $f_\Theta(x)$ [S1]. It combines two loss terms: the Monte Carlo volumetric loss $\hat L_{MC}$ (from the parent method) and a slice-based loss $L_{slice}$.

**Inputs:**
- Training iteration $t$.
- Gaussian parameters $\Theta$.
- Ground-truth voxel colors $y(x)$.
- Hyperparameters: $\lambda_{s,\max}$, $\gamma_s$, $\beta_s$.

**Outputs:**
- Gradients $\nabla_\Theta \mathcal{L}(t)$ for updating $\Theta$.

**Data Structures:**
- **Gaussian Set $G$**: A list of $m$ Gaussians. Each Gaussian $G_i$ contains:
  - $\mu_i \in \mathbb{R}^3$: Mean (spatial location).
  - $\Sigma_i \in \mathbb{R}^{3 \times 3}$: Covariance matrix (spread and orientation).
  - $C_i \in \mathbb{R}^3$: RGB color.
  - $\alpha_i \in \mathbb{R}$: Opacity.
- **Sample Set $S$**: A set of 2D pixel coordinates on a sampled plane. Each pixel maps to a 3D coordinate $x$ in the volume.

**Numbered Steps:**
1. **Sample Plane**: In each iteration $t$, sample a plane with an arbitrary orientation in 3D space [S1].
2. **Select Pixels**: Select a sparse set of 2D pixel locations $S$ on that plane. Each pixel corresponds to a specific 3D coordinate $x$ within the volume [S1].
3. **Compute Slice Loss**: Calculate the mean squared error between the predicted color $f_\Theta(x)$ and the ground-truth color $y(x)$ for all pixels in $S$:
   $$ L_{slice} = \frac{1}{|S|} \sum_{x \in S} \| f_\Theta(x) - y(x) \|_2^2 $$
   where $|S|$ is the number of sampled pixels [S1].
4. **Calculate Weights**: Compute time-varying weights $\lambda_s(t)$ and $\lambda_v(t)$ based on the current iteration $t$ [S1]:
   $$ \lambda_s(t) = \lambda_{s,\max} \sigma \left( \frac{t - \gamma_s}{\beta_s} \right) $$
   $$ \lambda_v(t) = 1 - \lambda_s(t) $$
   where $\sigma(\cdot)$ is the sigmoid function [S1].
5. **Combine Losses**: Compute the total loss $\mathcal{L}(t)$ by blending the Monte Carlo loss $\hat L_{MC}$ and the slice loss $L_{slice}$:
   $$ \mathcal{L}(t) = \lambda_v(t) \hat L_{MC} + \lambda_s(t) L_{slice} $$
   [S1].
6. **Update Parameters**: Perform gradient descent on $\Theta$ using $\nabla_\Theta \mathcal{L}(t)$ [S1].

**Parameters:**
- $\lambda_{s,\max}$: Maximum weight for the slice loss. Not given in the sources.
- $\gamma_s$: Iteration at which slice supervision becomes significant. Not given in the sources.
- $\beta_s$: Controls the smoothness of the transition. Not given in the sources.

**Edge Cases and Pitfalls:**
- **Abrupt Switching**: Switching supervision abruptly between voxel and slice modes can be detrimental to training stability. This is handled by using the sigmoid schedule to gradually introduce slice supervision [S1].
- **Precision**: The method relies on the accuracy of mapping 2D pixel coordinates to 3D voxel locations. If the plane orientation or pixel sampling is imprecise, the structured constraint may not align with the ground truth geometry [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sigmoid(x):
    """Standard sigmoid function."""
    return 1.0 / (1.0 + math.exp(-x))

def compute_slice_loss(Gaussians, plane, pixels_S, ground_truth_voxels):
    """
    Computes L_slice for a set of pixels on a plane.
    Gaussians: List of Gaussian objects with evaluate(x) method.
    plane: Object defining the 3D plane geometry.
    pixels_S: List of 2D pixel indices on the plane.
    ground_truth_voxels: Lookup function or array for y(x).
    """
    total_error = 0.0
    count = 0
    for pixel in pixels_S:
        # Map 2D pixel to 3D coordinate x on the plane
        x = plane.pixel_to_3d(pixel)
        
        # Evaluate predicted color f_theta(x)
        predicted_color = Gaussians.evaluate(x)
        
        # Get ground truth color y(x)
        true_color = ground_truth_voxels.get_color(x)
        
        # Compute squared L2 distance
        diff = predicted_color - true_color
        total_error += diff[0]**2 + diff[1]**2 + diff[2]**2
        count += 1
    
    if count == 0:
        return 0.0
    return total_error / count

def compute_weights(t, gamma_s, beta_s, lambda_s_max):
    """
    Computes lambda_s(t) and lambda_v(t).
    """
    # Sigmoid activation for slice supervision
    sigmoid_input = (t - gamma_s) / beta_s
    lambda_s = lambda_s_max * sigmoid(sigmoid_input)
    lambda_v = 1.0 - lambda_s
    return lambda_v, lambda_s

def train_step(Gaussians, t, gamma_s, beta_s, lambda_s_max, 
               monte_carlo_sampler, plane_sampler, pixel_sampler, 
               ground_truth_voxels, learning_rate):
    """
    One training iteration.
    """
    # 1. Sample plane and pixels
    plane = plane_sampler.sample_arbitrary_plane()
    pixels_S = pixel_sampler.sample_sparse_pixels(plane)
    
    # 2. Compute Slice Loss
    L_slice = compute_slice_loss(Gaussians, plane, pixels_S, ground_truth_voxels)
    
    # 3. Compute Monte Carlo Loss (from parent method)
    # Note: Monte Carlo sampler returns samples and weights for unbiased estimation
    L_MC = monte_carlo_sampler.compute_loss(Gaussians, ground_truth_voxels)
    
    # 4. Calculate Weights
    lambda_v, lambda_s = compute_weights(t, gamma_s, beta_s, lambda_s_max)
    
    # 5. Combine Losses
    L_total = lambda_v * L_MC + lambda_s * L_slice
    
    # 6. Update Parameters (Gradient Descent)
    # Gradients are computed via backpropagation through L_total
    gradients = compute_gradients(L_total, Gaussians)
    Gaussians.update(learning_rate, gradients)
    
    return L_total
```

## Subtopics

### Monte Carlo Volumetric Estimation
The parent method providing the voxel-based loss component. It uses importance sampling to estimate the volume rendering integral efficiently. Choose this when memory efficiency for large volumes is critical and spatial coherence is less important than global energy conservation. [12926-monte-carlo-volumetric-estimation.md]

## Operating characteristics
- **Time Complexity**: Depends on the number of Monte Carlo samples and the number of slice pixels $|S|$. The slice loss computation is $O(|S| \cdot m)$ if naive Gaussian evaluation is used, or optimized via spatial hashing. The Monte Carlo loss is $O(N_{MC} \cdot m)$.
- **Memory Complexity**: Memory efficient as it only stores the Gaussian parameters and does not require dense voxel grids for supervision. The plane sampling is transient.
- **Strengths**: Improves spatial coherence of the rendered slices, which is crucial for shear-warp rendering. Maintains sparsity of the input data.
- **Limitations**: Requires careful tuning of the curriculum schedule parameters ($\gamma_s, \beta_s$). Performance depends on the quality of the random plane sampling.
- **When to use**: When using shear-warp rendering and needing to improve slice fidelity without dense supervision.
- **When not to use**: When real-time training speed is paramount and slice coherence is not a priority, or when the volume has no clear planar structures.

## Background
The method was introduced in "Gaussian Volumetric Representation for Efficient Shear-Warp Visualization" [S1]. It addresses the limitations of sparse voxel supervision by incorporating planar constraints to better support the shear-warp rendering pipeline, which composites stacks of 2D slices. The curriculum learning aspect, using a sigmoid schedule to blend losses, is adapted from general curriculum learning practices [37] to ensure stable training.

## Sources
[S1] Gaussian Volumetric Representation for Efficient Shear-Warp Visualization, https://doi.org/10.1007/978-3-032-37461-5_35, fetched 2026-10-26
