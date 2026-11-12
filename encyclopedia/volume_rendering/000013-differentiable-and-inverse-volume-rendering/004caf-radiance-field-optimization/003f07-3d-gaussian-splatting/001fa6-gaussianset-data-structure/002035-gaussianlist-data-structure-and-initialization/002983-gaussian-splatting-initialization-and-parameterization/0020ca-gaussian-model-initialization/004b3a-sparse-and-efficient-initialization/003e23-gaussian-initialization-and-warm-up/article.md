# Gaussian Initialization and Warm-up

Also called: Random initialization, Warm-up stage

## Overview
Gaussian Initialization and Warm-up establishes the initial state of 3D Gaussian primitives for differentiable volume rendering, specifically addressing the challenges of converging from sparse or random starting positions [S1, S2]. It solves the problem of slow convergence and incorrect placement of Gaussians in low-reflectivity regions or complex dynamic scenes by providing a structured, low-bias starting point followed by a static optimization phase [S1]. It sets itself apart from standard 3D Gaussian Splatting (3DGS) by explicitly decoupling the initial placement from the final high-frequency detail, using a warm-up stage to stabilize positions before complex deformation or densification begins [S1]. Trade-offs include the computational cost of the initial warm-up iterations and the sensitivity of the random seed to the final convergence quality [S1].

## How it works
The method combines geometric initialization, parameter biasing, and a temporal warm-up phase. It is primarily designed for dynamic scenes (e.g., C-ToF data) but includes variants for static large-scale neural field representations [S1, S2].

### Inputs and Outputs
*   **Inputs:** Camera poses, near/far plane distances, scene bounding box (or unambiguous depth range), optional color images, raw signal images (e.g., C-ToF quads).
*   **Outputs:** A set of $N$ 3D Gaussians, each defined by position $\mathbf{x}$, covariance $\Sigma$ (or rotation/scale), opacity $\alpha$, and spherical harmonics coefficients $\mathbf{c}$ (or feature vector).

### Data Structures
Each Gaussian $i$ stores:
*   $\mathbf{x}_i \in \mathbb{R}^3$: Center position in world coordinates.
*   $\mathbf{s}_i \in \mathbb{R}^3$: Scale vector (diagonal of covariance).
*   $\mathbf{q}_i \in \mathbb{R}^4$: Rotation quaternion.
*   $\alpha_i \in \mathbb{R}$: Opacity.
*   $\mathbf{c}_i$: Color/Feature coefficients.
*   (Optional) $\mathbf{v}_i$: Velocity or deformation parameters if using a dynamic model [S1].

### Initialization Steps

1.  **Geometric Placement:**
    Initialize $N$ Gaussians randomly within the camera frustum. The positions are bounded by the near and far planes of the camera view [S1].
    *   For neural field variants (e.g., APMGSRN), initialize transformation matrices covering a near-global domain. Diagonals are sampled from $\mathcal{N}(1, 0.05)$, and remaining entries in the first three rows from $\mathcal{N}(0, 0.05)$ [S2].
    *   Initialize feature grids from a uniform distribution $\mathcal{U}(-0.0001, 0.0001)$ [S2].

2.  **Reflectivity and Opacity Biasing:**
    Set initial reflectivity (or opacity contribution) to a low value, typically between $0.01$ and $0.1$ [S1].
    *   *Rationale:* High initial reflectivity (e.g., $0.5$) causes Gaussians in low-reflectivity regions to be placed incorrectly far from the camera due to inverse-square falloff assumptions, leading to multiple opacity peaks and interference from closer splats [S1]. Low initial reflectivity allows Gaussians to be placed closer to the true surface, with high-reflectivity Gaussians occluding the scene and preventing interference [S1].

3.  **Coordinate Rescaling (Dynamic Scenes):**
    If the scene involves large real-world units (e.g., C-ToF depth), rescale input coordinates to the MLP (if used) using the unambiguous depth range. This ensures most values lie within $[0, 1]$ to ease optimization [S1].
    *   MLP weights are initialized with Xavier normal initialization.
    *   Final linear layer weights (mapping to delta position) are initialized from $\mathcal{N}(0, 10^{-5})$ to ensure stable, small initial deformations [S1].

4.  **Warm-up Stage:**
    Run an initial optimization phase for $2,000$ iterations [S1].
    *   Assume the scene is static during this phase.
    *   This helps place many Gaussians into useful positions before introducing complex dynamics or densification [S1].
    *   Loss function: $\mathcal{L} = \alpha \mathcal{L}_q + \mathcal{L}_c + \beta \mathcal{L}_f$, where $\mathcal{L}_q$ is the raw quad reconstruction loss (L2/SSIM), $\mathcal{L}_c$ is optional color loss, and $\mathcal{L}_f$ is a fidelity term [S1]. Typical $\alpha=5$ [S1].

5.  **Random Background Perturbation (Low-Reflectivity Areas):**
    For sensitive low-reflectivity scene areas, randomly change the background signal $q_{bg}$ uniformly between $-1$ and $1$ at each optimization iteration [S1].
    *   *Rationale:* A low, static background contribution can cause Gaussians to fail to place on surfaces because the background already reproduces the signal. Randomizing the background forces Gaussians to contribute to the fit [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def initialize_gaussians_and_warmup(cameras, scene_bounds, num_gaussians, 
                                    near_plane, far_plane, warmup_iters=2000):
    """
    Initializes 3D Gaussians and runs a static warm-up phase.
    """
    gaussians = []
    
    # Step 1: Geometric Placement
    for i in range(num_gaussians):
        # Random position within camera frustum bounded by near/far planes
        # Assuming uniform distribution within the bounding box of the frustum
        pos = random_point_in_frustum(cameras, near_plane, far_plane)
        
        # Step 2: Low Reflectivity Bias
        # Initialize reflectivity/opacity to low value to prevent far-away placement
        reflectivity = random.uniform(0.01, 0.1)
        
        # Initialize other parameters (scale, rotation, color) to defaults or small random values
        scale = np.array([0.1, 0.1, 0.1]) # Example default scale
        rotation = np.array([1.0, 0.0, 0.0, 0.0]) # Identity quaternion
        color = np.zeros(3)
        
        gaussians.append({
            'pos': pos,
            'reflectivity': reflectivity,
            'scale': scale,
            'rotation': rotation,
            'color': color
        })
        
    # Step 3: Coordinate Rescaling (if MLP is used for deformation)
    # Rescale scene coordinates to unambiguous depth range [0, 1]
    # This is handled internally by the MLP forward pass, not explicit here
    
    # Step 4 & 5: Warm-up Loop
    for iteration in range(warmup_iters):
        # Random background perturbation for low-reflectivity areas
        # This modifies the target signal for specific pixels/regions
        background_noise = random.uniform(-1, 1)
        
        # Render current Gaussians
        rendered_images = render_gaussians(gaussians, cameras)
        
        # Compute Loss
        # L = alpha * L_quad + L_color + beta * L_fidelity
        # L_quad uses L2 and SSIM on raw C-ToF quads
        loss = compute_loss(rendered_images, target_images, background_noise)
        
        # Backpropagate and update Gaussians
        # Assume static scene during warm-up (no deformation updates)
        update_gaussians(gaussians, loss)
        
    return gaussians
```

### Edge Cases and Pitfalls
*   **High Reflectivity Initialization:** Using high initial reflectivity (e.g., 0.5) leads to incorrect placement in low-reflectivity regions due to inverse-square falloff, causing multiple opacity peaks [S1].
*   **Static Background:** A static, low background signal prevents Gaussians from placing on surfaces in low-reflectivity areas. Randomizing the background signal is required [S1].
*   **MLP Initialization:** Final layer weights must be small ($10^{-5}$) to prevent large initial deformations [S1].
*   **Transformation Matrix Zeros:** In neural field variants, ensuring transformation matrix entries are non-zero (via $\mathcal{N}(0, 0.05)$) guarantees gradients can update every entry [S2].

## Subtopics

### [Dense Initialization (DI)](10026-dense-initialization-di.md)
Dense Initialization (DI) generates millions of small Gaussians to densely cover all visible surfaces at the outset, rather than starting with a sparse set and progressively refining it. It solves the problem of slow convergence and the computational overhead of iterative densification stages used in standard 3D Gaussian Splatting. DI sets itself apart from its sibling, Recurrent Gaussian Update, by eliminating the need for densification entirely through high-density matching, which also benefits ray tracing performance by reducing the workload of irrelevant primitives.

### [Recurrent Gaussian Update](10974-recurrent-gaussian-update.md)
Recurrent Gaussian Update is a gradient-free, feed-forward refinement method that iteratively corrects initial 3D Gaussian predictions by predicting incremental updates to Gaussian parameters and hidden states based on rendering errors from input views. It sets itself apart from standard 3D Gaussian Splatting (3DGS) optimization by eliminating gradient computation, achieving a 53× speedup in reconstruction time and converging in 4 iterations instead of 30,000. Unlike single-step feed-forward models, it uses a recurrent error-correction loop to achieve higher parameter efficiency, where a smaller recurrent model outperforms significantly larger single-step models.

### Low-Reflectivity Bias
A specific heuristic within the initialization process where initial reflectivity is set to a low value (0.01 to 0.1) [S1]. High initial reflectivity causes Gaussians in low-reflectivity regions to be incorrectly placed far away due to inverse-square falloff, leading to multiple opacity peaks and interference from closer splats [S1]. Low initial reflectivity allows Gaussians in low-reflectivity regions to be placed more correctly (closer fit to true scene). Gaussians in high-reflectivity regions are placed closer to the camera, occluding the scene, which prevents interference and allows more iterations to optimize reflectivity and position [S1]. Combining occupancy and low-reflectivity biases is effective for both low and high reflectivity regions, whereas using only occupancy bias harms low-reflectivity regions [S1].

### APMGSRN Initialization
A variant for Adaptively Placed Multi-Grid Scene Representation Networks [S2]. Transformation matrices are initialized to cover a near global domain with small random shears, rotations, and translations. Diagonals are sampled from $\mathcal{N}(1, 0.05)$, and remaining entries in the first three rows from $\mathcal{N}(0, 0.05)$ [S2]. Feature grids are initialized from $\mathcal{U}(-0.0001, 0.0001)$, and decoder network weights follow Glorot and Bengio [S2]. This ensures each entry in the transformation matrix is non-zero, guaranteeing gradients can be used to update each entry [S2].

## Operating characteristics
*   **Time Complexity:** The warm-up stage adds a fixed cost of $2,000$ iterations [S1]. For dynamic scenes, this is a one-time overhead before the main optimization. For neural field variants, initialization is fast, but the subsequent training time is reduced significantly (e.g., to 40 seconds in APMGSRN) due to better starting points [S2].
*   **Memory Complexity:** Depends on the number of Gaussians $N$. Dense Initialization (DI) requires significantly more memory due to millions of primitives [S1].
*   **Performance:** The warm-up stage improves convergence speed and final quality, particularly in dynamic scenes [S1]. Low-reflectivity bias prevents artifacts in dark regions [S1].
*   **Strengths:** Robust to initial random placement; handles low-reflectivity regions effectively; reduces need for complex densification heuristics.
*   **Limitations:** Requires careful tuning of initial reflectivity and background noise; warm-up iterations add to total training time; sensitive to camera pose accuracy [S1].
*   **When to use:** When optimizing dynamic radiance fields or scenes with varying reflectivity; when standard 3DGS densification is too slow or unstable.
*   **When not to use:** For very simple, static, high-reflectivity scenes where standard 3DGS initialization suffices.

## Background
The method originates from the optimization of dynamic radiance fields using 3D Gaussians, particularly for Time-of-Flight (C-ToF) data [S1]. The concept of warm-up stages and low-reflectivity bias was introduced to address the specific challenges of indirect depth optimization and dynamic scene reconstruction [S1]. The APMGSRN initialization variant was developed for large-scale neural field visualization [S2]. The approach builds on the foundational 3D Gaussian Splatting method [S1, S2].

## Sources
[S1] Time of the Flight of the Gaussians: Optimizing Depth Indirectly in Dynamic Radiance Fields, https://doi.org/10.1109/cvpr52734.2025.01958, fetched 2026-10-26
[S2] Adaptively Placed Multi-Grid Scene Representation Networks for Large-Scale Data Visualization, https://doi.org/10.1109/tvcg.2023.3327194, fetched 2026-10-26
