# Cone Discrete Sampling

Also called: IES Cone Sampling

## Overview
Cone Discrete Sampling is a volume rendering technique that alleviates aliasing by treating a camera pixel as a disc and casting a cone from the projection center, rather than a single ray. It integrates features within a series of conical frustums by blending data from the frustum's vertices using distance-dependent weights. It differs from its parent, [Cone Casting with Inscribed Sphere Sampling](11501-cone-casting-with-inscribed-sphere-sampling.md), by explicitly discretizing the cone into frustums and using vertex-based blending with learnable decay, and from [Local Implicit Ray Function (LIRF) Rendering](19676-local-implicit-ray-function-lirf-rendering.md) by relying on explicit tri-plane featurization and discrete vertex aggregation rather than continuous ray function approximation.

## How it works
**Inputs:**
*   Camera parameters (projection center, pixel corners).
*   A tri-plane based scene representation with $L$ levels of detail.
*   A set of sample points $\mathbf{x}$ along a ray within a conical frustum.

**Outputs:**
*   A Level-of-Detail (LoD) feature vector $\mathbf{Z}(\mathbf{x})$ for each sample point, which is subsequently passed to an MLP to predict color and signed distance.

**Data Structures:**
*   **Conical Frustum:** Defined by 8 vertices $\{\mathbf{x}_v\}_{v=1}^8$. These are obtained by casting rays through the pixel center and its four corners, intersected with the near and far planes of the frustum segment.
*   **Tri-plane Features:** For each vertex $\mathbf{x}_v$, features are projected onto three 2D planes (XY, YZ, XZ) to query base feature vectors $\mathbf{F}_l$.

**Parameters:**
*   $k$: Learnable weight decay parameter. Initial value is 80. [S1]
*   $\{\tau_l\}_{l=1}^L$: Kernel sizes for Gaussian convolution at each tri-plane level $l$. [S1]
*   $V$: Number of vertices in a conical frustum, fixed at 8. [S1]

**Steps:**

1.  **Frustum Construction:** Divide the ray through the pixel into a series of conical frustums. For each frustum, determine the 8 vertices $\mathbf{x}_v$ by intersecting rays from the camera center through the pixel corners and center with the frustum's near and far boundaries. [S1]

2.  **Multi-Convolved Featurization:** For each vertex $\mathbf{x}_v$, compute the multi-scale multi-convolved feature $\mathbf{G}_v(\mathbf{x}_v)$.
    *   Project $\mathbf{x}_v$ onto the tri-planes to query base feature vectors $\mathbf{F}_l$ for each level $l \in \{1, \dots, L\}$. [S1]
    *   Apply 2D Gaussian convolution $\mathcal{G}(\cdot, \tau_l)$ to each $\mathbf{F}_l$ using kernel size $\tau_l$. This aggregates features within a 3D sphere centered at $\mathbf{x}_v$ by leveraging the compressed 2D planes. [S1]
    *   Concatenate the convolved features from all levels:
        $$ \mathbf{G}_v(\mathbf{x}_v) = \bigcup_{l=1}^L \mathcal{G}(\mathbf{F}_l, \tau_l) \quad (5) $$
        where $\bigcup$ denotes concatenation. [S1]

3.  **Weight Calculation:** For a sample point $\mathbf{x}$ within the frustum, calculate the blending weight $W(\mathbf{x}, \mathbf{x}_v)$ for each vertex $\mathbf{x}_v$ based on the Euclidean distance:
    $$ W(\mathbf{x}, \mathbf{x}_v) = \exp(-k \|\mathbf{x}_v - \mathbf{x}\|) \quad (4) $$
    The parameter $k$ controls the decay rate. It must be aware of the frustum size; smaller frustums require a more rapid decrease (larger $k$) to maintain accuracy. [S1]

4.  **LoD Feature Aggregation:** Compute the final LoD feature $\mathbf{Z}(\mathbf{x})$ by summing the weighted vertex features:
    $$ \mathbf{Z}(\mathbf{x}) = \sum_{v=1}^V W(\mathbf{x}, \mathbf{x}_v) \mathbf{G}_v(\mathbf{x}_v) \quad (6) $$
    where $V=8$. [S1]

5.  **Prediction:** Pass $\mathbf{Z}(\mathbf{x})$ through a shallow 8-layer MLP to predict the signed distance $f$ and feature vector $\Theta$, from which color is derived. [S1]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def cone_discrete_sampling(sample_x, frustum_vertices, tri_planes, k, kernel_sizes):
    """
    Computes the LoD feature for a sample point within a conical frustum.
    
    Args:
        sample_x: 3D point (x, y, z) within the frustum.
        frustum_vertices: List of 8 vertices (x_v) defining the conical frustum.
        tri_planes: Dictionary or array containing the 2D feature planes for each level l.
        k: Learnable weight decay parameter (initially 80).
        kernel_sizes: List of L kernel sizes {tau_l} for Gaussian convolution.
        
    Returns:
        Z: Aggregated LoD feature vector.
    """
    V = 8
    L = len(kernel_sizes)
    Z = np.zeros_like(sample_x) # Placeholder for feature dimension; actual dim depends on tri-plane output
    
    # Step 2: Multi-Convolved Featurization for each vertex
    G_v_list = []
    for v in range(V):
        x_v = frustum_vertices[v]
        
        # Query base features from tri-planes for this vertex
        # Note: Implementation of tri-plane projection and querying is assumed to be handled by the 
        # underlying tri-plane representation structure.
        F_l_list = []
        for l in range(L):
            # Project x_v onto the 3 planes (XY, YZ, XZ) and query features
            # This returns a feature vector F_l for level l
            F_l = query_tri_plane(x_v, tri_planes[l]) 
            F_l_list.append(F_l)
            
        # Apply Gaussian convolution with different kernel sizes
        G_v = None
        for l in range(L):
            tau_l = kernel_sizes[l]
            # Apply 2D Gaussian convolution to F_l
            # The convolution aggregates features in the 2D plane, approximating 3D spherical aggregation
            G_l = gaussian_convolve_2d(F_l_list[l], tau_l)
            
            # Concatenate features from all levels
            if G_v is None:
                G_v = G_l
            else:
                G_v = np.concatenate([G_v, G_l], axis=-1)
                
        G_v_list.append(G_v)

    # Step 3 & 4: Weight Calculation and Aggregation
    Z = np.zeros_like(G_v_list[0])
    for v in range(V):
        x_v = frustum_vertices[v]
        G_v = G_v_list[v]
        
        # Calculate weight based on distance
        dist = np.linalg.norm(x_v - sample_x)
        W = np.exp(-k * dist)
        
        # Add weighted feature to sum
        Z += W * G_v
        
    return Z

def query_tri_plane(point, plane_features):
    """
    [G] Helper to simulate querying a tri-plane.
    Projects a 3D point onto 2D planes and interpolates features.
    Returns a feature vector.
    """
    # Project point (x,y,z) to XY, YZ, XZ
    xy_coords = (point[0], point[1])
    yz_coords = (point[1], point[2])
    xz_coords = (point[0], point[2])
    
    # Bilinear interpolation on each plane
    f_xy = interpolate_2d(plane_features['xy'], xy_coords)
    f_yz = interpolate_2d(plane_features['yz'], yz_coords)
    f_xz = interpolate_2d(plane_features['xz'], xz_coords)
    
    # Concatenate plane features
    return np.concatenate([f_xy, f_yz, f_xz], axis=-1)

def gaussian_convolve_2d(feature_map, kernel_size):
    """
    [G] Applies a 2D Gaussian convolution to a feature map.
    """
    # Create Gaussian kernel
    sigma = kernel_size / 2.0
    # Standard implementation of Gaussian kernel generation and convolution
    # ...
    return feature_map # Placeholder for actual convolution result
```

## Subtopics

### Multi-convolved Featurization
Uses 2D Gaussian convolution on each tri-plane to represent the region of integration for a conical frustum. For a vertex $\mathbf{x}_v$, the multi-scale multi-convolved feature $\mathbf{G}_v$ is defined by applying Gaussian convolutions with different kernel sizes $\{\tau_l\}$ for each level $l$. Features are combined using concatenation. Different kernel sizes cover various frequency details, allowing the representation to handle sparse samples within the frustum efficiently without increasing discretization density. [S1]

### LoD Feature Aggregation
The final step where the Level-of-Detail feature $\mathbf{Z}(\mathbf{x})$ is computed for a sample $\mathbf{x}$ by summing the weighted multi-convolved features of the frustum's vertices. This aggregates the continuous features from the vertices into a single representation for the sample point, enabling the subsequent MLP to predict color and geometry. [S1]

## Operating characteristics
*   **Complexity:** Reduces computational cost compared to increasing the number of sampled rays and points, as it integrates features within conical frustums using a fixed number of vertices (8) and levels. [S1]
*   **Performance:** Offers faster inference time than methods that increase ray counts to mitigate aliasing. [S1]
*   **Strengths:** Efficiently represents ray sampling within a cone using explicit tri-plane featurization. Avoids the complexity of shape-adaptive 3D Gaussian kernels by using a single learnable parameter $k$. [S1]
*   **Limitations:** The accuracy of the weight decay function $k$ is sensitive to the size of the conical frustum; improper tuning can lead to artifacts. [S1]
*   **When to use:** When rendering with tri-plane representations and needing to balance aliasing reduction with computational efficiency. [S1]

## Background
Cone Discrete Sampling was introduced in "Anti-Aliased Neural Implicit Surfaces with Encoding Level of Detail" (2024) by the authors of [S1]. It draws inspiration from Mip-NeRF but adapts the concept for tri-plane-based representations and explicit level-of-detail encoding rather than multi-resolution rendering. The method addresses the high computational costs associated with increasing sampled rays and points in volume rendering. [S1]

## Sources
[S1] Anti-Aliased Neural Implicit Surfaces with Encoding Level of Detail, https://doi.org/10.1145/3610548.3618197, fetched 2026-10-27
