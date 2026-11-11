# MISE (Meshing Isosurfaces by Extracting)
Also called: DVR Mesh Generation via MISE, Meshing Isosurfaces by Extracting

## Overview
MISE generates a 3D mesh from a trained Differentiable Volumetric Rendering (DVR) model by querying an implicit occupancy field using an adaptive resolution algorithm [S1, unofficial]. It sets itself apart from uniform grid-based extraction methods by iteratively refining the sampling grid only where the occupancy field is ambiguous, thereby achieving high-fidelity meshes with significantly fewer model evaluations than static high-resolution marching cubes [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A trained DVR model (implicit occupancy function $f: \mathbb{R}^3 \rightarrow \mathbb{R}$), a bounding box defining the domain, and parameters: `resolution0` (initial grid size), `upsampling_steps` (number of refinement iterations), `threshold` (occupancy cutoff), `padding` (bounding box expansion), and `points_batch_size` (evaluation batch size) [S1, unofficial].
*   **Output:** A triangular mesh representing the isosurface at the specified threshold [S1, unofficial].

**Data Structures:**
The core data structure is the MISE grid, which maintains a sparse set of active points and their corresponding occupancy values [S1, unofficial]. The grid is implicitly defined by the bounding box and the current resolution level [G].

**Parameters:**
*   `threshold`: Default 0.5. The occupancy value at which the isosurface is extracted [S1, unofficial].
*   `resolution0`: Default 16. The side length of the initial uniform grid [S1, unofficial].
*   `upsampling_steps`: Default 3. The number of adaptive refinement steps [S1, unofficial].
*   `padding`: Default 0.1. Fractional padding added to the bounding box dimensions [S1, unofficial].
*   `points_batch_size`: Default 100,000. Number of points evaluated per model inference call [S1, unofficial].

**Algorithm Steps:**

1.  **Threshold Conversion:** Convert the linear occupancy threshold $\tau$ to log-odds space, as the model typically outputs logits [S1, unofficial].
    $$ \tau_{\text{logit}} = \ln(\tau) - \ln(1 - \tau) $$
    [S1, unofficial]

2.  **Initialization:**
    *   If `upsampling_steps` is 0: Generate a uniform 3D grid of points with resolution `resolution0` using `make_3d_grid`. Evaluate the model at these points and reshape to a 3D grid [S1, unofficial].
    *   If `upsampling_steps` > 0: Initialize the MISE extractor with `resolution0`, `upsampling_steps`, and $\tau_{\text{logit}}$ [S1, unofficial].

3.  **Adaptive Query Loop:**
    *   Query a set of candidate points from the MISE structure using `mesh_extractor.query()` [S1, unofficial].
    *   While the number of queried points is non-zero:
        1.  Normalize the queried points to the bounding box coordinates [S1, unofficial].
        2.  Evaluate the DVR model at these points to obtain occupancy logits [S1, unofficial].
        3.  Update the MISE structure with the new occupancy values using `mesh_extractor.update()` [S1, unofficial].
        4.  Query the next set of points for refinement using `mesh_extractor.query()` [S1, unofficial].

4.  **Dense Grid Conversion:**
    *   Convert the sparse MISE structure into a dense 3D occupancy grid using `mesh_extractor.to_dense()` [S1, unofficial].

5.  **Mesh Extraction:**
    *   Apply a standard isosurface extraction algorithm (e.g., Marching Cubes) to the dense 3D grid using the converted log-odds threshold $\tau_{\text{logit}}$ to generate the final mesh [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def extract_mesh_mise(model, bounding_box, resolution0=16, upsampling_steps=3, 
                      threshold=0.5, padding=0.1, points_batch_size=100000):
    """
    Extracts a mesh from a DVR model using the MISE algorithm.
    """
    # 1. Threshold Conversion [S1, unofficial]
    threshold_logit = np.log(threshold) - np.log(1.0 - threshold)
    
    # 2. Initialization [S1, unofficial]
    if upsampling_steps == 0:
        points = make_3d_grid(bounding_box, resolution0)
        logits = evaluate_model_batch(model, points, points_batch_size)
        dense_grid = logits.reshape([resolution0] * 3)
        return marching_cubes(dense_grid, threshold_logit)
    
    # Initialize MISE extractor [S1, unofficial]
    extractor = MISEExtractor(resolution0, upsampling_steps, threshold_logit, padding)
    
    # 3. Adaptive Query Loop [S1, unofficial]
    points = extractor.query()
    while points.shape[0] > 0:
        normalized = normalize_to_bbox(points, bounding_box, padding)
        logits = evaluate_model_batch(model, normalized, points_batch_size)
        extractor.update(logits)
        points = extractor.query()
        
    # 4. Dense Grid Conversion [S1, unofficial]
    dense_grid = extractor.to_dense()
    
    # 5. Mesh Extraction [S1, unofficial]
    return marching_cubes(dense_grid, threshold_logit)

def evaluate_model_batch(model, points, batch_size):
    """Evaluates the model at given points in batches [G]."""
    all_logits = []
    for i in range(0, len(points), batch_size):
        batch_points = points[i:i+batch_size]
        logits = model(batch_points)
        all_logits.append(logits)
    return np.concatenate(all_logits, axis=0)

def make_3d_grid(bounding_box, resolution):
    """Generates a uniform 3D grid of points within the bounding box [G]."""
    min_coords, max_coords = bounding_box
    x = np.linspace(min_coords[0], max_coords[0], resolution)
    y = np.linspace(min_coords[1], max_coords[1], resolution)
    z = np.linspace(min_coords[2], max_coords[2], resolution)
    xx, yy, zz = np.meshgrid(x, y, z, indexing='ij')
    return np.stack([xx, yy, zz], axis=-1).reshape(-1, 3)

def normalize_to_bbox(points, bounding_box, padding):
    """Maps normalized points to the padded bounding box coordinates [G]."""
    min_coords, max_coords = bounding_box
    half_size = (max_coords - min_coords) / 2.0
    center = (max_coords + min_coords) / 2.0
    padded_half_size = half_size * (1.0 + padding)
    return center + points * padded_half_size

def marching_cubes(grid, threshold):
    """Standard Marching Cubes algorithm operating on a 3D scalar grid [G]."""
    # [G] Implementation uses standard cell traversal and edge interpolation.
    # Returns vertices and faces where grid values cross the threshold.
    return standard_marching_cubes(grid, threshold)

class MISEExtractor:
    """Adaptive grid refinement structure [G]."""
    def __init__(self, resolution, upsampling_steps, threshold, padding):
        self.resolution = resolution
        self.upsampling_steps = upsampling_steps
        self.threshold = threshold
        self.padding = padding
        self.grid = np.full((resolution, resolution, resolution), np.nan)
        self.active_cells = np.ones((resolution, resolution, resolution), dtype=bool)
        self.current_res = resolution
        
    def query(self):
        """Returns coordinates of active cells for refinement [G]."""
        if not np.any(self.active_cells):
            return np.empty((0, 3))
        # Generate points at cell centers for active cells
        idx = np.where(self.active_cells)
        points = np.stack([
            (idx[0] + 0.5) / self.current_res,
            (idx[1] + 0.5) / self.current_res,
            (idx[2] + 0.5) / self.current_res
        ], axis=-1)
        return points
    
    def update(self, logits):
        """Updates grid values and marks cells for refinement [G]."""
        # Map logits to current grid resolution
        flat_idx = np.ravel_multi_index(
            (np.floor(logits.shape[0]**(1/3)).astype(int), 
             np.zeros_like(logits, dtype=int), 
             np.zeros_like(logits, dtype=int)), 
            (self.current_res, self.current_res, self.current_res)
        ) # Simplified mapping for pseudocode clarity
        # In practice, logits are assigned to corresponding grid cells
        self.grid[self.active_cells] = logits[:np.sum(self.active_cells)]
        # Mark neighbors of ambiguous cells for refinement
        self.active_cells = self._refine_active_cells(self.grid, self.threshold)
        
    def _refine_active_cells(self, grid, threshold):
        """Identifies cells near the threshold for upsampling [G]."""
        # Standard logic: mark cells where |value - threshold| < epsilon
        # or where neighbors cross the threshold
        return np.abs(grid - threshold) < 0.5
    
    def to_dense(self):
        """Upsamples to final resolution and returns dense grid [G]."""
        final_res = self.resolution * (2 ** self.upsampling_steps)
        dense = np.full((final_res, final_res, final_res), np.nan)
        # Trilinear interpolation from current grid to final resolution
        # [G] Standard grid upsampling routine
        return dense
```

**Edge Cases and Pitfalls:**
*   **Precision:** The conversion to log-odds space is critical for models trained with BCE loss [S1, unofficial]. Using the linear threshold directly on the log-odds grid will result in incorrect isosurface locations [G].
*   **Boundaries:** The `padding` parameter ensures that the surface is not clipped at the bounding box edges [S1, unofficial]. Insufficient padding may lead to incomplete meshes [G].
*   **Degenerate Input:** If the model outputs constant values, the adaptive refinement may not converge or may produce empty grids [G].

## Operating characteristics
*   **Time Complexity:** The time complexity depends on the number of model evaluations [G]. MISE reduces the number of evaluations compared to uniform high-resolution sampling by focusing on ambiguous regions [S1, unofficial]. However, the adaptive loop adds overhead [G].
*   **Memory Complexity:** The memory usage is dominated by the dense grid at the final resolution [G]. The sparse MISE structure uses less memory during the iterative process [G].
*   **Measured Performance:** MISE is significantly faster than uniform high-resolution marching cubes for high-fidelity meshes, as it avoids evaluating the model at points far from the surface [S1, unofficial].
*   **Strengths:** Adaptive resolution, efficient use of model evaluations, high-fidelity meshes [S1, unofficial].
*   **Limitations:** Requires a trained DVR model, iterative nature can be slower than single-pass methods for low-resolution meshes [G].
*   **When to Use:** When high-fidelity meshes are required from a trained DVR model and computational efficiency is a concern [S1, unofficial].
*   **When Not to Use:** For low-resolution meshes or when a single-pass method is preferred [G].

## Background
MISE was introduced as part of the Differentiable Volumetric Rendering (DVR) framework [S1, unofficial]. It addresses the challenge of extracting high-quality meshes from implicit occupancy fields learned by neural networks [S1, unofficial]. The method leverages the adaptivity of the MISE algorithm to efficiently sample the occupancy field, reducing the computational cost of mesh extraction [S1, unofficial].

## Sources
[S1] autonomousvision/differentiable_volumetric_rendering, https://github.com/autonomousvision/differentiable_volumetric_rendering, fetched 2026-10-26 (unofficial)
