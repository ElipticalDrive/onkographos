# Point-to-Ellipsoid Network (P2ENet)
Also called: Neural Elliptical Parameter Estimation, P2ENet, Sparse 3D Convolutional Network for Gaussian Parameters

## Overview
P2ENet is a feed-forward 3D sparse convolutional neural network that transforms a colored 3D point cloud into a set of 3D Gaussian primitives (ellipsoids) with estimated parameters, enabling high-quality, hole-free rendering without per-scene optimization [S1]. It differs from per-scene optimization methods by using a single trained network to predict Gaussian parameters directly from the input geometry, and it differs from sibling methods like [Point-to-Gaussian Generation](15582-point-to-gaussian-generation.md) by employing a UNet-like sparse 3D convolutional architecture that processes the global context of the voxelized point cloud rather than local point neighborhoods alone [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A frame of 3D point cloud $P = \{(p_i, c_i) | i = 1, \dots, N\}$, where each point has a 3D coordinate $p=(x,y,z)$ and an RGB color $c=(r,g,b)$ [S1].
*   **Output:** For each point, a set of 11 parameters defining a 3D Gaussian primitive: center offset $\delta = (\delta_x, \delta_y, \delta_z)$, scale parameters $\sigma_x, \sigma_y, \sigma_z$, quaternion rotation $q = (q_w, q_x, q_y, q_z)$, and opacity $o$ [S1]. The network may also estimate a surface normal $n$ [S1].

**Data Structures and Layout:**
1.  **Input Feature Vector:** For each point, the input feature vector consists of absolute coordinates, color, and coordinate residuals: $\{p = (x, y, z), c = (r, g, b), r = (\delta_x, \delta_y, \delta_z)\}$ [S1].
2.  **Voxel Grid:** A sparse 3D voxel grid representing the point cloud. The grid is pruned to maintain geometry consistent with the encoder's occupancy [S1].
3.  **Primitive Parameters:** The final output for each occupied point is the tuple $\langle G(p + \delta, \Sigma), o, c, n \rangle$ [S1].

**Steps:**
1.  **Adaptive Voxelization:**
    *   Determine a scaling factor such that the scaled point cloud has an average density of 1 point per voxel [S1].
    *   Voxelize the scaled point cloud with a voxel size of 1 [S1].
    *   Calculate coordinate residuals $\delta = (\delta_x, \delta_y, \delta_z)$ resulting from the voxelization process (the difference between the original point coordinate and the voxel center) [S1].
    *   Construct the input feature for each point as $\{p, c, \delta\}$ [S1].

2.  **Encoder (Downsampling):**
    *   Process the voxelized data using a UNet-like architecture with 3D convolutions [S1].
    *   Perform 3 downsampling layers. In each layer, take every $2 \times 2 \times 2$ voxels and aggregate them into one voxel [S1].
    *   The new voxel is conceptually at the center of the $2 \times 2 \times 2$ block and holds the mean feature of the non-empty voxels in that block [S1].
    *   This effectively enlarges the voxel size by a factor of $8^3$ (or $2^9$) across the three layers [S1].

3.  **Decoder (Upsampling):**
    *   Perform 3 upsampling layers symmetric to the encoder [S1].
    *   In each layer, use transposed 3D convolution. Each non-empty voxel in the input generates $2 \times 2 \times 2$ voxels in the output [S1].
    *   Prune the output voxel grid using the ground truth occupancy mask from the corresponding encoder voxel grid to ensure the geometry matches the original point cloud structure [S1].
    *   The last layer of the decoder contains the same set of points as the original point cloud [S1].

4.  **Parameter Prediction:**
    *   For each occupied point in the final decoder layer, extract the point's feature vector [S1].
    *   Pass the feature vector through a shared Multi-Layer Perceptron (MLP) to predict the 11 Gaussian parameters [S1].

**Formulas:**
The covariance matrix $\Sigma \in \mathbb{R}^{3 \times 3}$ is parameterized to guarantee positive semi-definiteness [S1]:
$$ \Sigma = R^T S^T S R $$
where:
*   $R$ is the rotation matrix derived from the predicted quaternion $q = (q_w, q_x, q_y, q_z)$ [S1].
*   $S = \text{diag}(\sigma_x, \sigma_y, \sigma_z)$ is a diagonal matrix of scale parameters [S1].

The final Gaussian primitive is defined as $G(p + \delta, \Sigma)$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def p2enet(point_cloud):
    """
    point_cloud: List of tuples (p, c) where p=(x,y,z) and c=(r,g,b)
    Returns: List of tuples (delta, sigma_params, q, o) for each point
    """
    # Step 1: Adaptive Voxelization
    # Determine scaling factor for 1 point per voxel density
    scale_factor = compute_scale_for_density_1(point_cloud)
    scaled_points = [(p * scale_factor, c) for p, c in point_cloud]
    
    # Voxelize with voxel size 1
    voxel_grid = voxelize(scaled_points, voxel_size=1)
    
    # Calculate coordinate residuals
    input_features = []
    for p, c in point_cloud:
        # Find corresponding voxel center for p
        voxel_center = get_voxel_center(p, scale_factor, voxel_size=1)
        delta = p - voxel_center # (delta_x, delta_y, delta_z)
        input_features.append({
            'p': p,
            'c': c,
            'delta': delta
        })
        
    # Step 2: Encoder
    # Convert input features to sparse tensor
    sparse_tensor = create_sparse_tensor(input_features)
    
    # 3 Downsampling layers
    encoder_outputs = []
    for i in range(3):
        sparse_tensor = sparse_conv_downsample(sparse_tensor)
        encoder_outputs.append(sparse_tensor)
        
    # Step 3: Decoder
    # 3 Upsampling layers
    for i in range(3):
        # Transposed convolution
        sparse_tensor = sparse_conv_transpose(sparse_tensor)
        
        # Prune using encoder occupancy
        encoder_occ = encoder_outputs[2-i].occupancy_mask
        sparse_tensor = prune_sparse_tensor(sparse_tensor, encoder_occ)
        
    # Step 4: Parameter Prediction
    # The final sparse_tensor has the same points as the original cloud
    predictions = []
    for point_idx, features in enumerate(sparse_tensor.features):
        # Shared MLP
        params = shared_mlp(features)
        # params contains: delta (3), sigma (3), q (4), o (1)
        delta = params[0:3]
        sigma = params[3:6]
        q = params[6:10]
        o = params[10]
        predictions.append((delta, sigma, q, o))
        
    return predictions

def compute_scale_for_density_1(point_cloud):
    # Not given in the sources. Implementation depends on bounding box volume.
    pass

def voxelize(points, voxel_size):
    # Not given in the sources. Standard voxelization logic.
    pass

def get_voxel_center(p, scale_factor, voxel_size):
    # Not given in the sources.
    pass

def create_sparse_tensor(features):
    # Not given in the sources. Minkowski Engine specific.
    pass

def sparse_conv_downsample(sparse_tensor):
    # Takes every 2x2x2 voxels, aggregates to one with mean feature
    # Not given in the sources. Minkowski Engine specific.
    pass

def sparse_conv_transpose(sparse_tensor):
    # Each non-empty voxel generates 2x2x2 voxels
    # Not given in the sources. Minkowski Engine specific.
    pass

def prune_sparse_tensor(sparse_tensor, occupancy_mask):
    # Not given in the sources.
    pass

def shared_mlp(features):
    # Not given in the sources. Standard MLP structure.
    pass
```

**Parameters and Typical Values:**
*   **Voxelization Density:** 1 point per voxel [S1].
*   **Voxel Size:** 1 (after scaling) [S1].
*   **Encoder Layers:** 3 downsampling layers [S1].
*   **Decoder Layers:** 3 upsampling layers [S1].
*   **Splatting Boundary:** 3 times the standard deviation of the Gaussian [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** The parameterization $\Sigma = R^T S^T S R$ guarantees positive semi-definiteness, avoiding invalid covariance matrices [S1].
*   **Geometry Preservation:** The decoder pruning step is critical to ensure the output voxel grid matches the input geometry; without it, transposed convolutions may introduce artifacts or incorrect occupancy [S1].
*   **Input Preprocessing:** Point clouds should be preprocessed to remove noise and outliers before input [S1].

## Subtopics

### Point-to-Gaussian Generation
A sibling method that transforms point clouds into Gaussian ellipsoids. It uses the same variance function as the mesh converter but determines distance based on point proximity, whereas P2ENet uses a global sparse convolutional network. [Point-to-Gaussian Generation](15582-point-to-gaussian-generation.md)

## Operating characteristics
*   **Time Complexity:** Real-time processing is achieved on high-end GPUs (e.g., RTX 4090) for high-density point clouds [S1]. The sparse convolution approach is more efficient than nearest-neighbor based methods like DGCNN or PointNet++ for large-scale point clouds [S1].
*   **Memory Complexity:** Efficient handling of large point clouds with millions of points on GPU due to sparse data structures [S1].
*   **Strengths:** No per-scene optimization required; hole-free rendering; supports normal map generation for relighting and meshing; real-time performance [S1].
*   **Limitations:** Requires a trained neural network; performance depends on the quality of the training data (e.g., THuman 2.0 dataset) [S1].
*   **When to Use:** When rendering dynamic point cloud video or large static point clouds with high quality and speed, and when per-scene optimization is prohibitive [S1].
*   **When Not to Use:** When exact geometric fidelity without learned artifacts is required, or when no training data is available [S1].

## Background
P2ENet was introduced in "Low Latency Point Cloud Rendering with Learned Splatting" (2024) [S1]. It builds on the concept of 3D Gaussian splatting [15] and aims to overcome the need for per-scene optimization by using a feed-forward neural network. The method leverages the Minkowski Engine for efficient 3D sparse convolutions [S1]. It is designed for rendering 3D point cloud data captured by RGB-D cameras, LiDAR, or multi-view reconstruction [S1].

## Sources
[S1] Low Latency Point Cloud Rendering with Learned Splatting, https://doi.org/10.1109/cvprw63382.2024.00584, fetched 2026-10-26
