# Stochastic Gradient Estimation for Differentiable Rasterization

Also called: Per-Pixel Formulation, Per-Pixel Stochastic Gradient Estimation, SGE, Stochastic finite difference, transformed rasterizer

## Overview
This method transforms a non-differentiable rasterizer into a differentiable one by estimating gradients on a per-pixel basis rather than on the whole image, bounding the dimensionality of the optimization problem [S1]. It computes stochastic gradients for differentiable rasterization by summing per-pixel gradient estimates instead of a full-image error, reducing noise in high-dimensional parameter spaces [S1]. Unlike standard Stochastic Gradient Estimation which estimates gradients on the whole image (scaling poorly with dimensionality), this method estimates gradients on a per-pixel basis, bounding the dimensionality to the number of parameters contributing to a single pixel [S1]. Unlike backpropagation which requires a differentiable framework (Pytorch/Tensorflow), this method requires no external dependencies and works with existing non-differentiable rasterizers [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Scene parameters $\boldsymbol{\theta}$ (e.g., vertex positions, texture values, Gaussian attributes), random sign vector $\boldsymbol{s}$, perturbation magnitude vector $\boldsymbol{\epsilon}$, number of samples $N$.
*   **Outputs:** Gradient estimates $\frac{\partial f}{\partial \theta_i}$ for each parameter $\theta_i$.

**Data Structures:**
The rasterizer must output specific buffers to identify contributing parameters:
1.  **Opaque Geometry:** RGB buffer and an ID buffer containing the index of the rasterized triangle for each pixel [S1].
2.  **Transparent Geometry:** RGB buffer and a deep ID buffer containing multiple triangle IDs per pixel, representing an ordered list of triangles seen by the pixel [S1].
3.  **Textures:** RGB buffer, ID buffer, and a UV buffer containing the UV coordinates used to fetch the texture in each pixel [S1].
4.  **Volumes:** RGB buffer and voxel access structures for ray-marching [S1].

**Numbered Steps:**
1.  **Initialize Accumulators:** For each parameter $\theta_i$, initialize gradient accumulator $G_i = 0$ and sample counter $n=0$.
2.  **Loop for Averaging:** While $n < N$:
    3.  **Generate Random Sign Vector:** Generate a random sign vector $\boldsymbol{s}$ where each component $s_i \in \{-1, +1\}$ with probability $1/2$ [S1].
    4.  **Perturb Parameters:**
        *   Create $\boldsymbol{\theta}^+ = \boldsymbol{\theta} + \boldsymbol{s} \odot \boldsymbol{\epsilon}$ [S1].
        *   Create $\boldsymbol{\theta}^- = \boldsymbol{\theta} - \boldsymbol{s} \odot \boldsymbol{\epsilon}$ [S1].
    5.  **Rasterize Forward Pass (+):** Rasterize the scene with parameters $\boldsymbol{\theta}^+$ to produce image $I^+$ and auxiliary buffers (ID/UV) [S1].
    6.  **Rasterize Forward Pass (-):** Rasterize the scene with parameters $\boldsymbol{\theta}^-$ to produce image $I^-$ and auxiliary buffers (ID/UV) [S1].
    7.  **Compute Per-Pixel Gradients:** For each pixel $(w,h)$ impacted by parameter $\theta_i$:
        *   Retrieve the error $f_{w,h}(\boldsymbol{\theta})$ for the current unperturbed parameters (or compute it from $I^+$ and $I^-$ relative to a target, though the formula uses the difference of errors directly) [S1]. *Note: The source defines $f_{w,h}$ as the $l_2$ pixel error* [S1].
        *   Compute the per-pixel gradient contribution for $\theta_i$:
            $$ \Delta G_i^{(w,h)} = \frac{f_{w,h}(\boldsymbol{\theta}^+) - f_{w,h}(\boldsymbol{\theta}^-)}{2 s_i \epsilon_i} $$
            [S1]
        *   Accumulate: $G_i \leftarrow G_i + \Delta G_i^{(w,h)}$ [S1].
    8.  **Increment Sample Counter:** $n \leftarrow n + 1$ [S1].
9.  **Average Gradients:** For each parameter $\theta_i$, compute the final gradient estimate:
    $$ \frac{\partial f}{\partial \theta_i} = \frac{1}{N} \sum_{n=1}^{N} \sum_{(w,h) \text{ impacted by } \theta_i} \frac{f_{w,h}(\boldsymbol{\theta}^+) - f_{w,h}(\boldsymbol{\theta}^-)}{2 s_i \epsilon_i} $$
    [S1]
10. **Update Parameters:** Update $\boldsymbol{\theta}$ using an optimizer (e.g., Adam) with the computed gradients [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def stochastic_gradient_estimation(scene_params, epsilon, N, target_image):
    """
    Estimates gradients for scene parameters using per-pixel stochastic gradient estimation.
    
    Args:
        scene_params: Dictionary or array of scene parameters theta.
        epsilon: Perturbation magnitude vector epsilon.
        N: Number of stochastic gradient estimates to average.
        target_image: Target RGB image for error calculation.
        
    Returns:
        gradients: Estimated gradients dL/d_theta.
    """
    num_params = len(scene_params)
    gradients = np.zeros(num_params)
    
    for n in range(N):
        # Step 3: Generate random sign vector
        s = np.random.choice([-1, 1], size=num_params)
        
        # Step 4: Perturb parameters
        theta_plus = scene_params + s * epsilon
        theta_minus = scene_params - s * epsilon
        
        # Step 5 & 6: Rasterize
        # rasterize returns (image, id_buffer, uv_buffer)
        img_plus, id_buf_plus, uv_buf_plus = rasterize(theta_plus)
        img_minus, id_buf_minus, uv_buf_minus = rasterize(theta_minus)
        
        # Step 7: Compute per-pixel gradients
        # Calculate L2 error for each pixel
        error_plus = (img_plus - target_image) ** 2
        error_minus = (img_minus - target_image) ** 2
        
        # Iterate over pixels and parameters
        # Optimization: Only iterate over pixels impacted by specific parameters
        # For simplicity, we assume we can map pixels to parameters via ID/UV buffers
        
        for w in range(width):
            for h in range(height):
                # Identify parameters impacted by this pixel
                # This depends on the geometry type (opaque, transparent, texture)
                impacted_params = get_impacted_params(w, h, id_buf_plus, uv_buf_plus, scene_params)
                
                for idx in impacted_params:
                    theta_i = scene_params[idx]
                    s_i = s[idx]
                    eps_i = epsilon[idx]
                    
                    # Avoid division by zero if epsilon is 0
                    if eps_i == 0:
                        continue
                        
                    # Formula: (f(theta+) - f(theta-)) / (2 * s_i * eps_i)
                    # Note: f is the pixel error. The source uses f(w,h) as the error value.
                    # The gradient is summed over impacted pixels.
                    
                    # We need the error value at this specific pixel for both perturbations
                    err_p = error_plus[h, w, idx] # Assuming error is computed per param? 
                    # Correction: The error f(w,h) is the total pixel error or per-param error?
                    # Source Eq 8: f(w,h)(theta) is the pixel error. 
                    # The gradient d f / d theta_i is estimated by the difference in pixel error?
                    # No, Eq 8 says: sum over (w,h) impacted by theta_i of [ f(w,h)(theta+) - f(w,h)(theta-) ] / ...
                    # This implies f(w,h) is the scalar error at pixel (w,h).
                    # But the gradient is for theta_i.
                    # If theta_i affects multiple pixels, we sum the contributions.
                    
                    # Let's re-read Eq 8 carefully:
                    # d f / d theta_i = sum_{(w,h) impacted by theta_i} [ f(w,h)(theta+) - f(w,h)(theta-) ] / (2 s_i eps_i)
                    # This implies that the change in the TOTAL loss f(theta) due to theta_i is approximated by summing the changes in pixel errors f(w,h) for all pixels where theta_i is visible.
                    
                    # So, we calculate the pixel error difference for this pixel
                    pixel_err_diff = error_plus[h, w] - error_minus[h, w]
                    
                    gradients[idx] += pixel_err_diff / (2 * s_i * eps_i)

    # Step 9: Average
    gradients /= N
    
    return gradients
```
*Note: The pseudocode above simplifies the "impacted parameters" lookup. In practice, for opaque geometry, the ID buffer tells you which triangle (and thus which vertices) is visible. For textures, the UV buffer tells you which texel is visible. The loop structure should efficiently map pixels to their contributing parameters to avoid $O(W \times H \times D)$ complexity.*

**Parameters:**
*   $N$ (averaging samples): Number of stochastic gradient estimates averaged to reduce variance. Typical values in experiments are 8 or 32 [S1]. Another source mentions $N=128$ for performance breakdown [S1].
*   $\boldsymbol{\epsilon}$ (perturbation magnitude): The learning rate is set equal to the perturbation amplitude $\epsilon_i$ in all experiments [S1].
*   Adam Optimizer: Default parameters $\beta_1=0.9, \beta_2=0.999$ [S1].

**Edge Cases and Pitfalls:**
*   **Variance:** The estimator is stochastic and exhibits variance. The variance increases with the number of dimensions $d$. This is traded for performance compared to finite differences [S1].
*   **High Dimensionality:** Full-image error formulation causes high noise in high-dimensional spaces, making optimization impractical for large scenes (e.g., 100K triangles). The per-pixel approach bounds this [S1].
*   **Texture Resolution:** Texture gradient estimation approximates the elliptical footprint by only updating the closest texel to the center. This is sufficient if resolution is high enough to avoid sub-pixel scale texels [S1].
*   **Transparent Geometry:** Requires a deep ID buffer to handle depth complexity. The order of triangles must be preserved for correct alpha compositing gradient estimation [S1].

## Subtopics

### Gradient Depth Limiting (Limited-BW)
Limits gradient computation to the $N$ nearest Gaussians per pixel. Parameter: $N=10$ [S2, unofficial]. Pitfall: Restricting gradient depth to a small constant (e.g., 10) severely degrades reconstruction quality, proving that full depth complexity is necessary for high-fidelity rendering [S2, unofficial]. Number: On Average-30K, Full achieves PSNR 26.05. Limited-BW ($N=10$) drops to 19.19 [S2, unofficial]. Comparison: Limiting gradients to $N=10$ causes blurry boundaries and loss of detail (e.g., table edges and tree parts become indistinct). Unlimited depth complexity preserves sharp boundaries [S2, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   Time complexity per step is proportional to $N \times (\text{Rasterization Time} + \text{Gradient Computation Time})$.
*   Memory complexity is dominated by storing the auxiliary buffers (ID, UV) and the intermediate images for perturbation passes [S1].

**Measured Performance:**
*   On NVIDIA 4090 GPU for 1K triangles: P=12μs, R=28μs, G=13μs, D=21μs, Sum=6.8ms (with $N=128$) [S1].
*   On NVIDIA 4090 GPU for 10K triangles: P=14μs, R=38μs, G=14μs, D=21μs, Sum=8.4ms (with $N=128$) [S1].
*   On NVIDIA 4090 GPU for 100K triangles: P=54μs, R=126μs, G=35μs, D=105μs, Sum=27ms (with $N=128$) [S1].
*   Performance on Figure 4 experiment (1024x1024, 1748 vertices, 1M+ params): nvDiffRast takes 33ms/step with 8 samples; Ours takes 21ms/step with 8 samples [S1].
*   Performance on Figure 5 experiment (1024x1024, 1748 vertices, 791K params): nvDiffRast takes 33ms/step with 8 samples; Ours takes 18ms/step with 32 samples [S1].
*   Optimization of a control mesh of 2K triangles (50K triangles after subdivision) with 1024^2 PBR textures took 130 seconds on an Intel Arc 770 GPU [S1].

**Scaling:**
*   The method scales well up to 1M+ parameters because the dimensionality of the optimization problem per pixel is bounded [S1].
*   Compared to nvDiffRast, this method is slower per step due to noisy gradients requiring more iterations, but faster per step execution time within the engine. Overall qualitative results are similar with same optimization time [S1].

**Strengths and Limitations:**
*   **Strengths:** Simple to implement, no external dependencies (Pytorch/Tensorflow), works with existing non-differentiable rasterizers, scalable to high-dimensional scenes [S1].
*   **Limitations:** Noisy gradients require more iterations for convergence, scope is limited to raster graphics with direct visibility only (no shadows or multiple-bounce illumination) [S1].

**When to Use:**
*   Targeted at game developers and users with existing non-differentiable rasterization engines who cannot use existing differentiable rasterizers due to workflow or platform constraints [S1].
*   Used to optimize 3D assets including meshes, displacement mapping, Catmull-Clark subdivision surfaces, semi-transparent geometry, physically based materials, 3D volumetric data, and 3D Gaussian Splats [S1].

## Background
**Origin:**
The method was proposed by Thomas Deliot, Eric Heitz, and Laurent Belcour from Intel Corporation, France [S1]. It was published in the Proceedings of the ACM Computer Graphics and Interactive Techniques (SIGGRAPH Asia 2024) [S1]. The paper title is "Transforming a Non-Differentiable Rasterizer into a Differentiable One with Stochastic Gradient Estimation" [S1].

**History and Development:**
The method builds on previous work in differentiable rendering using stochastic gradient estimation, such as Fischer and Ritschel [2023] and Le Lidec et al. [2021], which did not scale to high-dimensional problems [S1]. The key insight was that estimating gradients on a per-pixel basis bounds the dimensionality of the optimization problem, making it scalable [S1].

**Usage:**
It is used to optimize 3D assets in their engine-specific geometry and material representations, allowing in-engine optimization without switching to external differentiable frameworks [S1].

## Sources
[S1] Transforming a Non-Differentiable Rasterizer into a Differentiable One with Stochastic Gradient Estimation, https://doi.org/10.1145/3651298, fetched 2026-10-26
[S2] GaussianSplatting論文のAblationについての解説・考察, https://qiita.com/kotauchisunsun/items/f1e89f28aa770b8d3b19, fetched 2026-10-29 (unofficial)
