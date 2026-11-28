# SfM-Based Triangle Initialization

Also called: Initialization and Training, SfM Triangle Init, Triangle Initialization

## Overview
SfM-Based Triangle Initialization is a method for initializing 3D triangle primitives for differentiable rendering by leveraging Structure-from-Motion (SfM) point clouds. It solves the problem of generating a geometrically plausible starting configuration for triangle splats, ensuring they are sized appropriately relative to local point density and randomly oriented to avoid bias. It sets apart from sibling methods like [Random Large-Variance Initialization](6129-random-large-variance-initialization.md) by using deterministic SfM data for position and local density for scale, rather than purely random sampling with large variance.

## How it works
**Inputs and Outputs:**
- Input: A set of 3D points $Q = \{q_1, ..., q_N\}$ from an SfM reconstruction.
- Output: A set of $N$ triangles, each defined by three vertices $V_i = \{v_{i,1}, v_{i,2}, v_{i,3}\}$, along with initial parameters for sharpness $\sigma$, opacity $o$, and spherical harmonic color coefficients $c$.

**Data Structures:**
- Each triangle $i$ is stored as a list of three 3D vertices.
- Associated scalar/vector attributes (opacity, SH coefficients) are stored per triangle.

**Steps:**
1. **Point Sampling (Optional, per [S2, unofficial]):**
   - If using the hybrid sampling strategy from [S2, unofficial], select $2/3$ of the target points using Farthest Point Sampling (FPS) on the SfM points $Q$, and the remaining $1/3$ uniformly at random. Let the selected subset be $Q'$. If not specified, use all $Q$.
2. **Local Density Estimation:**
   - For each point $q \in Q'$, find its three nearest neighbors in $Q$.
   - Calculate $d$ as the average Euclidean distance from $q$ to these three neighbors.
3. **Vertex Initialization:**
   - Sample three vertices $\{u_1, u_2, u_3\}$ uniformly at random from the unit sphere.
   - Scale and translate these vertices to form the triangle around $q$:
     $$v_{i,j} = q + k \cdot d \cdot u_j, \quad j \in \{1, 2, 3\}$$
     where $k$ is a scaling constant. [S1] describes this as creating an approximately equilateral, randomly oriented triangle with size proportional to $d$. [S2, unofficial] specifies the radius (distance from center to vertex) is set to one quarter of the average nearest-neighbor distance, implying $k \approx 0.25$ in that context, though [S1] leaves $k$ as a general constant.
4. **Parameter Initialization:**
   - Initialize opacity $o$, sharpness $\sigma$, and spherical harmonic coefficients $c$ to default values (e.g., opacity to 1.0, SH to white/neutral, sharpness to a standard value). [S1] mentions optimizing these; [S2, unofficial] does not specify initial values for non-vertex parameters.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def initialize_triangles(sfm_points, k=1.0, use_fps=False, fps_ratio=2/3):
    """
    Initialize 3D triangles from SfM points.
    
    Args:
        sfm_points: Array of shape (N, 3) containing SfM 3D points.
        k: Scaling constant for triangle size relative to neighbor distance.
        use_fps: If True, sample points using FPS for 2/3 of points.
        fps_ratio: Ratio of points to sample via FPS.
        
    Returns:
        vertices: Array of shape (N, 3, 3) containing 3 vertices per triangle.
    """
    N = sfm_points.shape[0]
    vertices = np.zeros((N, 3, 3))
    
    # Step 1: Point Sampling
    if use_fps:
        # Farthest Point Sampling implementation required
        # Select fps_ratio * N points via FPS, rest randomly
        # Note: Exact FPS algorithm not detailed in sources, assumed available
        indices = farthest_point_sampling(sfm_points, int(fps_ratio * N))
        random_indices = np.random.choice(N, N - len(indices), replace=False)
        selected_indices = np.concatenate([indices, random_indices])
        # Shuffle to maintain correspondence if needed, but order doesn't matter for set
        selected_points = sfm_points[selected_indices]
        # Map back to original indices if tracking is needed, but here we just process selected
        # For simplicity, we assume we process the selected subset and map results back
        # However, the output should be N triangles. 
        # Let's assume we initialize triangles for ALL points, but the 'target' points 
        # for initialization logic are the sampled ones. 
        # Re-reading [S2, unofficial]: "sample two-thirds... and draw remaining one-third... 
        # We then randomly initialize triangles around THESE sampled points".
        # This implies only sampled points get initialized? Or all points get initialized 
        # but the distribution is biased? 
        # [S1] implies initializing for "each SfM 3D point q".
        # [S2, unofficial] says "initialize triangles around these sampled points".
        # Ambiguity: Does [S2, unofficial] discard unsampled points? 
        # Given "Extreme Radiance Field Simplification", it likely reduces primitives.
        # But the parent is "Gaussian Splatting Initialization", which usually keeps N.
        # Let's assume for SfM-Based Triangle Initialization (general), we use all points.
        # The FPS part is a variant detail from [S2, unofficial].
        pass
    
    # Use all points for the general method as per [S1]
    points_to_init = sfm_points
    
    # Step 2 & 3: For each point
    for i, q in enumerate(points_to_init):
        # Find 3 nearest neighbors
        # Compute distances to all other points
        diffs = points_to_init - q
        dists = np.linalg.norm(diffs, axis=1)
        # Exclude self (distance 0)
        dists[i] = np.inf
        nearest_idx = np.argsort(dists)[:3]
        nearest_points = points_to_init[nearest_idx]
        
        # Average distance d
        d = np.mean(np.linalg.norm(nearest_points - q, axis=1))
        
        # Sample 3 vertices from unit sphere
        u = np.random.randn(3, 3)
        u = u / np.linalg.norm(u, axis=1, keepdims=True)
        
        # Scale and center
        vertices[i] = q + k * d * u
        
    return vertices
```

**Parameters:**
- $k$: Scaling constant. [S1] uses a general $k$. [S2, unofficial] implies $k=0.25$ (radius is $1/4$ of average NN distance).
- $\lambda$: Weight for L1 vs SSIM loss. Set to 0.8 [S2, unofficial].
- Optimizers: VectorAdam for vertex positions, Adam for other parameters [S2, unofficial].

**Edge Cases:**
- Sparse SfM points: $d$ may be large, leading to oversized triangles.
- Dense clusters: $d$ may be small, leading to tiny triangles.
- Degenerate triangles: Random sampling on unit sphere is unlikely to produce collinear points, but precision issues could arise.

## Subtopics

### [HDR-GS Initialization and Optimization](19685-hdr-gs-initialization-and-optimization.md)
HDR-GS initialization and optimization is a method for initializing 3D Gaussian primitives and training a Gaussian Splatting model for High Dynamic Range (HDR) novel view synthesis.

## Operating characteristics
- **Time Complexity:** Initialization is $O(N \cdot K)$ where $K$ is the number of points used for nearest neighbor search (typically small, e.g., 3-10). If FPS is used, it adds $O(N^2)$ or $O(N \log N)$ depending on implementation.
- **Memory Complexity:** $O(N)$ for storing vertices and attributes.
- **Strengths:** Geometrically aware initialization; adapts to local scene density; avoids the "blobby" artifacts of large-variance random init.
- **Limitations:** Requires high-quality SfM point cloud; sensitive to SfM noise/outliers; [S2, unofficial] suggests it may require more iterations or specific loss terms for texture.

## Background
Originated in the context of differentiable triangle splatting. [S1] (Triangle Splatting) proposes this initialization to replace Gaussian initialization with triangle primitives. [S2, unofficial] (DiffSoup) adapts it for extreme simplification, using a hybrid sampling strategy.

## Sources
[S1] Triangle Splatting for Real-Time Radiance Field Rendering, https://doi.org/10.1109/3DV69130.2026.00123, fetched 2026-10-27
[S2] DiffSoup: Direct Differentiable Rasterization of Triangle Soup for Extreme Radiance Field Simplification, https://doi.org/10.48550/arXiv.2603.27151, fetched 2026-10-30 (unofficial)
