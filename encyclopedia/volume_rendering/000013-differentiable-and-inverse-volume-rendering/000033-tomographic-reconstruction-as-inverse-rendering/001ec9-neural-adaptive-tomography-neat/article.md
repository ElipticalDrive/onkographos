# Neural Adaptive Tomography (NeAT)

## Overview
Neural Adaptive Tomography (NeAT) is an adaptive, hierarchical neural rendering pipeline for multi-view inverse rendering, designed specifically for the tomographic reconstruction of semi-transparent volumetric scenes from X-ray images [S1]. It combines an adaptive explicit octree representation with neural features stored in leaf nodes to achieve efficient reconstruction and simultaneous self-calibration of geometric and photometric parameters [S1]. Unlike purely implicit neural representations or hybrid methods like ACORN, NeAT uses a purely explicit hierarchical structure optimized via differentiable volume rendering, which outperforms alternatives in both quality and compute time for tomographic tasks where samples are widely scattered [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ X-ray projection images $I(p)$, where $p$ denotes pixel coordinates. Initial estimates of camera poses (source and detector positions/orientations) and a reference image $I_0$ (empty scan) are required but can be refined [S1].
*   **Outputs:** A reconstructed 3D volume represented by the optimized octree structure, neural feature grids, decoder weights, and calibrated camera parameters [S1].

**Data Structures:**
The scene is represented as a sparse octree. Each node in the octree has the following layout:
*   `is_leaf`: Boolean flag.
*   `children`: Array of 8 child nodes (if not a leaf).
*   `features`: A uniform 3D grid of neural feature vectors (only if `is_leaf` is true). The grid dimensions are fixed per node level (e.g., $k \times k \times k$).
*   `empty`: Boolean flag indicating if the node contains no density; empty leaf nodes are pruned [S1].

**Neural Decoder:**
A small multi-layer perceptron (MLP) decoder $\Phi$ maps a feature vector $f$ to a scalar density $\mu$:
$$ \mu = \Phi(f) $$
The decoder is shared across all leaf nodes and is optimized during training [S1].

**Numbered Steps:**

1.  **Initialization:**
    *   Initialize an octree with a single root node containing a coarse grid of random neural features.
    *   Initialize the neural decoder $\Phi$ with random weights.
    *   Initialize camera parameters (poses) and photometric parameters (reference image $I_0$, per-view exposure multipliers) [S1].

2.  **Ray Generation:**
    *   For each training image and each pixel $p$, generate a ray $r_p(t) = o_p + t \cdot d_p$, where $o_p$ is the X-ray source position and $d_p$ is the direction towards the detector pixel [S1].

3.  **Ray-Octree Intersection:**
    *   Trace ray $r_p$ through the octree.
    *   Identify the sequence of leaf nodes intersected by the ray.
    *   For each intersected leaf node, compute the entry and exit points $t_n, t_f$ within the node's local space [S1].

4.  **Neural Volume Sampling:**
    *   Sample points $t_i$ along the ray segment within each leaf node.
    *   For each sample point $x$, determine its local coordinates within the leaf node's feature grid.
    *   Perform trilinear interpolation of the feature vectors from the grid to obtain the feature vector $f(x)$ at $x$ [S1].

5.  **Density Decoding:**
    *   Compute the density $\mu(x) = \Phi(f(x))$ using the shared decoder network [S1].

6.  **Volume Rendering Integration:**
    *   Integrate the density along the ray using a weighted sum (quadrature) to compute the estimated ray integral $\hat{I}'_p(p)$:
        $$ \hat{I}'_p(p) = \sum_{i} w_i \mu(x_i) $$
        where $w_i$ are the quadrature weights determined by the sampling strategy and node boundaries [S1].

7.  **Photometric Calibration:**
    *   Apply the Beer-Lambert law to compute the final estimated pixel intensity $\hat{I}(p)$:
        $$ \hat{I}(p) = \hat{I}_0(p) \exp \left( - \hat{I}'_p(p) \right) $$
        where $\hat{I}_0(p)$ is the spatially varying source intensity (reference image) possibly modulated by per-view exposure multipliers [S1].

8.  **Loss Computation:**
    *   Compute the total loss $L_{total}$:
        $$ L_{total} = \sum_p ||I(p) - \hat{I}(p)||^2 + \lambda_{tv} L_{tv} + \lambda_{bc} L_{bc} $$
        where $I(p)$ is the measured intensity, $\lambda_{tv}$ and $\lambda_{bc}$ are hyperparameters [S1].
    *   **Total Variation (TV) Loss:** Compute on feature vectors (preferred for speed/quality):
        $$ L_{tv} = \sum_n \|\nabla f_n\|_1 $$
        where the gradient is computed over the feature grid indices of each leaf node [S1].
    *   **Boundary Consistency (BC) Loss:** Minimize feature discontinuities between neighboring nodes $n$ and $m$ on their shared boundary surface $\Lambda_{bc}$:
        $$ L_{bc} = \sum_{(n,m) \in N} \sum_{x \in \Lambda_{bc}} |f_n(x) - f_m(x)| $$
        where $N$ is the set of neighboring node pairs, and $f_n(x), f_m(x)$ are the interpolated features at boundary point $x$ from both nodes [S1].

9.  **Optimization:**
    *   Perform gradient descent to update:
        *   Neural feature grids in all leaf nodes.
        *   Decoder weights $\Phi$.
        *   Camera poses (geometric self-calibration).
        *   Reference image $I_0$ and exposure multipliers (photometric self-calibration) [S1].

10. **Octree Update (Adaptive Refinement):**
    *   Compute volumetric error for each node $n$:
        $$ E(n) = \rho_{max}(n) \cdot \sum_{(p,t_n,t_f) \in \Omega_n} ||I(p) - \hat{I}(p)||^2 $$
        where $\Omega_n$ is the set of samples passing through node $n$, and $\rho_{max}(n)$ is the maximum density in node $n$ [S1].
    *   Solve a mixed-integer program (MIP) to determine the optimal tree structure (splitting nodes with high error, pruning empty nodes) based on $E(n)$ [S1].
    *   Re-initialize features in newly split nodes and transfer/interpolate features from parent to children [S1].

11. **Iteration:**
    *   Repeat steps 2-10 until convergence or maximum iterations [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def train_neat(X_ray_images, initial_poses, initial_I0, num_iterations, lambda_tv, lambda_bc):
    """
    X_ray_images: List of 2D arrays, shape (H, W), measured intensities.
    initial_poses: List of camera parameters (source, detector).
    initial_I0: 2D array, reference image (empty scan).
    """
    # Initialize Octree
    octree = Octree(root_features=random_grid())
    
    # Initialize Decoder
    decoder = MLP()
    
    # Initialize Calibration Parameters
    poses = copy(initial_poses)
    I0 = copy(initial_I0)
    exposures = np.ones(len(X_ray_images))
    
    for iteration in range(num_iterations):
        total_loss = 0
        grad_features = []
        grad_decoder = []
        grad_poses = []
        grad_I0 = []
        grad_exposures = []
        
        # Accumulate gradients for all pixels
        for view_idx, I_measured in enumerate(X_ray_images):
            pose = poses[view_idx]
            exposure = exposures[view_idx]
            
            for p in pixels:
                # 1. Generate Ray
                ray = generate_ray(p, pose)
                
                # 2. Intersect Octree
                samples = octree.trace_ray(ray)
                
                # 3. Sample and Decode
                densities = []
                for sample in samples:
                    f = octree.interpolate_features(sample.position)
                    mu = decoder.decode(f)
                    densities.append(mu)
                
                # 4. Integrate (Simplified Riemann Sum)
                ray_integral = sum(d * w for d, w in zip(densities, samples.weights))
                
                # 5. Photometric Model
                I0_val = I0[p] * exposure
                I_estimated = I0_val * np.exp(-ray_integral)
                
                # 6. Loss Components
                mse_loss = (I_measured[p] - I_estimated) ** 2
                
                # TV Loss (computed on features)
                tv_loss = compute_tv_loss(octree)
                
                # BC Loss (computed on boundaries)
                bc_loss = compute_bc_loss(octree)
                
                # Total Loss for this pixel (approximate for gradient accumulation)
                # Note: In practice, gradients are backpropagated through the differentiable renderer
                pixel_loss = mse_loss + lambda_tv * tv_loss + lambda_bc * bc_loss
                
                # Accumulate gradients (conceptual)
                # grad_features.append(backprop_features(pixel_loss))
                # grad_decoder.append(backprop_decoder(pixel_loss))
                # grad_poses.append(backprop_poses(pixel_loss))
                # grad_I0.append(backprop_I0(pixel_loss))
                # grad_exposures.append(backprop_exposure(pixel_loss))
                
                total_loss += pixel_loss
        
        # Update Parameters
        octree.update_features(grad_features)
        decoder.update_weights(grad_decoder)
        poses.update(grad_poses)
        I0.update(grad_I0)
        exposures.update(grad_exposures)
        
        # Octree Refinement (every K iterations)
        if iteration % 10 == 0:
            octree.refine(lambda_tv, lambda_bc)

    return octree, decoder, poses, I0, exposures

class Octree:
    def __init__(self, root_features):
        self.root = Node(features=root_features)
    
    def trace_ray(self, ray):
        # Returns list of Sample objects with position, weight, and leaf_node reference
        pass
    
    def interpolate_features(self, position):
        # Find leaf node, compute local coords, trilinear interpolate
        pass
    
    def compute_tv_loss(self):
        # Sum of L1 norms of gradients of feature grids
        pass
    
    def compute_bc_loss(self):
        # Sum of L1 norms of feature differences at boundaries
        pass
    
    def refine(self, lambda_tv, lambda_bc):
        # Compute E(n) for each node
        # Solve MIP to split/prune
        # Re-initialize new nodes
        pass

class Node:
    def __init__(self, features=None):
        self.features = features # 3D grid of vectors
        self.children = [None] * 8
        self.is_leaf = features is not None
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
*   **Time Complexity:** The optimization process is iterative and involves differentiable volume rendering, ray tracing through an octree, and neural network decoding. The time complexity is dominated by the number of iterations, the number of pixels, and the depth of the octree. NeAT achieves reconstruction times far superior to existing neural inverse rendering methods [S1].
*   **Memory Complexity:** Memory usage is proportional to the number of leaf nodes in the octree and the size of the feature grids. The sparse nature of the octree allows for efficient memory usage compared to dense voxel grids [S1].
*   **Scaling:** NeAT scales well with resolution due to the adaptive octree structure, which only refines regions with detail. It handles full 3D cone beam data effectively [S1].
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
