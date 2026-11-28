# Self-Organizing Gaussians (SOG)
Also called: SOG Grid, Structured 2D Grid Mapping, gsplat Grid
## Overview
Self-Organizing Gaussians (SOG) solves the compression bottleneck of unstructured 3D Gaussian Splatting by mapping scattered Gaussian attributes onto a structured 2D grid [S1]. This spatial organization groups similar attributes together, transforming the data into a format compatible with standard image codecs (PNG, JPEG, EXR) [S2, unofficial]. Unlike CodecGS, which uses tri-plane feature prediction, SOG directly projects attributes into 2D maps to leverage existing lossy and lossless image compression pipelines [S1].
## How it works
**Inputs and Outputs:**
*   **Input:** A set of $N$ unstructured 3D Gaussians, each defined by position $\mathbf{p}_i$, covariance $\mathbf{\Sigma}_i$, opacity $\alpha_i$, spherical harmonic coefficients $\mathbf{c}_i$, and scale $\mathbf{s}_i$.
*   **Output:** A set of 2D texture maps (grids) representing the Gaussian attributes, ready for image-based compression.

**Data Structures:**
The method constructs a 2D grid (image) of fixed resolution $W \times H$. Each cell in the grid corresponds to a spatial region in the 3D scene. The grid stores flattened attribute vectors. For example, a grid might store position $(x, y, z)$, opacity, and the first few bands of spherical harmonics per cell. The exact layout depends on the attribute being mapped, but typically each pixel $(u, v)$ in the 2D grid maps to a specific Gaussian or a cluster of Gaussians.

**Steps:**
1.  **Grid Initialization:** Define a 2D grid of size $W \times H$. The resolution is chosen to balance compression ratio and reconstruction fidelity.
2.  **Spatial Mapping:** Project each unstructured Gaussian $i$ onto the 2D grid. This involves determining the grid cell $(u_i, v_i)$ that best represents the Gaussian's 3D position $\mathbf{p}_i$. The mapping function $f: \mathbb{R}^3 \to \mathbb{Z}^2$ assigns each Gaussian to a specific pixel or region.
3.  **Attribute Assignment:** Store the Gaussian's attributes (position, scale, rotation, opacity, SH coefficients) into the corresponding grid cells. If multiple Gaussians map to the same cell, they may be averaged, merged, or stored in a list depending on the specific implementation variant.
4.  **Smoothing Regularization (Training Phase):** During the optimization of the Gaussian parameters, a smoothing regularization term is added to the loss function. This term penalizes large differences between attributes of Gaussians mapped to adjacent grid cells, encouraging spatial smoothness in the 2D representation [S1]. This enhances compressibility by reducing high-frequency artifacts in the resulting 2D maps.
5.  **Compression:** The resulting 2D grids are encoded using standard image codecs (e.g., PNG for lossless, JPEG/EXR for lossy) [S2, unofficial].

**Formulas:**
The smoothing regularization term $L_{smooth}$ can be expressed as:
$$ L_{smooth} = \sum_{(u,v) \in Grid} \sum_{(u',v') \in Neighbors(u,v)} \| \mathbf{A}_{u,v} - \mathbf{A}_{u',v'} \|^2 $$
where $\mathbf{A}_{u,v}$ is the attribute vector stored at grid cell $(u,v)$, and $Neighbors(u,v)$ are the adjacent cells [S1].

**Parameters:**
*   Grid Resolution ($W \times H$): Not given in the sources.
*   Smoothing Weight: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Collision:** Multiple Gaussians mapping to the same grid cell require a strategy for merging or prioritizing attributes.
*   **Boundary Effects:** Gaussians near the edge of the scene may map to grid boundaries, potentially causing artifacts if not handled.
*   **Precision:** Quantization of attributes before grid storage can introduce errors, especially for high-frequency SH coefficients.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def self_organizing_gaussians(gaussians, grid_width, grid_height, smoothing_weight):
    """
    Maps unstructured Gaussians to a 2D grid for compression.
    
    Args:
        gaussians: List of Gaussian objects with attributes (pos, scale, rot, opacity, sh_coeffs).
        grid_width: Width of the 2D grid.
        grid_height: Height of the 2D grid.
        smoothing_weight: Weight for the smoothing regularization term.
        
    Returns:
        grid: 2D array of shape (grid_height, grid_width, num_attributes).
    """
    # Initialize grid with zeros or NaNs
    num_attributes = 3 + 1 + 3 + 3 + 27  # Example: pos(3) + opacity(1) + scale(3) + rot(4) + sh(27)
    grid = np.zeros((grid_height, grid_width, num_attributes))
    
    # Map Gaussians to grid cells
    for g in gaussians:
        # Project 3D position to 2D grid coordinates
        u, v = project_to_grid(g.pos, grid_width, grid_height)
        
        # Store attributes in the grid cell
        # Note: In case of collision, simple overwriting or averaging might be used
        grid[v, u] = flatten_attributes(g)
        
    # Apply smoothing regularization during training (conceptual)
    # This step is typically integrated into the training loop
    # loss = reconstruction_loss + smoothing_weight * compute_smoothing_loss(grid)
    
    return grid

def project_to_grid(pos, width, height):
    """
    Projects a 3D position to 2D grid coordinates.
    Implementation details depend on the specific projection method.
    """
    # Placeholder for projection logic
    # Typically involves normalizing 3D coordinates to [0, 1] and scaling to grid dimensions
    u = int(pos.x * width)
    v = int(pos.y * height)
    return u, v

def flatten_attributes(gaussian):
    """
    Flattens Gaussian attributes into a 1D array.
    """
    return np.concatenate([
        gaussian.pos,
        [gaussian.opacity],
        gaussian.scale,
        gaussian.rotation,
        gaussian.sh_coeffs.flatten()
    ])
```
## Subtopics

### CodecGS
Uses tri-plane architecture and video coding techniques for feature plane compression. Choose when higher compression ratios are needed at the cost of more complex decoding. [S1]

### PngCompression
A sibling method focusing on lossless PNG encoding of Gaussian parameters. Choose when lossless reconstruction is critical. [Parent]

### Splat parameter encoding and codec selection
Discusses general strategies for encoding Gaussian parameters. Choose when selecting the appropriate codec for a specific use case. [Parent]

## Operating characteristics
*   **Time Complexity:** Depends on the number of Gaussians $N$ and grid resolution $W \times H$. Mapping is $O(N)$. Compression depends on the image codec.
*   **Memory Complexity:** $O(W \times H \times \text{num\_attributes})$.
*   **Strengths:** Leverages mature, efficient image codecs. Simple to implement.
*   **Limitations:** Grid resolution limits spatial detail. Collision handling can be complex.
*   **When to use:** When standard image compression is preferred for simplicity and compatibility.
*   **When not to use:** When high spatial fidelity is required without grid quantization artifacts.
## Background
Self-Organizing Gaussians (SOG) was introduced as a method to compress 3D Gaussian Splatting representations by organizing them into a structured 2D grid [S1]. It is used in conjunction with the gsplat library [S1]. The method addresses the inefficiency of storing unstructured Gaussian data by exploiting spatial coherence.
## Sources
[S1] 3DGS.zip: A survey on 3D Gaussian Splatting Compression Methods, https://arxiv.org/abs/2407.09510v5, fetched 2026-10-26
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
