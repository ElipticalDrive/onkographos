# Gaussian Initialization and Warm-up

Also called: Data/Random Initialization for Gaussian Splatting, Initial Approximate Posteriors Initialization

## Overview
Gaussian Initialization and Warm-up establishes the initial state of a set of $N$ 3D Gaussians with parameters $\mu_i, \Sigma_i, \alpha_i, C_i$ to facilitate stable optimization in differentiable volume rendering [G]. It solves the problem of unstable convergence and incorrect primitive placement caused by poor initial guesses in high-dimensional non-convex optimization landscapes [S1][S2, unofficial]. It sets itself apart from its siblings, such as [Dense Initialization (DI)](10026-dense-initialization-di.md) which generates millions of primitives upfront, and [Structure-Aware Initialization](13704-structure-aware-initialization.md) which adapts placement to local complexity, by focusing on the statistical and heuristic initialization of a sparse set followed by a static warm-up phase [Gaussian Initialization and Warm-up](9657-gaussian-initialization-and-warm-up.md).

## How it works
**Inputs:**
- `num_gaussians`: Total number of Gaussians $N$.
- `near_plane`, `far_plane`: Camera frustum depth bounds.
- `voxel_data` (optional): Voxel grid containing spatial hashes and averaged attributes.
- `use_deformation_mlp` (optional): Boolean flag indicating if coordinates require rescaling for MLP input.
- `unambiguous_depth_range` (optional): Tuple $(d_{min}, d_{max})$ for coordinate rescaling.
- `initialization_mode`: Enum or string specifying 'Random', 'Data', or 'Voxel'.

**Outputs:**
- A list of $N$ Gaussian objects, each containing fields: `mu` (mean, 3D vector), `sigma` (covariance, 3x3 matrix), `alpha` (opacity, scalar), `C_dc` (DC color, 3D vector), `C_ac` (high-frequency SH coefficients, 12D vector).

**Data Structures:**
- `Gaussian`: A struct or dictionary with fields:
  - `mu`: `float[3]`
  - `sigma`: `float[3][3]`
  - `alpha`: `float`
  - `C_dc`: `float[3]`
  - `C_ac`: `float[12]`

**Parameters:**
- `alpha_init_range`: Low opacity range $[0.01, 0.1]$ to prevent inverse-square falloff interference [S1, G].
- `warmup_iterations`: Number of static optimization iterations, typically $2,000$ [Gaussian Initialization and Warm-up](9657-gaussian-initialization-and-warm-up.md).
- `background_noise_range`: Uniform range $[-1, 1]$ for random background signal perturbation [S1].

**Steps:**

1. **Spatial Initialization**:
   - If `voxel_data` is provided:
     - Sample $N$ points uniformly within the camera frustum.
     - For each point, query the voxel hash to find the containing voxel.
     - Set `mu` to the voxel's center or the sampled point within the voxel.
     - Set `sigma` to the largest volume covariance found in the voxel [S4].
     - Set `C_dc` and `C_ac` to the average color attributes of the voxel [S4].
   - If `voxel_data` is not provided:
     - Sample depth $d \sim U(\text{near\_plane}, \text{far\_plane})$.
     - Sample NDC coordinates $x, y \sim U(-1.0, 1.0)$.
     - Compute mean $\mu = [x \cdot d, y \cdot d, d]$.
     - If `use_deformation_mlp` and `unambiguous_depth_range` are set:
       - Rescale $\mu$: $\mu' = \frac{\mu - d_{min}}{d_{max} - d_{min}}$ [S1].
     - Initialize `sigma` to Identity matrix $I$ [G].
     - Initialize `C_dc` to $[1.0, 1.0, 1.0]$ [G].
     - Initialize `C_ac` to zeros [G].

2. **Attribute Initialization**:
   - Set opacity $\alpha \sim U(0.01, 0.1)$ [S1, G].
   - If `initialization_mode` is 'Data' (Variational Bayes context):
     - Sample means $m_{s,k,init}$ and $m_{c,k,init}$ from data $D$ [S2, unofficial].
     - Set initial approximate posterior parameters according to Table 3 in [S2, unofficial]:
       - For Spatial $q(\mu_{k,s}, \Sigma_{k,s})$:
         - 2D: $m_{s,k}=10^{-5}\cdot 1$, $\kappa_{s,k}=2.25\cdot 10^4 \cdot n_c \cdot I$, $V_{s,k}=4$, $n_{s,k}=m_{s,k,init}$.
         - 3D: $m_{s,k}=10^{-6}\cdot 1$, $\kappa_{s,k}=2.25\cdot 10^6 \cdot n_c \cdot I$, $V_{s,k}=5$, $n_{s,k}=m_{s,k,init}$.
       - For Color $q(\mu_{k,c}, \Sigma_{k,c})$:
         - 2D: $m_{c,k}=10^{-2}\cdot 1$, $\kappa_{c,k}=10^6 \cdot I$, $n_{c,k}=5$, $m_{c,k,init}=m_{c,k,init}$.
         - 3D: $m_{c,k}=10^{-2}\cdot 1$, $\kappa_{c,k}=10^8 \cdot I$, $n_{c,k}=5$, $m_{c,k,init}=m_{c,k,init}$.
       - For Dirichlet $q(\pi)$: $\alpha_k=1$, $n_c=1/n_c$ [S2, unofficial].
   - If `initialization_mode` is 'Random' (Variational Bayes context):
     - $m_{k,s,init} \sim U[-1, 1]$, $m_{k,c,init} \sim \delta(0)$ [S2, unofficial].

3. **Low-Reflectivity Bias Handling** (Optional, for C-ToF or sensitive scenes):
   - Initialize reflectivity (opacity) to low values ($0.01$–$0.1$) to prevent Gaussians in low-reflectivity regions from being incorrectly placed far away due to inverse-square falloff [S1].
   - During optimization, randomly perturb the background signal uniformly between $-1$ and $1$ at each iteration to prevent Gaussians from being placed on a surface unnecessarily when the background already reproduces the signal [S1].

4. **Warm-up Phase**:
   - Run optimization for $2,000$ iterations assuming the scene is static [Gaussian Initialization and Warm-up](9657-gaussian-initialization-and-warm-up.md).
   - This places Gaussians into useful positions before enabling dynamic updates or complex deformations [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def initialize_gaussians(num_gaussians, near_plane, far_plane, voxel_data=None, 
                         use_deformation_mlp=False, unambiguous_depth_range=None, 
                         initialization_mode='Random'):
    """
    Initialize a set of 3D Gaussians.
    
    Args:
        num_gaussians (int): Number of Gaussians to initialize.
        near_plane (float): Near plane of the camera frustum.
        far_plane (float): Far plane of the camera frustum.
        voxel_data (object, optional): Voxel grid with get_spatial_hash and averaged attributes.
        use_deformation_mlp (bool): Whether to rescale coordinates for MLP input.
        unambiguous_depth_range (tuple): (d_min, d_max) for rescaling.
        initialization_mode (str): 'Random', 'Data', or 'Voxel'.
        
    Returns:
        list: List of Gaussian dictionaries.
    """
    gaussians = []
    
    for _ in range(num_gaussians):
        gaussian = {}
        
        # 1. Spatial Initialization
        if voxel_data is not None:
            # Sample within frustum to query voxel
            depth = np.random.uniform(near_plane, far_plane)
            ndc_x, ndc_y = np.random.uniform(-1.0, 1.0), np.random.uniform(-1.0, 1.0)
            mu = np.array([ndc_x * depth, ndc_y * depth, depth])
            
            # Rescale if needed for MLP
            if use_deformation_mlp and unambiguous_depth_range is not None:
                scale = unambiguous_depth_range[1] - unambiguous_depth_range[0]
                mu = (mu - unambiguous_depth_range[0]) / scale
                
            # Get voxel attributes
            voxel = voxel_data.get_spatial_hash(mu)
            sigma = voxel.largest_volume_covariance
            C_dc = voxel.average_color_attributes['dc']
            C_ac = voxel.average_color_attributes['ac']
        else:
            # Random initialization within frustum
            depth = np.random.uniform(near_plane, far_plane)
            ndc_x, ndc_y = np.random.uniform(-1.0, 1.0), np.random.uniform(-1.0, 1.0)
            mu = np.array([ndc_x * depth, ndc_y * depth, depth])
            
            if use_deformation_mlp and unambiguous_depth_range is not None:
                scale = unambiguous_depth_range[1] - unambiguous_depth_range[0]
                mu = (mu - unambiguous_depth_range[0]) / scale
                
            sigma = np.eye(3)
            C_dc = np.array([1.0, 1.0, 1.0])
            C_ac = np.zeros(12)
            
        # 2. Attribute Initialization
        alpha = np.random.uniform(0.01, 0.1)
        
        if initialization_mode == 'Data':
            # Data-driven initialization (Variational Bayes)
            # Sample from data D (not implemented here as D is external)
            # Set posterior parameters as per Table 3 in [S2, unofficial]
            pass 
        elif initialization_mode == 'Random':
            # Random initialization (Variational Bayes)
            # m_s,init ~ U[-1, 1], m_c,init ~ delta(0)
            pass
            
        gaussian['mu'] = mu
        gaussian['sigma'] = sigma
        gaussian['alpha'] = alpha
        gaussian['C_dc'] = C_dc
        gaussian['C_ac'] = C_ac
        gaussians.append(gaussian)
        
    return gaussians

def warm_up_static(gaussians, iterations=2000, background_noise_range=(-1.0, 1.0)):
    """
    Run static warm-up optimization.
    
    Args:
        gaussians (list): List of Gaussian dictionaries.
        iterations (int): Number of warm-up iterations.
        background_noise_range (tuple): Range for random background perturbation.
    """
    for i in range(iterations):
        # Perturb background signal for sensitive low-reflectivity areas [S1]
        bg_noise = np.random.uniform(background_noise_range[0], background_noise_range[1])
        
        # Perform optimization step (e.g., SGD) on Gaussians
        # This step is implementation-specific and depends on the renderer
        # For the purpose of initialization, we assume this loop runs
        # to place Gaussians in useful positions [Gaussian Initialization and Warm-up](9657-gaussian-initialization-and-warm-up.md)
        optimize_gaussians(gaussians, bg_noise)

def optimize_gaussians(gaussians, bg_noise):
    """
    Placeholder for optimization step.
    In a real implementation, this would compute gradients and update parameters.
    """
    pass
```

## Subtopics

### [Dense Initialization (DI)](10026-dense-initialization-di.md)
Dense Initialization (DI) generates millions of small Gaussians to densely cover all visible surfaces at the outset, rather than starting with a sparse set and progressively refining it. It solves the problem of slow convergence and the computational overhead of iterative densification stages used in standard 3D Gaussian Splatting. DI sets itself apart from its nearest alternative, Surface from Motion initialization with iterative densification, by eliminating the need for densification entirely through high-density matching, which also benefits ray tracing performance by reducing the workload of irrelevant primitives.

### [Recurrent Gaussian Update](10974-recurrent-gaussian-update.md)
Recurrent Gaussian Update is a gradient-free, feed-forward refinement method that iteratively corrects initial 3D Gaussian predictions by predicting incremental updates to Gaussian parameters and hidden states based on rendering errors from input views. It solves the problem of inaccurate initial reconstructions in single-step feed-forward models by establishing a feedback loop that progressively refines the Gaussian set without requiring backpropagation or gradient computation. Unlike standard differentiable optimization which relies on iterative gradient descent, this method uses a learned recurrent network to map rendering errors directly to parameter updates, enabling convergence in a fixed number of iterations (typically 4) rather than thousands.

### [Structure-Aware Initialization](13704-structure-aware-initialization.md)
Structure-Aware Initialization is a strategy that adapts primitive placement and scale to local image complexity using local variance. It solves the problem of inefficient coverage in radiance-field optimization by ensuring dense, fine-grained primitives in complex regions and sparse, coarse coverage in smooth areas. This method accelerates convergence and improves final fidelity, particularly when using high primitive counts.

### [Cluster-level Weighting](17081-cluster-level-weighting.md)
Cluster-level Weighting is a method for adaptively determining the contribution of multi-scale Gaussians in a Pyramidal Gaussian hierarchy during rendering. It solves the problem of balancing detail and performance by assigning unique weight sets to spatial clusters of Gaussians rather than individual splats, thereby incorporating positional context without the computational cost of per-Gaussian weight networks. It differs from sibling methods like Dense Initialization or Structure-Aware Initialization by focusing on the *rendering-time* adaptive weighting of an existing hierarchical structure, rather than the *initialization* of the Gaussian parameters themselves.

### Low-Reflectivity Bias
Low-Reflectivity Bias is a heuristic initialization strategy for scenes with varying reflectivity, particularly in C-ToF or dynamic radiance fields [S1]. It initializes Gaussian reflectivity (opacity) to a low value ($0.01$–$0.1$) to prevent Gaussians in low-reflectivity regions from being incorrectly placed far away due to inverse-square falloff, which leads to multiple opacity peaks and interference from closer splats [S1]. Low initial reflectivity allows Gaussians in low-reflectivity regions to be placed more correctly (closer fit to true scene), while Gaussians in high-reflectivity regions are placed closer to the camera, occluding the scene and preventing interference [S1]. Combining occupancy and low-reflectivity biases is effective for both low and high reflectivity regions, whereas using only occupancy bias harms low-reflectivity regions [S1].

### APMGSRN Initialization
APMGSRN Initialization is a specific initialization scheme for Adaptively Placed Multi-Grid Scene Representation Networks [S3]. It initializes transformation matrices to cover a near global domain with small random shears, rotations, and translations. Diagonals are sampled from $N(1, 0.05)$, and remaining entries in the first three rows are sampled from $N(0, 0.05)$ [S3]. Feature grids are initialized from $U(-0.0001, 0.0001)$, and decoder network weights follow Glorot and Bengio initialization [S3]. This ensures each entry in the transformation matrix is non-zero, guaranteeing gradients can be used to update each entry [S3].

### Fine-tuning/Recoloring with Initialization
Fine-tuning/Recoloring with Initialization is a constrained retraining process that fixes Gaussian means after voxelization and fine-tunes the remaining attributes (covariance $\Sigma$, SH coefficients $C$, opacity $\alpha$) to minimize training loss [S4]. It initializes DC colors $C_i^{(dc)}$, high-frequency SH coefficients $C_i^{(ac)}$, and opacities $\alpha_i$ of the voxelized Gaussian means by averaging the attributes within each voxel [S4]. The covariance $\Sigma$ is initialized by the largest volume Gaussian in the voxel [S4]. This initialization requires approximately $8,000$ iterations to approach ground-truth PSNR, whereas fine-tuning without initialization requires $30,000$ iterations and fails to recover ground-truth PSNR at higher rates [S4]. Initializing attributes to zero takes more iterations and often results in sub-optimal rendering quality because Gaussian splitting and merging are disabled during retraining [S4].

### Appearance-aware Rendering with Color Correction
Appearance-aware Rendering with Color Correction is a mechanism to handle lighting variations across images by applying a per-cluster color correction $\Delta c_i$ based on camera appearance embedding and cluster embedding, before final alpha blending [S5, unofficial]. It computes color correction $\Delta c_i = F_\phi (\text{Concat}(E_c[k_i], E_{app}[n]))$ for the $i$-th Gaussian belonging to cluster $k_i$ viewed from camera $n$ [S5, unofficial]. The final color is rendered by sorting Gaussians by depth and blending adjusted colors $(c_i + \Delta c_i)$ weighted by cluster-level weight $w_{k_i,n}[l_i]$ and opacity $\alpha_i$ [S5, unofficial]. The rendering equation is:
$$
\hat{C} = \sum_{i=1}^{N} \frac{(c_i + \Delta c_i) \cdot w_{k_i,n}[l_i] \cdot \alpha_i}{\prod_{j=1}^{i-1} (1 - w_{k_j,n}[l_j] \cdot \alpha_j)}
$$
where $c_i$ is spherical harmonic color, $\Delta c_i$ is color correction, $w$ is cluster-level weight, and $\alpha$ is opacity [S5, unofficial]. The Appearance Embedding $E_{app}$ is a learnable tensor of size $R^{N_{cam} \times D}$, where $N_{cam}$ is the total count of cameras [S5, unofficial].

### Geometric Subsampling for Level Initialization
Geometric Subsampling for Level Initialization reduces the number of Gaussians at higher pyramid levels by dividing the input point count by powers of two during initialization, enabling coarser representations at upper levels [S6, unofficial]. It uses a boolean flag `subsample_levels` (default: true) to enable this behavior [S6, unofficial]. A pitfall is that subsampling assumes the input point cloud is spatially randomized; non-randomized data may cause clustering or gaps at subsampled levels [S6, unofficial].

## Operating characteristics
- **Time Complexity**: Initialization is $O(N)$ for random sampling. Voxel-based initialization is $O(N \log V)$ or $O(N)$ with hash maps, where $V$ is the number of voxels. Warm-up adds $O(I \cdot N)$ where $I$ is the number of warm-up iterations.
- **Memory Complexity**: $O(N)$ to store Gaussian parameters. Voxel-based initialization requires additional memory for the voxel grid.
- **Measured Performance**: Fine-tuning with voxel-based initialization requires ~8K iterations to approach ground-truth PSNR, compared to 30K iterations without initialization [S4].
- **Scaling**: Scales linearly with the number of Gaussians. Voxel-based initialization benefits from spatial coherence but requires efficient voxel hashing.
- **Strengths**: Prevents inverse-square falloff interference, stabilizes optimization in low-reflectivity regions, and accelerates convergence through informed initialization.
- **Limitations**: Random initialization may lead to sub-optimal local minima without warm-up. Voxel-based initialization depends on the quality of the voxel grid.
- **When to use**: Use when starting with a sparse set of Gaussians and needing stable convergence. Use voxel-based initialization when compressing or reconstructing from point clouds. Use low-reflectivity bias for C-ToF or scenes with high dynamic range reflectivity.
- **When not to use**: Avoid random initialization without warm-up for complex scenes. Avoid voxel-based initialization if memory is constrained and the scene is sparse.

## Background
Gaussian Initialization and Warm-up originated with the introduction of 3D Gaussian Splatting [Gaussian Initialization and Warm-up](9657-gaussian-initialization-and-warm-up.md). The method was developed to address the instability of optimizing Gaussian parameters from random initialization. The low-reflectivity bias was introduced in [S1] to handle C-ToF data. Variational Bayes Gaussian Splatting [S2, unofficial] introduced data-driven and random initialization schemes for approximate posteriors. APMGSRN [S3] introduced specific initialization for multi-grid scene representations. Adaptive Voxelization [S4] proposed voxel-based initialization for compression. PyGS [S5, unofficial] introduced appearance-aware rendering with color correction. Geometric Subsampling [S6, unofficial] was introduced for pyramidal Gaussian hierarchies.

## Sources
[S1] Time of the Flight of the Gaussians: Optimizing Depth Indirectly in Dynamic Radiance Fields, https://doi.org/10.1109/cvpr52734.2025.01958, fetched 2026-10-26
[S2] Variational Bayes Gaussian Splatting, https://arxiv.org/abs/2410.03592v2, fetched 2026-10-26 (unofficial)
[S3] Adaptively Placed Multi-Grid Scene Representation Networks for Large-Scale Data Visualization, https://doi.org/10.1109/tvcg.2023.3327194, fetched 2026-10-26
[S4] Adaptive Voxelization for Transform Coding of 3D Gaussian Splatting Data, https://doi.org/10.1109/icip55913.2025.11084522, fetched 2026-10-26
[S5] PyGS: Large-scale Scene Representation with Pyramidal 3D Gaussian Splatting, https://doi.org/10.48550/arxiv.2405.16829, fetched 2026-10-26 (unofficial)
[S6] DBauer15/gscache, https://github.com/DBauer15/gscache, fetched 2026-10-26 (unofficial)
