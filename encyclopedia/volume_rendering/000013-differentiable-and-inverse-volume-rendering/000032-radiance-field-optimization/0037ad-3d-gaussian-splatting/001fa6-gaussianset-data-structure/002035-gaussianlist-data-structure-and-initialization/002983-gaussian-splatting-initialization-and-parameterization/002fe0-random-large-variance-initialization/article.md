# Random Large-Variance Initialization

Also called: RAIN-GS, RAIN-GS Initialization, Sparse-Large-Variance Init

## Overview
Random Large-Variance Initialization (RAIN-GS) relaxes the strict Structure-from-Motion (SfM) point cloud assumption required by standard 3D Gaussian Splatting by introducing sparse, large-variance random initialization [S1, unofficial]. Unlike standard methods that rely solely on dense SfM points, this approach allows optimization to proceed even when SfM data is sparse or absent, serving as a fallback or complementary initialization strategy [S1, unofficial].

## How it works
**Inputs:**
- A set of $N$ 3D points $Q = \{q_1, \dots, q_N\}$ from a SfM reconstruction, where each $q_i \in \mathbb{R}^3$.
- A scaling constant $k$ (typically $k=1$ or similar small value, though exact typical values are not given in the sources).

**Outputs:**
- A set of 3D triangles, each defined by three vertices $\{v_1, v_2, v_3\}$, centered around each SfM point.

**Data Structures:**
- Each triangle is represented by three 3D vertex positions $v_i \in \mathbb{R}^3$.
- Additional learnable parameters per triangle: sharpness $\sigma$, opacity $o$, and spherical harmonic color coefficients $c$.

**Steps:**
1. For each SfM point $q \in Q$:
   a. Identify the three nearest neighbors of $q$ in the point cloud.
   b. Calculate $d$, the average Euclidean distance from $q$ to these three nearest neighbors.
   c. Sample three vertices $\{u_1, u_2, u_3\}$ uniformly at random from the unit sphere ($\|u_i\| = 1$).
   d. Scale the sampled vertices by $d$ and translate them by $q$ to create the triangle vertices:
      $$v_i = q + k \cdot d \cdot u_i, \quad i \in \{1, 2, 3\}$$
   e. This results in an approximately equilateral, randomly oriented triangle centered at $q$ with size proportional to local point density $d$.

**Parameters:**
- $k$: Scaling constant determining triangle size relative to neighbor distance $d$. Value not specified in sources.

**Edge Cases and Pitfalls:**
- If $d=0$ (e.g., duplicate points), the triangle collapses. A guard should handle $d \approx 0$ by using a minimum distance or skipping the point.
- Uniform sampling on the unit sphere must be done correctly (e.g., using spherical coordinates with $\cos \theta$ uniform) to avoid clustering at poles.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def sample_uniform_sphere(n):
    """Sample n points uniformly on the unit sphere."""
    # Using the method: phi uniform in [0, 2pi), cos(theta) uniform in [-1, 1]
    phi = np.random.uniform(0, 2 * np.pi, n)
    cos_theta = np.random.uniform(-1, 1, n)
    sin_theta = np.sqrt(1 - cos_theta**2)
    x = sin_theta * np.cos(phi)
    y = sin_theta * np.sin(phi)
    z = cos_theta
    return np.stack([x, y, z], axis=-1)

def random_large_variance_init(points, k=1.0):
    """
    Initialize triangles around SfM points using sparse large-variance random init.
    
    Args:
        points: Array of shape (N, 3) containing SfM 3D points.
        k: Scaling constant for triangle size.
        
    Returns:
        triangles: Array of shape (N, 3, 3) containing vertices for each triangle.
    """
    N = points.shape[0]
    triangles = np.zeros((N, 3, 3))
    
    # Compute nearest neighbors (assuming efficient KDTree or similar)
    # For simplicity, we assume a function get_3_nearest_neighbors exists
    # which returns indices of the 3 nearest neighbors for each point.
    # In practice, use scipy.spatial.cKDTree or similar.
    # Here we simulate the distance calculation.
    
    # Placeholder for neighbor finding logic [G]
    # In implementation: use KDTree.query(points, k=4) to get 3 nearest neighbors
    # (excluding self).
    
    for i, q in enumerate(points):
        # Find 3 nearest neighbors
        # dists, indices = tree.query(q, k=4) # k=4 to exclude self
        # neighbor_indices = indices[1:] # Skip self
        # neighbor_points = points[neighbor_indices]
        
        # Calculate average distance d
        # d = np.mean(np.linalg.norm(neighbor_points - q, axis=1))
        
        # Handle edge case: if d is very small or zero
        if d < 1e-6:
            # Fallback: use a small random offset or skip
            # For now, assume d > 0
            pass
            
        # Sample 3 vertices from unit sphere
        u = sample_uniform_sphere(3) # Shape (3, 3)
        
        # Scale and translate
        triangles[i] = q + k * d * u
        
    return triangles
```

## Subtopics

### SfM-Based Triangle Initialization
This variant explicitly uses the SfM point cloud to define the initial geometry. It creates triangles centered at each SfM point, with size proportional to the local point density (average distance to nearest neighbors). It is chosen when SfM data is available but sparse, allowing the model to explore a larger initial volume than point-based initialization. It differs from pure random initialization by anchoring the geometry to known scene structure.

## Operating characteristics
- **Time Complexity:** $O(N \log N)$ for nearest neighbor search using KDTree, where $N$ is the number of SfM points. Sampling and scaling are $O(N)$.
- **Memory Complexity:** $O(N)$ to store the triangle vertices.
- **Strengths:** Allows optimization in scenes with sparse SfM data; provides a structured initial guess that respects local geometry.
- **Limitations:** Still relies on some SfM data; performance degrades if SfM points are very sparse or noisy.
- **When to use:** When SfM reconstruction is incomplete or sparse, but some structure exists.
- **When not to use:** When no SfM data is available (use pure random init) or when SfM is dense and accurate (standard 3DGS init may be better).

## Background
Originated in the RAIN-GS method, which relaxes the SfM-only initialization assumption [S1, unofficial]. It is part of the broader effort to make Gaussian Splatting more robust to varying input data quality. The triangle-based initialization is detailed in "Triangle Splatting for Real-Time Radiance Field Rendering" [S2].

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S2] Triangle Splatting for Real-Time Radiance Field Rendering, https://doi.org/10.1109/3DV69130.2026.00123, fetched 2026-10-27
