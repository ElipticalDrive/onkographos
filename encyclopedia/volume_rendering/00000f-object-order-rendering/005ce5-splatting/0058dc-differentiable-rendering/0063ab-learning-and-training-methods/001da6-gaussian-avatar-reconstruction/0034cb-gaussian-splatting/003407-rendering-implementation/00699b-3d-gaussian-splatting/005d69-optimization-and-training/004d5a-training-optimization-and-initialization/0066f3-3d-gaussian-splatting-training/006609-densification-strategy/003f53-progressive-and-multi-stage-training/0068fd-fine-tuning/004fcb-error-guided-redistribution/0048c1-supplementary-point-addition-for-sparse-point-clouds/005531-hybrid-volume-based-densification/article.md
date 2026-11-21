# Hybrid Volume-based Densification

Also called: Point Densitification based on Hybrid Volume

## Overview
Hybrid Volume-based Densification is a strategy that uses a grid-based volumetric representation to supply missing Gaussian points in low-texture, distant, or sky regions where Structure-from-Motion (SfM) initialization fails, preventing empty 2D splats [S1]. It differs from sibling methods like [Distance-Aware Replication on Tangent-planes (DART)](12502-distance-aware-replication-on-tangent-planes-dart.md) by relying on a continuous neural radiance field (MLP) to predict density and position, rather than geometric replication on tangent planes, making it suitable for filling large voids where no initial point cloud exists.

## How it works
**Inputs and Outputs:**
*   **Inputs:** An MLP $f_\theta$ parameterized by $\theta$, a set of rays $\mathbf{R}$, near/far plane distances $u_n, u_f$, and a density threshold $\tau$.
*   **Outputs:** A set of new 3D Gaussian splats (position, covariance, color, opacity) added to the scene representation.

**Data Structures:**
*   **MLP $f_\theta$:** A Multi-Layer Perceptron taking a 3D location $\mathbf{x}$ and a direction vector $\mathbf{d}$ as input.
*   **Grid-based Volume:** Implicitly defined by the MLP $f_\theta$. It maps $(\mathbf{x}, \mathbf{d})$ to density $\sigma_\theta \in [0, \infty)$ and color $\mathbf{c}_\theta \in [0, 1]^3$ [S1].

**Formulas:**
1.  **MLP Prediction:**
    $$ \mathbf{c}_\theta, \sigma_\theta = f_\theta(\mathbf{x}, \mathbf{d}) $$
    Where $\mathbf{x}$ is the 3D location and $\mathbf{d}$ is the viewing direction [S1].

2.  **Predicted Depth $\hat{D}_\theta(\mathbf{r})$:**
    The expected depth along a ray $\mathbf{r}$ is approximated by integrating sampled particles weighted by transmittance:
    $$ \hat{\mathbf{D}}_\theta(\mathbf{r}) = \int_{u_n}^{u_f} Q(u) \sigma_\theta(\mathbf{r}(u)) u du $$
    Where $u_n$ and $u_f$ are the near and far clipping planes [S1].

3.  **Accumulated Transmittance $Q(u)$:**
    $$ Q(u) = \exp \left( -\int_{u_n}^u \sigma_\theta(\mathbf{r}(s)) ds \right) $$
    This represents the probability of light traveling from the camera to distance $u$ without interaction [S1].

4.  **Gaussian Position and Function:**
    The 3D position $\mathbf{x}$ for a new Gaussian is derived from the ray origin $\mathbf{o}$, direction $\mathbf{d}_\mathbf{r}$, and predicted depth:
    $$ \mathbf{x} = \mathbf{o} + \hat{\mathbf{D}}_\theta(\mathbf{r}) \mathbf{d}_\mathbf{r} $$
    The Gaussian function $G(x)$ uses this position as its mean $\mu$:
    $$ G(x) = e^{-\frac{1}{2}(\mathbf{o} + \hat{\mathbf{D}}_\theta(\mathbf{r})\mathbf{d}_\mathbf{r} - \mu)^T \mathbf{\Sigma}^{-1} (\mathbf{o} + \hat{\mathbf{D}}_\theta(\mathbf{r})\mathbf{d}_\mathbf{r} - \mu)} $$
    Where $\mathbf{\Sigma}$ is the covariance matrix [S1].

**Parameters:**
*   $\tau$: Predefined density threshold for adding new Gaussians. Value not explicitly given in the sources [S1].
*   $u_n, u_f$: Predefined near and far planes for rendering [S1].

**Steps:**
1.  **Warm-up:** Perform warm-up calculations in the grid-based volume to ensure stability of the MLP predictions [S1].
2.  **Candidate Selection:** Select candidate locations $\mathbf{x}$ within a bounded range (implicitly defined by the ray set $\mathbf{R}$ and depth bounds $u_n, u_f$) [S1].
3.  **Density Check:** For each candidate location, evaluate the density $\sigma_\theta$ predicted by $f_\theta$.
4.  **Densification:** If the density at $\mathbf{x}$ meets or exceeds the threshold $\tau$, introduce a new Gaussian splat into the pipeline. This includes initializing its position (derived from $\hat{\mathbf{D}}_\theta$), covariance matrix, color, and opacity [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hybrid_volume_densification(mlp_f_theta, rays, u_n, u_f, threshold_tau):
    """
    Adds new 3D Gaussians based on a grid-based volumetric representation.
    
    Args:
        mlp_f_theta: MLP function f_theta(x, d) returning (color, density).
        rays: List of ray objects, each with origin o, direction d_r, and sample points u.
        u_n: Near plane distance.
        u_f: Far plane distance.
        threshold_tau: Density threshold for adding new Gaussians.
        
    Returns:
        new_gaussians: List of new Gaussian splat parameters.
    """
    new_gaussians = []
    
    # Step 1: Warm up calculations (implementation detail not specified in source)
    # [G] Typically involves a few forward passes to stabilize MLP gradients or outputs.
    warmup_passes(mlp_f_theta, rays)

    for ray in rays:
        # Calculate accumulated transmittance Q(u) for all sample points u along the ray
        # Q(u) = exp(-integral from u_n to u of sigma_theta(r(s)) ds)
        # [G] In practice, this is computed discretely over sampled points.
        transmittances = compute_transmittance(ray, mlp_f_theta, u_n, u_f)
        
        # Calculate predicted depth D_hat_theta(r)
        # D_hat = integral from u_n to u_f of Q(u) * sigma_theta(r(u)) * u du
        predicted_depth = compute_predicted_depth(ray, mlp_f_theta, transmittances, u_n, u_f)
        
        # Determine 3D position x
        # x = o + D_hat_theta(r) * d_r
        position = ray.origin + predicted_depth * ray.direction
        
        # Evaluate density at this position using the MLP
        # sigma_theta = f_theta(x, d)
        # [G] The direction d for the MLP input is typically the ray direction.
        _, density = mlp_f_theta(position, ray.direction)
        
        # Step 3 & 4: Check threshold and add Gaussian if met
        if density >= threshold_tau:
            # Initialize Gaussian parameters
            # Covariance matrix Sigma is not explicitly defined in the source for initialization.
            # [G] A common default is an isotropic covariance based on a small radius or 
            # inherited from a nearby point, but the source states "associated Gaussian parameters"
            # without specifying the exact initialization formula for Sigma.
            gaussian = create_gaussian(
                position=position,
                color=mlp_f_theta(position, ray.direction)[0], # Color from MLP
                opacity=1.0, # [G] Default opacity; source does not specify initial opacity.
                covariance=initialize_covariance(position) # [G] Placeholder for covariance init.
            )
            new_gaussians.append(gaussian)
            
    return new_gaussians

def compute_transmittance(ray, mlp, u_n, u_f):
    # Discrete approximation of integral for transmittance
    # Q(u_i) = exp(-sum_{j=0}^{i-1} sigma(theta(r(s_j))) * delta_s)
    samples = ray.sample_points(u_n, u_f)
    q_values = [1.0] # Q(u_n) is 1.0 (no absorption before near plane)
    accumulated_sigma = 0.0
    
    for i in range(len(samples) - 1):
        u = samples[i]
        # Evaluate density at sample point
        # Note: ray.r(s) is the 3D point on the ray at distance s
        point = ray.origin + u * ray.direction
        _, sigma = mlp(point, ray.direction)
        
        # Approximate integral step
        delta_s = samples[i+1] - samples[i]
        accumulated_sigma += sigma * delta_s
        
        # Compute Q for the next point
        q_next = math.exp(-accumulated_sigma)
        q_values.append(q_next)
        
    return q_values

def compute_predicted_depth(ray, mlp, transmittances, u_n, u_f):
    # Discrete approximation of integral for depth
    # D_hat = sum_{i} Q(u_i) * sigma(theta(r(u_i))) * u_i * delta_u
    samples = ray.sample_points(u_n, u_f)
    depth_integral = 0.0
    
    for i in range(len(samples) - 1):
        u = samples[i]
        point = ray.origin + u * ray.direction
        _, sigma = mlp(point, ray.direction)
        
        # Use the transmittance Q(u) at the start of the interval
        # Note: transmittances list has length len(samples)
        q_u = transmittances[i]
        
        delta_u = samples[i+1] - samples[i]
        depth_integral += q_u * sigma * u * delta_u
        
    return depth_integral
```

**Edge Cases and Pitfalls:**
*   **Precision:** The points generated by this strategy lack the precision of LiDAR or SfM points [S1]. A hybrid optimization strategy is required to refine these positions [S1].
*   **Stability:** Warm-up calculations are necessary to prevent instability in the grid-based volume during densification [S1].
*   **Empty Splats:** Without this method, low-texture or distant areas result in blank areas that cannot be rendered accurately due to missing SfM points [S1].

## Subtopics

### HO-Gaussian
The specific implementation of this method within the HO-Gaussian framework, which combines this densification with a hybrid optimization strategy to refine the less precise points generated by the volume [S1].

## Operating characteristics
*   **Time Complexity:** Depends on the number of rays sampled and the depth of the MLP $f_\theta$. Each candidate location requires a forward pass through the MLP and integration steps.
*   **Memory Complexity:** Requires storing the MLP parameters $\theta$ and the grid-based volume representation.
*   **Strengths:** Effectively fills holes in sky, distant, and low-texture regions where SfM fails, preventing empty 2D splats [S1].
*   **Limitations:** The generated points lack the precision of SfM points, necessitating further optimization [S1].
*   **When to use:** In urban scenes or general scenes with significant low-texture or distant regions where initial SfM point clouds are sparse [S1].

## Background
*   **Origin:** Introduced in "HO-Gaussian: Hybrid Optimization of 3D Gaussian Splatting for Urban Scenes" by Li et al. [S1].
*   **History:** Developed to address the limitation of standard 3D Gaussian Splatting, where cloning and splitting operations fail in low-texture or distant areas where initial SfM points are missing [S1].
*   **Usage:** Used in urban scene reconstruction where SfM extraction is poor in certain regions [S1].

## Sources
[S1] HO-Gaussian: Hybrid Optimization of 3D Gaussian Splatting for Urban Scenes, https://doi.org/10.1007/978-3-031-73027-6_2, fetched 2026-10-26
