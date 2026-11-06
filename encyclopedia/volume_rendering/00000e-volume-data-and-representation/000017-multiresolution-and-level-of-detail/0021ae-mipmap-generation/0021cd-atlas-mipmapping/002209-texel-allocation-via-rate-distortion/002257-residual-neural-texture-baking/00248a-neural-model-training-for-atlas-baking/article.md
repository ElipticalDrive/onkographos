# Neural Model Training for Atlas Baking

Also called: NeST-Splatting Configuration, Stage 1 Training

## Overview
This method defines the training configuration for a neural appearance model used to generate a residual texture atlas for splatting-based rendering [S1, unofficial]. It solves the problem of learning a high-fidelity, view-dependent residual color field from scratch, which is subsequently baked into a static spatial atlas to remove neural network inference costs during rendering [S1, unofficial]. It is distinguished from its sibling [Residual Neural Texture Baking](2709-residual-neural-texture-baking.md) by providing the specific hyperparameters and optimization schedule required to train the underlying hash-grid and MLP model before the baking step occurs.

## How it works
The method trains a neural model using a 2D surfel rasterizer. The inputs are a set of multi-view images and an initial set of 3D surfels. The output is a trained neural network (hash grid + MLP) and surfel parameters, which are then used to bake a residual texture atlas.

### Data Structures
*   **Surfel**: Contains position, opacity, scale, rotation, shape parameters, and per-surfel color coefficients.
*   **Spherical Voronoi (SV) Basis**: For each surfel, $K=7$ sites are stored. Each site has a position on the sphere and a color value.
*   **Hash Grid**: A multi-resolution pyramid with 4 levels (the four finest of a six-level pyramid). Each level has $2^{21}$ entries. Each entry stores 4 features. The input coordinates are contracted to $[-5, 5]$ using scene contraction.
*   **MLP**: A fused MLP with architecture $16 \rightarrow 16 \rightarrow 16 \rightarrow 3$. It takes the hashed features as input and outputs a 3-channel residual color $\rho$.

### Formulas
1.  **Primitive Rasterization**: 2D surfels are rasterized using a compact Beta kernel with support $k = 3 \sigma$ [S1, unofficial]. A screen-space low-pass filter is applied.
2.  **Base Color**: The base color $C_{base}$ is computed using the Spherical Voronoi basis:
    $$ C_{base} = \text{ReLU}(f_{SV} + 0.5) $$
    where $f_{SV}$ is the interpolated value from the SV basis [S1, unofficial].
3.  **Fragment Color**: The final predicted color $C_{pred}$ is:
    $$ C_{pred} = \text{ReLU}(C_{base} + \rho) $$
    where $\rho$ is the residual output from the MLP queried at the ray-surfel intersection [S1, unofficial].
4.  **Loss Function**:
    $$ \mathcal{L} = \mathcal{L}_1 + 0.2(1 - \text{SSIM}) + \lambda \mathcal{L}_\beta $$
    where $\mathcal{L}_1$ is the L1 loss, SSIM is the Structural Similarity Index, and $\mathcal{L}_\beta$ is the falloff regularizer [S1, unofficial].

### Parameters and Schedule
*   **Optimizer**: Adam [S1, unofficial].
*   **Total Iterations**: 35,000 [S1, unofficial].
*   **Learning Rates**:
    *   Position: $1.6 \times 10^{-4}$ decaying exponentially to $1.6 \times 10^{-6}$.
    *   Opacity: $0.05$.
    *   Scale: $5 \times 10^{-3}$.
    *   Rotation: $1 \times 10^{-3}$.
    *   Shape: $1 \times 10^{-3}$.
    *   SV Sites: $2 \times 10^{-3}$ decaying to $2 \times 10^{-4}$.
    *   SV Colors: $8 \times 10^{-4}$.
    *   SV Temperature: $6 \times 10^{-3}$.
    *   Hash Grid: $2 \times 10^{-2}$.
    *   MLP: $1 \times 10^{-3}$.
*   **Warm-up and Freezing**: SV sites and temperatures warm up over the first 1,000 iterations and are frozen for the last 3,000 iterations [S1, unofficial].
*   **Update Frequency**: Hash grid and MLP are updated every iteration until 5,000 iterations, then every tenth iteration. Frozen iterations reuse cached per-fragment residuals [S1, unofficial].

### Densification and Pruning
*   **Initial Phase**: Densification and pruning occur every 500 iterations until the surfel count reaches 20,000.
    *   Gradient thresholds: $1.5 \times 10^{-4}$ (signed) and $6 \times 10^{-4}$ (absolute).
    *   Importance threshold: 30 views.
    *   Loss threshold: 0.1.
*   **Late Pruning**: Every 3,000 iterations, from 20,000 to 30,000 surfels, a score-based prune is performed with a threshold of 0.9 [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialization
surfels = initialize_surfels_from_point_cloud()
hash_grid = HashGrid(levels=4, entries_per_level=2**21, features=4)
mlp = MLP(layers=[16, 16, 16, 3], activation=ReLU)
sv_basis = SphericalVoronoi(k=7)

# Optimizers
opt_position = Adam(surfels.positions, lr=1.6e-4, decay=True)
opt_opacity = Adam(surfels.opacity, lr=0.05)
opt_scale = Adam(surfels.scale, lr=5e-3)
opt_rotation = Adam(surfels.rotation, lr=1e-3)
opt_shape = Adam(surfels.shape, lr=1e-3)
opt_sv_sites = Adam(sv_basis.sites, lr=2e-3, decay=True)
opt_sv_colors = Adam(sv_basis.colors, lr=8e-4)
opt_sv_temp = Adam(sv_basis.temperature, lr=6e-3)
opt_hash = Adam(hash_grid.parameters, lr=2e-2)
opt_mlp = Adam(mlp.parameters, lr=1e-3)

# Training Loop
for iteration in range(35000):
    # 1. Densification and Pruning
    if iteration % 500 == 0 and iteration < 15000: # Until 20k
        densify_and_prune(surfels, 
                          grad_threshold_signed=1.5e-4, 
                          grad_threshold_abs=6e-4, 
                          importance_threshold=30, 
                          loss_threshold=0.1)
    elif iteration % 3000 == 0 and 15000 <= iteration < 30000: # Late prune
        late_prune(surfels, score_threshold=0.9)

    # 2. Forward Pass
    # Sample a batch of views
    views = sample_views()
    
    # Rasterize surfels to get screen-space coverage
    rasterized = rasterize_2d_surfels(surfels, views, kernel=BetaKernel(k=3*sigma))
    
    # Compute base color from SV basis
    f_sv = compute_sv_interpolation(rasterized, sv_basis)
    c_base = relu(f_sv + 0.5)
    
    # Query hash grid and MLP for residual
    # Only update hash/MLP if iteration < 5000 or iteration % 10 == 0
    if iteration < 5000 or iteration % 10 == 0:
        coords = contract_to_neg5_5(rasterized.intersection_points)
        features = hash_grid.query(coords)
        rho = mlp(features)
    else:
        rho = cached_residuals[rasterized.fragment_ids]
        
    # Compute final color
    c_pred = relu(c_base + rho)
    
    # 3. Loss Computation
    loss_l1 = l1_loss(c_pred, views.target_color)
    loss_ssim = 1 - ssim(c_pred, views.target_color)
    loss_beta = compute_falloff_regularizer(surfels.shape) # L_beta
    total_loss = loss_l1 + 0.2 * loss_ssim + 0.005 * loss_beta
    
    # 4. Backward Pass and Update
    # Warm-up for SV sites/temp: first 1000 iters
    # Freeze SV sites/temp: last 3000 iters (iter > 32000)
    freeze_sv = iteration > 32000
    
    if not freeze_sv:
        opt_sv_sites.zero_grad()
        opt_sv_colors.zero_grad()
        opt_sv_temp.zero_grad()
        total_loss.backward(retain_graph=True) # Retain for other grads if needed
        opt_sv_sites.step()
        opt_sv_colors.step()
        opt_sv_temp.step()
        
    # Update other parameters
    opt_position.zero_grad()
    opt_opacity.zero_grad()
    opt_scale.zero_grad()
    opt_rotation.zero_grad()
    opt_shape.zero_grad()
    opt_hash.zero_grad()
    opt_mlp.zero_grad()
    
    total_loss.backward()
    opt_position.step()
    opt_opacity.step()
    opt_scale.step()
    opt_rotation.step()
    opt_shape.step()
    opt_hash.step()
    opt_mlp.step()
```

### Edge Cases and Pitfalls
*   **Precision**: The hash grid uses scene contraction to $[-5, 5]$. Ensure coordinate normalization matches this range [S1, unofficial].
*   **Freezing**: SV sites and temperatures must be frozen in the last 3,000 iterations to prevent instability [S1, unofficial].
*   **Update Frequency**: After 5,000 iterations, hash and MLP updates are sparse (every 10th iteration). Cached residuals must be maintained correctly to avoid using stale data [S1, unofficial].
*   **Kernel Choice**: The Beta kernel is required for the falloff regularizer $\mathcal{L}_\beta$. Using a Gaussian kernel (HQ variant) makes the regularizer inapplicable [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: Training takes 35,000 iterations. The exact time depends on the number of surfels and views, but the method is designed for ultrafast baking post-training [S1, unofficial].
*   **Memory Complexity**: The hash grid dominates memory usage with $4 \times 2^{21}$ entries of 4 features each. Surfel count is capped at 30,000 [S1, unofficial].
*   **Hardware**: Timings are reported on a single RTX 5090 [S1, unofficial].
*   **Strengths**: Produces a high-fidelity residual model suitable for baking into a static atlas, enabling fast rendering without neural inference [S1, unofficial].
*   **Limitations**: Requires careful tuning of learning rates and pruning thresholds. The Beta kernel regularizer is specific to this configuration [S1, unofficial].

## Background
This configuration is part of the "Bake It Till You Make It" method, which uses a 2D surfel rasterizer with a hash-grid and MLP appearance model [S1, unofficial]. It is implemented on top of the NeST-Splatting codebase [S1, unofficial]. The method is designed to be scene-agnostic, using one configuration for every scene [S1, unofficial].

## Sources
[S1] Bake It Till You Make It: Ultrafast Spatial Texture-Atlas Splatting, https://doi.org/10.48550/arXiv.2607.13808, fetched 2026-10-29 (unofficial)
