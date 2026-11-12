# Gaussian Initialization and Warm-up

Also called: Random initialization, Warm-up stage

## Overview
Gaussian Initialization and Warm-up establishes the initial state of 3D Gaussian primitives for radiance field optimization, addressing the instability caused by poor starting positions and incorrect reflectivity assumptions. It sets apart from naive random placement by combining frustum-bounded randomization with a low-reflectivity bias and a static warm-up phase, preventing Gaussians from converging to incorrect local minima in low-reflectivity regions [S1]. Unlike dense initialization strategies that pre-compute millions of primitives, this method starts sparse and relies on subsequent densification or refinement [S1].

## How it works
**Inputs:** Camera frustum bounds (near/far planes), optional color images, raw C-ToF images or phasors, camera poses.
**Outputs:** Initial set of 3D Gaussians with positions, covariances, opacities, and colors.

**Data Structures:**
*   **Gaussian:** A structure containing position $\mathbf{x} \in \mathbb{R}^3$, covariance matrix $\mathbf{\Sigma} \in \mathbb{R}^{3 \times 3}$ (or rotation/scale decomposition), opacity $\alpha \in [0,1]$, and spherical harmonics coefficients for color.
*   **MLP (Optional):** If used for deformation, an MLP with 8 layers, 256 neurons each, and positional encoding frequencies $L_x=10, L_t=10$.

**Steps:**

1.  **Frustum Bounded Randomization:**
    Initialize Gaussian positions $\mathbf{x}$ uniformly at random within the camera frustum, bounded by the near and far clipping planes [S1].
    *   *Note:* Positions are in real-world units. If an MLP is used for deformation, input coordinates to the MLP are rescaled using the unambiguous depth range to ensure most values lie within $[0, 1]$ [S1].

2.  **Low-Reflectivity Bias Initialization:**
    Initialize the reflectivity (or base opacity/reflectivity parameter) to a low value, typically between $0.01$ and $0.1$ [S1].
    *   *Rationale:* High initial reflectivity (e.g., $0.5$) combined with occupancy bias causes Gaussians in low-reflectivity regions to be incorrectly placed far from the camera due to inverse-square falloff assumptions. This leads to interference from closer splats and multiple opacity peaks [S1]. Low initial reflectivity allows Gaussians to be placed closer to the true scene surface, with high-reflectivity Gaussians occluding the scene and preventing interference [S1].

3.  **Static Warm-up Phase:**
    Run an initial optimization stage for $2,000$ iterations assuming the scene is static [S1].
    *   *Purpose:* This helps place many Gaussians into useful positions before introducing dynamics or densification [S1].

4.  **Background Perturbation (Low-Reflectivity Areas):**
    For sensitive low-reflectivity scene areas, randomly change the background signal $q_{bg}$ uniformly between $-1$ and $1$ at each optimization iteration [S1].
    *   *Purpose:* Prevents Gaussians from failing to place on surfaces because the background signal already sufficiently reproduces the C-ToF signal, which would otherwise result in only low-contributing Gaussians [S1].

5.  **MLP Initialization (If applicable):**
    *   MLP biases are initialized to zero.
    *   MLP weights are initialized with Xavier normal initialization.
    *   The final linear layer (mapping 256 features to delta position $\delta \mathbf{x}$) has weights initialized from a normal distribution with standard deviation $10^{-5}$ to ensure stable optimization by starting deformations from small values [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def initialize_gaussians(camera_frustum, num_gaussians, use_mlp=False):
    """
    Initialize 3D Gaussians for radiance field optimization.
    
    Args:
        camera_frustum: Object containing near_plane, far_plane, and projection matrices.
        num_gaussians: Number of Gaussians to initialize.
        use_mlp: Boolean flag indicating if a deformation MLP is used.
        
    Returns:
        gaussians: List of Gaussian objects with initialized parameters.
    """
    gaussians = []
    
    # Step 1: Frustum Bounded Randomization
    # Generate random positions within the camera frustum
    # Implementation detail: Sample uniformly in camera space, then transform to world space
    # bounded by near and far planes.
    positions = sample_random_positions_in_frustum(camera_frustum, num_gaussians)
    
    # If MLP is used, rescale coordinates for MLP input
    if use_mlp:
        unambiguous_depth_range = camera_frustum.far_plane - camera_frustum.near_plane
        positions_for_mlp = rescale_to_range(positions, camera_frustum.near_plane, unambiguous_depth_range)
    else:
        positions_for_mlp = None

    # Step 2: Low-Reflectivity Bias Initialization
    # Initialize reflectivity/opacity to a low value [0.01, 0.1]
    # Using 0.1 as a typical value [S1]
    reflectivity = np.full(num_gaussians, 0.1)
    
    # Initialize other Gaussian parameters (covariance, color) to default/random values
    # Covariance: Typically isotropic or small random scales initially
    covariances = initialize_covariances(num_gaussians)
    colors = initialize_colors(num_gaussians)
    
    for i in range(num_gaussians):
        g = Gaussian(
            position=positions[i],
            covariance=covariances[i],
            opacity=reflectivity[i], # Reflectivity acts as base opacity
            color=colors[i],
            mlp_input=positions_for_mlp[i] if use_mlp else None
        )
        gaussians.append(g)
        
    return gaussians

def initialize_mlp_weights(num_layers=8, hidden_dim=256, final_layer_std=1e-5):
    """
    Initialize MLP weights for deformation field.
    """
    layers = []
    for i in range(num_layers):
        # Xavier normal initialization for hidden layers
        if i < num_layers - 1:
            weight = xavier_normal_init(hidden_dim, hidden_dim)
            bias = np.zeros(hidden_dim)
        else:
            # Final layer: small std dev for stable optimization
            weight = np.random.normal(0, final_layer_std, (1, hidden_dim))
            bias = np.zeros(1)
        layers.append((weight, bias))
    return layers

def randomize_background_signal(current_background_signal):
    """
    Randomly perturb background signal for low-reflectivity areas.
    """
    # Uniformly between -1 and 1 [S1]
    perturbation = np.random.uniform(-1, 1, current_background_signal.shape)
    return current_background_signal + perturbation
```

**Parameters:**
*   **Initial Reflectivity:** $0.01$ to $0.1$ [S1].
*   **Warm-up Iterations:** $2,000$ [S1].
*   **MLP Final Layer Std Dev:** $10^{-5}$ [S1].
*   **Background Perturbation Range:** $[-1, 1]$ [S1].

**Edge Cases and Pitfalls:**
*   **High Reflectivity Initialization:** Setting initial reflectivity to high values (e.g., $0.5$) causes Gaussians in low-reflectivity regions to be placed incorrectly far from the camera due to inverse-square falloff, leading to multiple opacity peaks and interference [S1].
*   **Static Assumption:** The warm-up phase assumes a static scene. If the scene is highly dynamic, this phase may place Gaussians in incorrect positions [S1].
*   **Low-Reflectivity Signal:** Without background perturbation, Gaussians may fail to place on surfaces in low-reflectivity areas because the background signal already reproduces the C-ToF signal [S1].

## Subtopics

### [Dense Initialization (DI)](10026-dense-initialization-di.md)
Dense Initialization (DI) is a Gaussian initialization strategy that generates millions of small Gaussians to densely cover all visible surfaces at the outset, rather than starting with a sparse set and progressively refining it. It solves the problem of slow convergence and the computational overhead of iterative densification stages used in standard 3D Gaussian Splatting. DI sets itself apart from its sibling, Recurrent Gaussian Update, by eliminating the need for densification entirely through high-density matching, which also benefits ray tracing performance by reducing the workload of irrelevant primitives.

### [Recurrent Gaussian Update](10974-recurrent-gaussian-update.md)
Recurrent Gaussian Update is a gradient-free, feed-forward refinement method that iteratively corrects initial 3D Gaussian predictions by predicting incremental updates to Gaussian parameters and hidden states based on rendering errors from input views. It sets itself apart from standard 3D Gaussian Splatting (3DGS) optimization by eliminating gradient computation, achieving a 53× speedup in reconstruction time and converging in 4 iterations instead of 30,000. Unlike single-step feed-forward models, it uses a recurrent error-correction loop to achieve higher parameter efficiency, where a smaller recurrent model outperforms significantly larger single-step models.

### Low-Reflectivity Bias
Low-Reflectivity Bias is a specific initialization heuristic that sets initial Gaussian reflectivity to a low value (0.01 to 0.1) [S1]. High initial reflectivity (e.g., 0.5) causes Gaussians in low-reflectivity regions to be incorrectly placed far away due to inverse-square falloff, leading to multiple opacity peaks and interference from closer splats [S1]. Low initial reflectivity allows Gaussians in low-reflectivity regions to be placed more correctly (closer fit to true scene). Gaussians in high-reflectivity regions are placed closer to the camera, occluding the scene, which prevents interference and allows more iterations to optimize reflectivity and position [S1]. Combining occupancy and low-reflectivity biases is effective for both low and high reflectivity regions, whereas using only occupancy bias harms low-reflectivity regions [S1].

### APMGSRN Initialization
APMGSRN Initialization is a method for initializing transformation matrices and feature grids in scene representation networks [S2]. Transformation matrices are initialized to cover a near global domain with small random shears, rotations, and translations. Diagonals are sampled from $N(1, 0.05)$, and remaining entries in the first three rows are sampled from $N(0, 0.05)$ [S2]. Feature grids are initialized from $U(-0.0001, 0.0001)$, and decoder network weights follow Glorot and Bengio initialization [S2]. This ensures each entry in the transformation matrix is non-zero, guaranteeing gradients can be used to update each entry [S2].

## Operating characteristics
*   **Time Complexity:** The initialization step is $O(N)$ where $N$ is the number of Gaussians. The warm-up phase adds $2,000$ iterations of optimization, which is a fixed cost independent of scene complexity but dependent on the number of Gaussians and rendering resolution [S1].
*   **Memory Complexity:** $O(N)$ for storing Gaussian parameters. If an MLP is used, additional memory is required for MLP weights [S1].
*   **Strengths:** Prevents convergence to incorrect local minima in low-reflectivity regions. The warm-up phase stabilizes the optimization process by placing Gaussians in useful positions early [S1].
*   **Limitations:** The static warm-up assumption may be invalid for highly dynamic scenes. Random initialization may still require densification to achieve high-fidelity reconstruction [S1].
*   **When to Use:** Standard 3D Gaussian Splatting optimization, especially in scenes with varying reflectivity.
*   **When Not to Use:** Highly dynamic scenes where the static warm-up assumption is invalid. Dense scenes where pre-computed dense initialization might be more efficient [S1].

## Background
The method originates from the optimization of dynamic radiance fields using 3D Gaussians, specifically in the context of Time-of-Flight (C-ToF) imaging [S1]. The low-reflectivity bias and warm-up phase were introduced to address the instability of inverse rendering with Gaussians, particularly in low-reflectivity regions where standard initialization fails [S1]. The APMGSRN initialization method was introduced for large-scale scene representation networks to ensure stable training and effective gradient flow [S2].

## Sources
[S1] Time of the Flight of the Gaussians: Optimizing Depth Indirectly in Dynamic Radiance Fields, https://doi.org/10.1109/cvpr52734.2025.01958, fetched 2026-10-26
[S2] Adaptively Placed Multi-Grid Scene Representation Networks for Large-Scale Data Visualization, https://doi.org/10.1109/tvcg.2023.3327194, fetched 2026-10-26
