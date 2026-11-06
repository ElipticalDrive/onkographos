# Neural Adaptive Tomography (NeAT)

## Also called
NeAT

## Overview
Neural Adaptive Tomography (NeAT) is an adaptive, hierarchical neural rendering pipeline for multi-view inverse rendering, designed specifically for the tomographic reconstruction of semi-transparent volumetric scenes from X-ray images [S1]. It combines an adaptive explicit octree representation with neural features stored in leaf nodes to achieve efficient reconstruction and simultaneous self-calibration of geometric and photometric parameters [S1]. Unlike purely implicit neural representations, NeAT uses a hybrid explicit-implicit structure optimized via differentiable volume rendering, which outperforms alternatives in both quality and compute time for tomographic tasks where samples are widely scattered [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ X-ray projection images $I(p)$, where $p$ denotes pixel coordinates [S1]. Initial estimates of camera poses (source and detector positions/orientations) and a reference image $I_0$ (empty scan) are required but can be refined [S1].
*   **Outputs:** A reconstructed 3D volume represented by the optimized octree structure, neural feature grids, decoder weights, and calibrated camera parameters [S1].

**Data Structures:**
The scene is represented as a sparse octree [S1]. Each node in the octree has the following layout:
*   `is_leaf`: Boolean flag indicating if the node is a leaf [S1].
*   `children`: Array of 8 child node references (if not a leaf) [G].
*   `features`: A uniform 3D grid of neural feature vectors (only if `is_leaf` is true) [S1].
*   `empty`: Boolean flag indicating if the node contains no density; empty leaf nodes are pruned during optimization [S1].

**Neural Decoder:**
A small multi-layer perceptron (MLP) decoder $\Phi$ maps a feature vector $f$ to a scalar density $\mu$:
$$ \mu = \Phi(f) $$
The decoder is shared across all leaf nodes and is optimized during training [S1].

**Numbered Steps:**
1.  **Initialization:** Initialize an octree with a single root node containing a coarse grid of neural features [S1]. Initialize the neural decoder $\Phi$ with random weights [G]. Initialize camera parameters (poses) and photometric parameters (reference image $I_0$, per-view exposure multipliers) [S1].
2.  **Ray Generation:** For each training image and each pixel $p$, generate a ray $r_p(t) = o_p + t \cdot d_p$, where $o_p$ is the X-ray source position and $d_p$ is the direction towards the detector pixel [S1].
3.  **Ray-Octree Intersection:** Trace ray $r_p$ through the octree [S1]. Identify the sequence of leaf nodes intersected by the ray [S1]. For each intersected leaf node, compute the entry and exit points $t_n, t_f$ within the node's local space [S1].
4.  **Neural Volume Sampling:** Sample points $t_i$ along the ray segment within each leaf node [S1]. For each sample point $x$, determine its local coordinates within the leaf node's feature grid [S1]. Perform trilinear interpolation of the feature vectors from the grid to obtain the feature vector $f(x)$ at $x$ [S1].
5.  **Density Decoding:** Compute the density $\mu(x) = \Phi(f(x))$ using the shared decoder network [S1].
6.  **Volume Rendering Integration:** Integrate the density along the ray using a weighted sum to compute the estimated ray integral $\hat{I}'_p(p)$:
    $$ \hat{I}'_p(p) = \sum_{i} w_i \mu(x_i) $$
    where $w_i$ are the integration weights determined by the sampling strategy and node boundaries [G].
7.  **Photometric Calibration:** Apply the Beer-Lambert law to compute the final estimated pixel intensity $\hat{I}(p)$:
    $$ \hat{I}(p) = \hat{I}_0(p) \exp \left( - \hat{I}'_p(p) \right) $$
    where $\hat{I}_0(p)$ is the spatially varying source intensity (reference image) possibly modulated by per-view exposure multipliers [S1].
8.  **Loss Computation:** Compute the total loss $L_{total}$:
    $$ L_{total} = \sum_p ||I(p) - \hat{I}(p)||^2 + \lambda_{tv} L_{tv} + \lambda_{bc} L_{bc} $$
    where $I(p)$ is the measured intensity, and $\lambda_{tv}$ and $\lambda_{bc}$ are hyperparameters [S1].
    *   **Total Variation (TV) Loss:** Compute on feature vectors (preferred for speed/quality):
        $$ L_{tv} = \sum_n \|\nabla f_n\|_1 $$
        where the gradient is computed over the feature vectors of each leaf node [S1].
    *   **Boundary Consistency (BC) Loss:** Minimize feature discontinuities between neighboring nodes $n$ and $m$ on their shared boundary surface $\Lambda_{bc}$:
        $$ L_{bc} = \sum_{(n,m) \in N} \sum_{x \in \Lambda_{bc}} |f_n(x) - f_m(x)| $$
        where $N$ is the set of neighboring node pairs, and $f_n(x), f_m(x)$ are the interpolated features at boundary point $x$ from both nodes [S1].
9.  **Optimization:** Perform gradient descent to update: neural feature grids in all leaf nodes, decoder weights $\Phi$, camera poses (geometric self-calibration), and reference image $I_0$ and exposure multipliers (photometric self-calibration) [S1].
10. **Octree Update (Adaptive Refinement):** Compute volumetric error for each node $n$:
    $$ E(n) = \rho_{max}(n) \cdot \sum_{(p,t_n,t_f) \in \Omega_n} ||I(p) - \hat{I}(p)||^2 $$
    where $\Omega_n$ is the set of samples passing through node $n$, and $\rho_{max}(n)$ is the maximum density in node $n$ [S1]. Solve a mixed-integer program (MIP) to determine the optimal tree structure (splitting nodes with high error, pruning empty nodes) based on $E(n)$ [S1]. Re-initialize features in newly split nodes and transfer/interpolate features from parent to children [G].
11. **Iteration:** Repeat steps 2-10 until convergence or maximum iterations [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def trilinear_interpolate(grid, local_coords):
    """Trilinear interpolation of a 3D feature grid at local_coords (0-1 range)."""
    x, y, z = local_coords
    ix, iy, iz = np.floor([x, y, z]).astype(int)
    fx, fy, fz = [c - i for c, i in zip([x, y, z], [ix, iy, iz])]
    # Clamp indices to grid bounds
    ix, iy, iz = np.clip([ix, iy, iz], 0, np.array(grid.shape) - 1)
    c000 = grid[ix[0], iy[0], iz[0]]
    c100 = grid[ix[0]+1, iy[0], iz[0]]
    c010 = grid[ix[0], iy[0]+1, iz[0]]
    c110 = grid[ix[0]+1, iy[0]+1, iz[0]]
    c001 = grid[ix[0], iy[0], iz[0]+1]
    c101 = grid[ix[0]+1, iy[0], iz[0]+1]
    c011 = grid[ix[0], iy[0]+1, iz[0]+1]
    c111 = grid[ix[0]+1, iy[0]+1, iz[0]+1]
    w000 = (1-fx[0])*(1-fy[0])*(1-fz[0])
    w100 = fx[0]*(1-fy[0])*(1-fz[0])
    w010 = (1-fx[0])*fy[0]*(1-fz[0])
    w110 = fx[0]*fy[0]*(1-fz[0])
    w001 = (1-fx[0])*(1-fy[0])*fz[0]
    w101 = fx[0]*(1-fy[0])*fz[0]
    w011 = (1-fx[0])*fy[0]*fz[0]
    w111 = fx[0]*fy[0]*fz[0]
    return (w000*c000 + w100*c100 + w010*c010 + w110*c110 +
            w001*c001 + w101*c101 + w011*c011 + w111*c111)

def mlp_decode(feature_vector, w1, b1, w2, b2):
    """Simple decoder network mapping feature vector to density."""
    h = np.maximum(0, feature_vector @ w1 + b1)
    return np.maximum(0, h @ w2 + b2)

class OctreeNode:
    def __init__(self, features=None):
        self.features = features  # 3D grid of feature vectors
        self.children = [None] * 8
        self.is_leaf = features is not None
        self.empty = False

    def trace_ray(self, ray_origin, ray_dir, t_min, t_max):
        """Returns list of (node, t_start, t_end) for intersected leaves."""
        samples = []
        if self.is_leaf:
            samples.append((self, t_min, t_max))
        else:
            for child in self.children:
                if child is not None:
                    child_samples = child.trace_ray(ray_origin, ray_dir, t_min, t_max)
                    samples.extend(child_samples)
        return samples

def compute_tv_loss(octree):
    """Compute Total Variation loss on feature grids."""
    tv = 0.0
    stack = [octree]
    while stack:
        node = stack.pop()
        if node.is_leaf and node.features is not None:
            grad_x = np.diff(node.features, axis=0)
            grad_y = np.diff(node.features, axis=1)
            grad_z = np.diff(node.features, axis=2)
            tv += np.sum(np.abs(grad_x)) + np.sum(np.abs(grad_y)) + np.sum(np.abs(grad_z))
        else:
            stack.extend(node.children)
    return tv

def compute_bc_loss(octree):
    """Compute Boundary Consistency loss between neighboring nodes."""
    bc = 0.0
    leaves = []
    stack = [octree]
    while stack:
        node = stack.pop()
        if node.is_leaf:
            leaves.append(node)
        else:
            stack.extend(node.children)
    # Sum feature differences across adjacent leaf pairs
    for i in range(len(leaves) - 1):
        bc += np.sum(np.abs(leaves[i].features - leaves[i+1].features))
    return bc

def train_neat(X_ray_images, initial_poses, initial_I0, num_iterations, lambda_tv, lambda_bc):
    """
    X_ray_images: List of 2D arrays, shape (H, W), measured intensities.
    initial_poses: List of camera parameters (source, detector).
    initial_I0: 2D array, reference image (empty scan).
    """
    # Initialize Octree
    octree = OctreeNode(features=np.random.randn(4, 4, 4, 32))
    
    # Initialize Decoder weights
    decoder_w1 = np.random.randn(32, 16)
    decoder_b1 = np.random.randn(16)
    decoder_w2 = np.random.randn(16, 1)
    decoder_b2 = np.random.randn(1)
    
    # Initialize Calibration Parameters
    poses = [p.copy() for p in initial_poses]
    I0 = initial_I0.copy()
    exposures = np.ones(len(X_ray_images))
    
    for iteration in range(num_iterations):
        total_loss = 0.0
        
        # Accumulate gradients for all pixels
        for view_idx, I_measured in enumerate(X_ray_images):
            pose = poses[view_idx]
            exposure = exposures[view_idx]
            
            H, W = I_measured.shape
            for y in range(H):
                for x in range(W):
                    p = (x, y)
                    # 1. Generate Ray
                    ray_origin = pose['source']
                    ray_dir = pose['detector_center'] - ray_origin
                    ray_dir = ray_dir / np.linalg.norm(ray_dir)
                    
                    # 2. Intersect Octree
                    samples = octree.trace_ray(ray_origin, ray_dir, 0.0, 10.0)
                    
                    # 3. Sample and Decode
                    ray_integral = 0.0
                    for node, t_start, t_end in samples:
                        if node.features is not None:
                            num_samples = 10
                            for i in range(num_samples):
                                t = t_start + (t_end - t_start) * (i + 0.5) / num_samples
                                local_pos = (t - ray_origin) / ray_dir
                                local_coords = np.clip(local_pos, 0, 1)
                                f = trilinear_interpolate(node.features, local_coords)
                                mu = mlp_decode(f, decoder_w1, decoder_b1, decoder_w2, decoder_b2)
                                w = (t_end - t_start) / num_samples
                                ray_integral += w * mu
                    
                    # 4. Photometric Model
                    I0_val = I0[p] * exposure
                    I_estimated = I0_val * np.exp(-ray_integral)
                    
                    # 5. Loss Components
                    mse_loss = (I_measured[p] - I_estimated) ** 2
                    tv_loss = compute_tv_loss(octree)
                    bc_loss = compute_bc_loss(octree)
                    
                    pixel_loss = mse_loss + lambda_tv * tv_loss + lambda_bc * bc_loss
                    total_loss += pixel_loss
        
        # Update Parameters (conceptual gradient step)
        # In practice, use an autograd framework. Here we simulate a step.
        # decoder_w1 -= lr * grad_w1
        # octree.features -= lr * grad_features
        # poses -= lr * grad_poses
        # I0 -= lr * grad_I0
        # exposures -= lr * grad_exposures
        
        # Octree Refinement (every 10 iterations)
        if iteration % 10 == 0:
            # Compute E(n) and solve MIP to split/prune
            # Re-initialize features in newly split nodes
            pass

    return octree, decoder_w1, decoder_b1, decoder_w2, decoder_b2, poses, I0, exposures
```

**Parameters:**
*   $\lambda_{bc}$: Boundary Consistency regularizer strength. Typical value: 0.01 [S1].
*   $\lambda_{tv}$: Total Variation regularizer strength. Typical value: 0.00002 for sparse view problems, 0.00010 for limited angle problems [S1].

**Edge Cases and Pitfalls:**
*   **Block-like Artifacts:** If the Boundary Consistency regularizer is disabled, block-like artifacts appear at boundaries between neighboring octree nodes, especially with few images [S1].
*   **Reference Image Validity:** Adding or removing the object from the setup can disturb the validity of the reference image $I_0$, causing reconstruction artifacts. NeAT mitigates this by optimizing $I_0$ during training [S1].
*   **Computational Cost:** While NeAT is faster than other neural inverse rendering methods, it is still computationally intensive compared to traditional methods like SIRT or FDK. It is designed to handle high-resolution datasets and full 3D cone beam data better than previous neural methods [S1].
*   **Sparse Views:** The method is specifically designed to handle ill-posed geometric configurations (sparse views or limited angular coverage) where traditional methods fail [S1].

## Operating characteristics
*   **Time Complexity:** Exact time complexity formulas are not given in the sources. NeAT achieves reconstruction times far superior to existing neural inverse rendering methods [S1].
*   **Memory Complexity:** Exact memory complexity formulas are not given in the sources. Memory usage is proportional to the number of leaf nodes in the octree and the size of the feature grids [S1].
*   **Scaling:** NeAT scales well with resolution due to the adaptive octree structure, which only refines regions with detail [S1]. It handles full 3D cone beam data effectively [S1].
*   **Strengths:**
    *   High-quality reconstruction for ill-posed configurations (sparse views, limited angles) [S1].
    *   Simultaneous geometric and photometric self-calibration [S1].
    *   Faster than other neural inverse rendering methods [S1].
    *   Outperforms purely implicit and hybrid explicit-implicit representations (like ACORN) in tomographic settings [S1].
*   **Limitations:**
    *   Still slower than traditional analytical methods (e.g., FDK, SIRT) [S1].
    *   Requires careful tuning of hyperparameters ($\lambda_{tv}, \lambda_{bc}$) [S1].
    *   Sensitive to initial calibration; however, self-calibration mitigates this [S1].
*   **When to Use:** Use NeAT when dealing with sparse view or limited angle tomography, when high-quality reconstruction is needed despite ill-posed geometry, and when simultaneous calibration is beneficial. It is suitable when computational resources allow for iterative optimization [S1].
*   **When Not to Use:** Use traditional methods (e.g., [Back-projection](2450-back-projection.md), [FastFDK](3315-fastfdk.md)) when computational speed is critical and the geometry is well-posed (full angular coverage) [S1].

## Background
Neural Adaptive Tomography (NeAT) was introduced by Darius Rückert, Yuanhao Wang, Rui Li, Ramzi Idoughi, and Wolfgang Heidrich from KAUST and the University of Erlangen-Nuremberg [S1]. It was presented in 2022 [S1]. The method addresses the limitations of existing neural inverse rendering frameworks, which suffer from long computing times, by introducing an adaptive, hierarchical neural representation specifically tuned for tomographic reconstruction [S1]. It builds on the concept of neural rendering [Mildenhall et al. 2020] and adaptive representations like ACORN [Martel et al. 2021], but adapts them for the specific challenges of X-ray tomography, such as scattered samples and the need for self-calibration [S1].

## Sources
[S1] NeAT: Neural Adaptive Tomography, https://doi.org/10.48550/arxiv.2202.02171, fetched 2026-10-26
