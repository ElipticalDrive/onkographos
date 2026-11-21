# MISE (Meshing Isosurfaces by Extracting)
Also called: DVR Mesh Generation via MISE, Meshing Isosurfaces by Extracting

## Overview
MISE generates a 3D mesh from a trained Differentiable Volumetric Rendering (DVR) model by adaptively querying an implicit occupancy field. It solves the problem of efficiently extracting high-fidelity meshes from neural implicit representations without requiring a uniform high-resolution grid evaluation, which is computationally prohibitive. It sets itself apart from siblings like [FlexiCubes](3669-flexicubes.md) and [Differentiable Marching Cubes](9774-differentiable-marching-cubes.md) by using a hierarchical, adaptive point-querying strategy rather than fixed-grid marching cubes or differentiable grid-based extraction.

## How it works
**Inputs:**
- A trained DVR model (implicit occupancy field $f: \mathbb{R}^3 \rightarrow \mathbb{R}$).
- Bounding box defining the spatial domain.
- Parameters: `threshold`, `resolution0`, `upsampling_steps`, `padding`, `points_batch_size`.

**Outputs:**
- A dense 3D grid of occupancy values.
- A 3D mesh extracted via standard marching cubes on the dense grid.

**Data Structures:**
- `MISE` object: Manages the hierarchical grid structure and active points.
- `points`: A tensor of shape $(N, 3)$ containing the 3D coordinates of points to evaluate.
- `values`: A tensor of shape $(N, 1)$ containing the model output (logits) for the points.

**Steps:**
1.  **Threshold Conversion:** Convert the float occupancy threshold $t$ to log-odds space for the model output:
    $$ t_{logit} = \ln(t) - \ln(1 - t) $$
    [S1, unofficial]

2.  **Initialization:**
    - If `upsampling_steps` is 0:
        - Generate a uniform grid of points using `make_3d_grid` with resolution $n_x = \text{resolution0}$.
        - Evaluate the model on these points.
        - Reshape results to a 3D grid.
    - If `upsampling_steps` > 0:
        - Initialize the MISE extractor with `resolution0`, `upsampling_steps`, and $t_{logit}$.
        [S1, unofficial]

3.  **Adaptive Querying Loop:**
    - Query points from the MISE extractor: `points = mesh_extractor.query()`.
    - While `points.shape[0] != 0`:
        - Normalize `points` to the bounding box coordinates.
        - Evaluate the model on `points` in batches of size `points_batch_size`.
        - Update the MISE structure with the evaluated values: `mesh_extractor.update(values)`.
        - Query the next set of points: `points = mesh_extractor.query()`.
        [S1, unofficial]

4.  **Densification:**
    - Convert the MISE structure to a dense 3D grid: `grid = mesh_extractor.to_dense()`.
    - Apply marching cubes on the dense grid using the original float threshold $t$.
    [S1, unofficial]

**Parameters:**
- `threshold`: Occupancy threshold for marching cubes. Default: 0.5.
- `resolution0`: Starting resolution for MISE. Default: 16.
- `upsampling_steps`: Number of upsampling steps. Default: 3.
- `padding`: Padding added to the bounding box size. Default: 0.1.
- `points_batch_size`: Batch size for point evaluation. Default: 100000.
[S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import torch

def extract_mesh_mise(model, bounding_box, threshold=0.5, resolution0=16, 
                      upsampling_steps=3, padding=0.1, points_batch_size=100000):
    """
    Extract mesh using MISE from a trained DVR model.
    """
    # 1. Convert threshold to log-odds
    threshold_logit = np.log(threshold) - np.log(1.0 - threshold)
    
    # 2. Initialize MISE or uniform grid
    if upsampling_steps == 0:
        # Generate uniform grid
        points = make_3d_grid(resolution=resolution0, bounding_box=bounding_box)
        # Evaluate model
        values = evaluate_model(model, points, batch_size=points_batch_size)
        # Reshape to 3D grid
        grid = reshape_to_3d_grid(values, resolution=resolution0)
    else:
        # Initialize MISE extractor
        mesh_extractor = MISEExtractor(
            resolution0=resolution0,
            upsampling_steps=upsampling_steps,
            threshold=threshold_logit,
            padding=padding
        )
        
        # 3. Adaptive Querying Loop
        points = mesh_extractor.query()
        while points.shape[0] != 0:
            # Normalize points to bounding box
            normalized_points = normalize_to_bbox(points, bounding_box)
            
            # Evaluate model
            values = evaluate_model(model, normalized_points, batch_size=points_batch_size)
            
            # Update MISE structure
            mesh_extractor.update(values)
            
            # Query next points
            points = mesh_extractor.query()
        
        # 4. Densification
        grid = mesh_extractor.to_dense()
    
    # Extract mesh using marching cubes on the dense grid
    # Note: marching_cubes function is assumed to be available from a library like skimage
    vertices, faces, normals, values = marching_cubes(grid, level=threshold)
    
    return vertices, faces
```
[G] `make_3d_grid`, `normalize_to_bbox`, `evaluate_model`, `reshape_to_3d_grid`, `MISEExtractor`, `marching_cubes` are assumed to be implemented or available from standard libraries (e.g., `skimage.measure.marching_cubes`). The MISE extractor's internal logic for hierarchical querying is encapsulated in the `MISEExtractor` class.

## Operating characteristics
- **Time Complexity:** Sub-linear in the volume of the domain, as it only queries points near the isosurface. Complexity depends on the number of active points in the MISE hierarchy.
- **Memory Complexity:** Proportional to the number of points queried and the final dense grid size. The dense grid size is determined by the final resolution after upsampling steps.
- **Performance:** Efficient for high-resolution mesh extraction compared to uniform grid evaluation. Performance scales with the complexity of the shape and the number of upsampling steps.
- **Strengths:** Adaptive resolution, efficient memory usage for complex shapes.
- **Limitations:** Requires iterative querying, which may be slower for simple shapes compared to uniform grid methods. Sensitive to the choice of `resolution0` and `upsampling_steps`.
- **When to use:** When extracting high-fidelity meshes from neural implicit representations where uniform grid evaluation is too expensive.
- **When not to use:** For simple shapes where a low-resolution uniform grid suffices, or when real-time extraction is required and the overhead of adaptive querying is prohibitive.

## Background
MISE was introduced in the context of Differentiable Volumetric Rendering (DVR) for neural implicit surface reconstruction. It provides a method to efficiently extract meshes from the learned occupancy fields without the computational cost of evaluating the field on a dense uniform grid. The method leverages the hierarchical nature of the MISE algorithm to focus computation on regions of interest near the isosurface.

## Sources
[S1] autonomousvision/differentiable_volumetric_rendering, https://github.com/autonomousvision/differentiable_volumetric_rendering, fetched 2026-10-26 (unofficial)
