# DropGaussian


## Overview
DropGaussian is a structural regularization technique designed to mitigate overfitting in 3D Gaussian Splatting (3DGS) when trained on a sparse set of input views [S1]. It operates by randomly excluding a subset of Gaussians from the rendering and gradient computation during training, while simultaneously scaling the opacity of the remaining Gaussians to preserve the total color contribution [S1]. Unlike prior-based methods that rely on external depth signals or 2D generative models, DropGaussian is prior-free and introduces no additional computational complexity or external data requirements [S1].

## How it works
DropGaussian modifies the standard 3DGS training loop by introducing a stochastic dropout mechanism applied to the Gaussian primitives [S1]. The method addresses the issue where Gaussians far from the camera or occluded in sparse views receive insufficient gradient feedback, leading to overfitting [S1]. By randomly removing closer Gaussians, the visibility of distant Gaussians is improved, allowing them to receive larger gradients and contribute more effectively to the optimization [S1].

### Inputs and Outputs
*   **Inputs:**
    *   Set of 3D Gaussians $\mathcal{G} = \{g_i\}$, where each $g_i$ contains position $\mathbf{p}_i$, scale $\mathbf{s}_i$, rotation $\mathbf{q}_i$, opacity $o_i$, and spherical harmonic coefficients $\mathbf{c}_i$ [S1].
    *   Current training iteration index $t$ [S1].
    *   Total number of training iterations $t_{\text{total}}$ [S1].
    *   Base dropping rate parameter $r$ (typically 0.1) [S1].
    *   Scaling factor $\gamma$ (typically 1) [S1].
    *   Target training view image $I_{\text{target}}$ [S1].
*   **Outputs:**
    *   Rendered image $I_{\text{rendered}}$ using the modified Gaussian set [S1].
    *   Gradients backpropagated only to the non-dropped Gaussians [S1].

### Data Structures
The method operates on the standard 3DGS primitive representation defined in the parent article [Pruning and Density Control](16215-pruning-and-density-control.md) [S1]. No new data structures are introduced; the method modifies the opacity values $o_i$ temporarily during the forward pass [S1].

### Parameters
*   $r$: Base dropping rate. Represents the fraction of Gaussians to remove. Typical value: $0.1$ (10%) [S1].
*   $\gamma$: Scaling factor for the progressive dropping rate. Values: $0$ or $1$ [S1].
*   $t$: Current training iteration index [S1].
*   $t_{\text{total}}$: Total number of training iterations [S1].

### Formulas
The dropping rate $r_t$ increases progressively with the training iteration $t$ [S1]:

$$
r_t = \gamma \cdot \frac{t}{t_{\text{total}}}
$$

For each Gaussian $i$ in the current iteration:
1.  A binary mask $M(i)$ is determined based on whether the Gaussian is dropped [S1].
2.  The modified opacity $\tilde{o}_i$ is calculated as:

$$
\tilde{o}_i = M(i) \cdot o_i
$$

where:
$$
M(i) = 
\begin{cases} 
\frac{1}{1 - r_t} & \text{if Gaussian } i \text{ is NOT dropped} \\
0 & \text{if Gaussian } i \text{ IS dropped}
\end{cases}
$$

This compensation factor ensures that the total opacity contribution of the remaining Gaussians to the pixel color is maintained, compensating for the removed Gaussians [S1].

### Steps
1.  **Calculate Dropping Rate:** Compute the current dropping rate $r_t$ using the formula $r_t = \gamma \cdot \frac{t}{t_{\text{total}}}$ [S1].
2.  **Select Gaussians to Drop:** Randomly select a subset of Gaussians to be dropped. The selection probability for each Gaussian corresponds to $r_t$ [S1].
3.  **Compute Modified Opacity:** For each Gaussian $i$:
    *   If $i$ is selected to be dropped, set its effective opacity contribution to 0 (effectively removing it from rendering) [S1].
    *   If $i$ is not dropped, scale its opacity $o_i$ by the factor $\frac{1}{1 - r_t}$ to get $\tilde{o}_i$ [S1].
4.  **Render:** Perform differentiable rasterization using the modified opacities $\tilde{o}_i$ to render the image $I_{\text{rendered}}$ from the current camera viewpoint [S1].
5.  **Compute Loss:** Calculate the loss $\mathcal{L}$ (typically L1 + SSIM) between $I_{\text{rendered}}$ and the target training view $I_{\text{target}}$ [S1].
6.  **Backpropagate:** Compute gradients of $\mathcal{L}$ with respect to the Gaussian parameters. Gradients are only backpropagated to the non-dropped Gaussians (those with $M(i) \neq 0$) [S1].
7.  **Update:** Update the parameters of the non-dropped Gaussians using the optimizer (e.g., Adam) [S1].
8.  **Evaluation:** During the test/evaluation phase, render using all Gaussians without dropping any (i.e., $r_t = 0$ effectively, or no masking) to generate high-quality novel view images [S1].

### Edge Cases and Pitfalls
*   **Timing of Application:** Overfitting in sparse-view settings primarily manifests during the later stages of training rather than the initial phase [S1]. Applying the drop ratio too early may negatively impact initial convergence [S1]. The progressive schedule $r_t$ addresses this by starting with a low or zero drop rate and increasing it over time [S1].
*   **Distant Gaussians:** In sparse-view conditions, Gaussians far from the camera often have low transmittance and are occluded, leading to low gradient feedback [S1]. DropGaussian mitigates this by improving their visibility through the random removal of closer Gaussians [S1].
*   **Opacity Compensation:** Failure to apply the compensation factor $\frac{1}{1-r_t}$ would result in a decrease in the cumulative opacity contributing to the color value of each pixel, leading to darker renders and incorrect optimization [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def copy_gaussian(g):
    """Shallow copy of a Gaussian primitive."""
    new_g = type(g)()
    new_g.position = g.position
    new_g.scale = g.scale
    new_g.rotation = g.rotation
    new_g.opacity = g.opacity
    new_g.sh_coeffs = g.sh_coeffs
    return new_g

def drop_gaussian_forward_pass(gaussians, t, t_total, r_base, gamma):
    """
    Performs one forward pass of DropGaussian.
    
    Args:
        gaussians: List of Gaussian objects, each with attributes:
                   - opacity: float (original opacity o_i)
                   - position, scale, rotation, sh_coeffs
        t: int, current iteration index
        t_total: int, total number of training iterations
        r_base: float, base dropping rate (e.g., 0.1)
        gamma: float, scaling factor (0 or 1)
        
    Returns:
        modified_gaussians: List of Gaussian objects with modified opacity
    """
    # Step 1: Calculate current dropping rate
    r_t = gamma * (t / t_total)
    
    # Step 2: Randomly select Gaussians to drop
    num_gaussians = len(gaussians)
    num_to_drop = int(num_gaussians * r_t)
    indices = list(range(num_gaussians))
    import random
    random.shuffle(indices)
    dropped_indices = set(indices[:num_to_drop])
    
    # Step 3: Compute modified opacity
    compensation_factor = 1.0 / (1.0 - r_t) if r_t < 1.0 else float('inf')
    modified_gaussians = []
    
    for i, g in enumerate(gaussians):
        new_g = copy_gaussian(g)
        if i in dropped_indices:
            new_g.opacity = 0.0
            new_g.is_dropped = True
        else:
            new_g.opacity = g.opacity * compensation_factor
            new_g.is_dropped = False
        modified_gaussians.append(new_g)
        
    return modified_gaussians

def drop_gaussian_backward_pass(modified_gaussians, loss, optimizer):
    """
    Performs backward pass and optimization step.
    
    Args:
        modified_gaussians: List of Gaussian objects from forward pass
        loss: Scalar loss value (L1 + SSIM)
        optimizer: Optimizer object (e.g., Adam)
    """
    # Step 5 & 6: Compute gradients and backpropagate
    # Gradients are only computed for non-dropped Gaussians
    # This is handled by the differentiable rasterizer which ignores dropped Gaussians (opacity=0)
    
    # Step 7: Update parameters
    optimizer.step()
    optimizer.zero_grad()
```

## Subtopics

### Optimization Schedule and Densification
Manages the growth curve of primitives, synchronizing densification with rendering resolution [Parent]. DropGaussian complements this by controlling overfitting during the densified phase [G]. [Link](15107-optimization-schedule-and-densification.md)

### Reconstruction-aware Pruning Scheduler
Uses reconstruction error to guide pruning [Parent]. DropGaussian differs by using random structural regularization rather than error-based pruning [G]. [Link](16199-reconstruction-aware-pruning-scheduler.md)

## Operating characteristics
*   **Time Complexity**: The method adds negligible computational overhead. It involves a simple random selection and opacity scaling, which are $O(N)$ operations where $N$ is the number of Gaussians. The core rasterization and gradient computation remain unchanged in complexity [S1].
*   **Memory Complexity**: No additional memory is required beyond the standard 3DGS representation. The opacity values are modified in-place or copied temporarily during the forward pass [S1].
*   **Performance**: DropGaussian achieves competitive performance in sparse-view settings (e.g., 3-view, 6-view, 9-view) on benchmarks like LLFF, Mip-NeRF360, and Blender, often surpassing prior-based methods in PSNR without increasing computational complexity [S1].
*   **Strengths**: Prior-free, no external data required, simple implementation, effective against overfitting in sparse views [S1].
*   **Limitations**: Requires careful tuning of the progressive schedule (though the linear schedule is effective). May not be as effective in dense-view settings where overfitting is less of an issue [G].
*   **When to use**: Sparse-view novel view synthesis where overfitting to training views is a concern [S1].
*   **When not to use**: Dense-view settings where standard 3DGS performs well, or when external priors are available and preferred [G].

## Background
DropGaussian was introduced in the paper "DropGaussian: Structural Regularization for Sparse-view Gaussian Splatting" [S1]. The method addresses the limitation of standard 3DGS in sparse-view settings, where it tends to overfit to the limited number of training views [S1]. Prior methods relied on strong priors such as 2D generative contextual information or external depth signals, which can introduce error propagation and high computational burdens [S1]. DropGaussian provides a simple, prior-free alternative by mimicking the dropout technique from neural network training, applied structurally to the Gaussian primitives [S1]. The code and models are publicly available [S1].

## Sources
[S1] DropGaussian: Structural Regularization for Sparse-view Gaussian Splatting, https://doi.org/10.1109/cvpr52734.2025.02012, fetched 2026-10-26
