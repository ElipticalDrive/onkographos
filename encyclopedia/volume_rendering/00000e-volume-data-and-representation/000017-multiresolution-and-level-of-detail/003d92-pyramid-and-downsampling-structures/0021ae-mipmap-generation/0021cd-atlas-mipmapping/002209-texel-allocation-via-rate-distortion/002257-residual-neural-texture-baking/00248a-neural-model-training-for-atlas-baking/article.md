# Neural Model Training for Atlas Baking

Also called: NeST-Splatting Configuration, Stage 1 Training

## Overview
This method defines the training configuration for a neural appearance model used to generate a residual texture atlas for splatting-based rendering [S1, unofficial]. It solves the problem of learning a high-fidelity, view-dependent residual color field from scratch, which is subsequently baked into a static spatial atlas to remove neural network inference costs during rendering [S1, unofficial]. It is distinguished from its sibling [Residual Neural Texture Baking](2709-residual-neural-texture-baking.md) by providing the specific hyperparameters and optimization schedule required to train the underlying hash-grid and MLP model before the baking step occurs [S1, unofficial].

## How it works
The method trains a neural model using a 2D surfel rasterizer [S1, unofficial]. The inputs are a set of multi-view images and an initial set of 3D surfels [S1, unofficial]. The output is a trained neural network (hash grid + MLP) and surfel parameters, which are then used to bake a residual texture atlas [S1, unofficial].

### Data Structures
*   **Surfel**: Contains position, opacity, scale, rotation, shape parameters, and per-surfel color coefficients [S1, unofficial].
*   **Spherical Voronoi (SV) Basis**: For each surfel, $K=7$ sites are stored [S1, unofficial]. Each site has a position on the sphere and a color value [S1, unofficial].
*   **Hash Grid**: A multi-resolution pyramid with 4 levels (the four finest of a six-level pyramid) [S1, unofficial]. Each level has $2^{21}$ entries [S1, unofficial]. Each entry stores 4 features [S1, unofficial]. The input coordinates are contracted to $[-5, 5]$ using scene contraction [S1, unofficial].
*   **MLP**: A fused MLP with architecture $16 \rightarrow 16 \rightarrow 16 \rightarrow 3$ [S1, unofficial]. It takes the hashed features as input and outputs a 3-channel residual color $\rho$ [S1, unofficial].

### Formulas
1.  **Primitive Rasterization**: 2D surfels are rasterized using a compact Beta kernel with support $k = 3 \sigma$ [S1, unofficial]. A screen-space low-pass filter is applied [S1, unofficial].
2.  **Base Color**: The base color $C_{base}$ is computed using the Spherical Voronoi basis:
    $$ C_{base} = \text{ReLU}(f_{SV} + 0.5) $$
    where $f_{SV}$ is the interpolated value from the SV basis [S1, unofficial].
3.  **Fragment Color**: The final predicted color $C_{pred}$ is:
    $$ C_{pred} = \text{ReLU}(C_{base} + \rho) $$
    where $\rho$ is the residual output from the MLP queried at the ray-surfel intersection [S1, unofficial].
4.  **Loss Function**:
    $$ \mathcal{L} = \mathcal{L}_1 + 0.2(1 - \text{SSIM}) + \lambda \mathcal{L}_\beta $$
    where $\mathcal{L}_1$ is the L1 loss, SSIM is the Structural Similarity Index, and $\mathcal{L}_\beta$ is the falloff regularizer with $\lambda = 0.005$ [S1, unofficial].

### Parameters and Schedule
*   **Optimizer**: Adam [S1, unofficial].
*   **Total Iterations**: 35,000 [S1, unofficial].
*   **Learning Rates**:
    *   Position: $1.6 \times 10^{-4}$ decaying exponentially to $1.6 \times 10^{-6}$ [S1, unofficial].
    *   Opacity: $0.05$ [S1, unofficial].
    *   Scale: $5 \times 10^{-3}$ [S1, unofficial].
    *   Rotation: $1 \times 10^{-3}$ [S1, unofficial].
    *   Shape: $1 \times 10^{-3}$ [S1, unofficial].
    *   SV Sites: $2 \times 10^{-3}$ decaying to $2 \times 10^{-4}$ [S1, unofficial].
    *   SV Colors: $8 \times 10^{-4}$ [S1, unofficial].
    *   SV Temperature: $6 \times 10^{-3}$ [S1, unofficial].
    *   Hash Grid: $2 \times 10^{-2}$ [S1, unofficial].
    *   MLP: $1 \times 10^{-3}$ [S1, unofficial].
*   **Warm-up and Freezing**: SV sites and temperatures warm up over the first 1,000 iterations and are frozen for the last 3,000 iterations [S1, unofficial].
*   **Update Frequency**: Hash grid and MLP are updated every iteration until 5,000 iterations, then every tenth iteration. Frozen iterations reuse cached per-fragment residuals [S1, unofficial].

### Densification and Pruning
*   **Initial Phase**: Densification and pruning occur every 500 iterations until the surfel count reaches 20,000 [S1, unofficial].
    *   Gradient thresholds: $1.5 \times 10^{-4}$ (signed) and $6 \times 10^{-4}$ (absolute) [S1, unofficial].
    *   Importance threshold: 30 views [S1, unofficial].
    *   Loss threshold: 0.1 [S1, unofficial].
*   **Late Pruning**: Every 3,000 iterations, from 20,000 to 30,000 surfels, a score-based prune is performed with a threshold of 0.9 [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Data structures
class Surfel:
    def __init__(self, pos, opacity, scale, rot, shape, sv_sites, sv_colors, sv_temp):
        self.pos = pos
        self.opacity = opacity
        self.scale = scale
        self.rot = rot
        self.shape = shape
        self.sv_sites = sv_sites
        self.sv_colors = sv_colors
        self.sv_temp = sv_temp

class HashGrid:
    def __init__(self, levels, entries_per_level, features):
        self.levels = levels
        self.entries_per_level = entries_per_level
        self.features = features
        self.data = [{} for _ in range(levels)]
    def query(self, coords):
        return [0.0] * 4

class MLP:
    def __init__(self, layers):
        self.layers = layers
    def forward(self, features):
        return [0.0, 0.0, 0.0]

def initialize_surfels_from_point_cloud():
    return []

def sample_views():
    return {"target_color": [0.0, 0.0, 0.0]}

def rasterize_2d_surfels(surfels, views, kernel):
    return [{"intersection_points": [0.0, 0.0, 0.0], "fragment_id": 0}]

def compute_sv_interpolation(fragments, sv_basis):
    return 0.0

def contract_to_neg5_5(coords):
    return coords

def relu(x):
    return max(0.0, x)

def l1_loss(pred, target):
    return sum(abs(p - t) for p, t in zip(pred, target))

def ssim(pred, target):
    return 1.0

def compute_falloff_regularizer(shape):
    return 0.0

def compute_gradients(loss):
    # Standard autograd pass computing gradients for all tracked parameters
    pass

def apply_optimizer(params, lr):
    for p in params:
        p -= lr * p.grad

def densify_and_prune(surfels, grad_threshold_signed, grad_threshold_abs, importance_threshold, loss_threshold):
    # Applies multi-view densification and pruning criteria
    return surfels

def late_prune(surfels, score_threshold):
    # Removes surfels below score threshold
    return surfels

# Initialization
surfels = initialize_surfels_from_point_cloud()
hash_grid = HashGrid(levels=4, entries_per_level=2**21, features=4)
mlp = MLP(layers=[16, 16, 16, 3])
sv_basis = {"sites": [], "colors": [], "temperature": 0.0}
cached_residuals = {}
surfel_count = len(surfels)

# Training Loop
for iteration in range(35000):
    # 1. Densification and Pruning
    if iteration % 500 == 0 and surfel_count < 20000:
        surfels = densify_and_prune(surfels, 
                                    grad_threshold_signed=1.5e-4, 
                                    grad_threshold_abs=6e-4, 
                                    importance_threshold=30, 
                                    loss_threshold=0.1)
        surfel_count = len(surfels)
    elif iteration % 3000 == 0 and 20000 <= surfel_count < 30000:
        surfels = late_prune(surfels, score_threshold=0.9)
        surfel_count = len(surfels)

    # 2. Forward Pass
    views = sample_views()
    rasterized = rasterize_2d_surfels(surfels, views, kernel="Beta")
    f_sv = compute_sv_interpolation(rasterized, sv_basis)
    c_base = relu(f_sv + 0.5)

    update_hash_mlp = (iteration < 5000) or (iteration % 10 == 0)
    if update_hash_mlp:
        coords = contract_to_neg5_5(rasterized[0]["intersection_points"])
        features = hash_grid.query(coords)
        rho = mlp.forward(features)
        for frag in rasterized:
            cached_residuals[frag["fragment_id"]] = rho
    else:
        rho = cached_residuals[rasterized[0]["fragment_id"]]

    c_pred = relu(c_base + rho)

    # 3. Loss Computation
    loss_l1 = l1_loss(c_pred, views["target_color"])
    loss_ssim = 1 - ssim(c_pred, views["target_color"])
    loss_beta = compute_falloff_regularizer(surfels[0].shape)
    total_loss = loss_l1 + 0.2 * loss_ssim + 0.005 * loss_beta

    # 4. Backward Pass and Update
    freeze_sv = iteration > 32000
    warmup_sv = iteration < 1000

    compute_gradients(total_loss)

    # Update SV parameters
    if not freeze_sv:
        apply_optimizer(sv_basis["sites"], 2e-3 if warmup_sv else 2e-4)
        apply_optimizer(sv_basis["colors"], 8e-4)
        apply_optimizer(sv_basis["temperature"], 6e-3)

    # Update other parameters
    apply_optimizer([s.pos for s in surfels], 1.6e-4)
    apply_optimizer([s.opacity for s in surfels], 0.05)
    apply_optimizer([s.scale for s in surfels], 5e-3)
    apply_optimizer([s.rot for s in surfels], 1e-3)
    apply_optimizer([s.shape for s in surfels], 1e-3)
    apply_optimizer(hash_grid.data, 2e-2)
    apply_optimizer(mlp.layers, 1e-3)
```

### Edge Cases and Pitfalls
*   **Precision**: The hash grid uses scene contraction to $[-5, 5]$ [S1, unofficial].
*   **Freezing**: SV sites and temperatures are frozen for the last 3,000 iterations [S1, unofficial].
*   **Update Frequency**: After 5,000 iterations, hash and MLP updates are sparse (every 10th iteration). Frozen iterations reuse cached per-fragment residuals [S1, unofficial].
*   **Kernel Choice**: The Beta kernel is required for the falloff regularizer $\mathcal{L}_\beta$. Using a Gaussian kernel (HQ variant) makes the regularizer inapplicable [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: Training takes 35,000 iterations [S1, unofficial].
*   **Memory Complexity**: The hash grid dominates memory usage with $4 \times 2^{21}$ entries of 4 features each [S1, unofficial].
*   **Hardware**: Timings are reported on a single RTX 5090 [S1, unofficial].
*   **Strengths**: Produces a high-fidelity residual model suitable for baking into a static atlas, enabling fast rendering without neural inference [S1, unofficial].
*   **Limitations**: Requires careful tuning of learning rates and pruning thresholds. The Beta kernel regularizer is specific to this configuration [S1, unofficial].

## Background
This configuration is part of the "Bake It Till You Make It" method, which uses a 2D surfel rasterizer with a hash-grid and MLP appearance model [S1, unofficial]. It is implemented on top of the NeST-Splatting codebase [S1, unofficial]. The method is designed to be scene-agnostic, using one configuration for every scene [S1, unofficial].

## Sources
[S1] Bake It Till You Make It: Ultrafast Spatial Texture-Atlas Splatting, https://doi.org/10.48550/arXiv.2607.13808, fetched 2026-10-29 (unofficial)
