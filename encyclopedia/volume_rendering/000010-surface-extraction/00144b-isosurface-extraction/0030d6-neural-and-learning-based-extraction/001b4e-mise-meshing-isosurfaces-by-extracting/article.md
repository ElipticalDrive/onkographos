# MISE (Meshing Isosurfaces by Extracting)
Also called: DVR Mesh Generation via MISE, Meshing Isosurfaces by Extracting

## Overview
MISE generates a 3D mesh from a trained Differentiable Volumetric Rendering (DVR) model by querying an implicit occupancy field using an adaptive resolution algorithm [S1, unofficial]. It sets itself apart from uniform grid-based extraction methods by iteratively refining the sampling grid only where the occupancy field is ambiguous, thereby achieving high-fidelity meshes with significantly fewer model evaluations than static high-resolution marching cubes.

## How it works
**Inputs and Outputs:**
*   **Input:** A trained DVR model (implicit occupancy function $f: \mathbb{R}^3 \rightarrow \mathbb{R}$), a bounding box defining the domain, and parameters: `resolution0` (initial grid size), `upsampling_steps` (number of refinement iterations), `threshold` (occupancy cutoff), `padding` (bounding box expansion), and `points_batch_size` (evaluation batch size).
*   **Output:** A triangular mesh representing the isosurface at the specified threshold.

**Data Structures:**
The core data structure is the MISE grid, which maintains a sparse set of active points and their corresponding occupancy values. The grid is implicitly defined by the bounding box and the current resolution level.

**Parameters:**
*   `threshold`: Default 0.5. The occupancy value at which the isosurface is extracted.
*   `resolution0`: Default 16. The side length of the initial uniform grid.
*   `upsampling_steps`: Default 3. The number of adaptive refinement steps.
*   `padding`: Default 0.1. Fractional padding added to the bounding box dimensions.
*   `points_batch_size`: Default 100,000. Number of points evaluated per model inference call.

**Algorithm Steps:**

1.  **Threshold Conversion:** Convert the linear occupancy threshold $\tau$ to log-odds space, as the model typically outputs logits.
    $$ \tau_{logit} = \ln(\tau) - \ln(1 - \tau) $$
    [S1, unofficial]

2.  **Initialization:**
    *   If `upsampling_steps` is 0: Generate a uniform 3D grid of points with resolution `resolution0` using `make_3d_grid`. Evaluate the model at these points and reshape to a 3D grid.
    *   If `upsampling_steps` > 0: Initialize the MISE extractor with `resolution0`, `upsampling_steps`, and $\tau_{logit}$.
    [S1, unofficial]

3.  **Adaptive Query Loop:**
    *   Query a set of candidate points from the MISE structure using `mesh_extractor.query()`.
    *   While the number of queried points is non-zero:
        1.  Normalize the queried points to the bounding box coordinates.
        2.  Evaluate the DVR model at these points to obtain occupancy logits.
        3.  Update the MISE structure with the new occupancy values using `mesh_extractor.update()`.
        4.  Query the next set of points for refinement using `mesh_extractor.query()`.
    [S1, unofficial]

4.  **Dense Grid Conversion:**
    *   Convert the sparse MISE structure into a dense 3D occupancy grid using `mesh_extractor.to_dense()`.
    [S1, unofficial]

5.  **Mesh Extraction:**
    *   Apply a standard isosurface extraction algorithm (e.g., Marching Cubes) to the dense 3D grid using the original linear `threshold` to generate the final mesh.
    [S1, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_mesh_mise(model, bounding_box, resolution0=16, upsampling_steps=3, 
                      threshold=0.5, padding=0.1, points_batch_size=100000):
    """
    Extracts a mesh from a DVR model using the MISE algorithm.
    
    Args:
        model: The trained DVR model with a forward method.
        bounding_box: Tuple (min_coords, max_coords) defining the domain.
        resolution0: Initial grid resolution.
        upsampling_steps: Number of adaptive refinement steps.
        threshold: Occupancy threshold for isosurface extraction.
        padding: Fractional padding for bounding box.
        points_batch_size: Batch size for model evaluation.
        
    Returns:
        vertices, faces: The extracted mesh.
    """
    # 1. Convert threshold to log-odds
    # [S1, unofficial]
    threshold_logit = np.log(threshold) - np.log(1.0 - threshold)
    
    # 2. Initialize MISE or uniform grid
    # [S1, unofficial]
    if upsampling_steps == 0:
        # Generate uniform grid
        points = make_3d_grid(bounding_box, resolution=resolution0)
        # Evaluate model
        logits = evaluate_model_batch(model, points, points_batch_size)
        # Reshape to 3D grid
        grid_shape = [resolution0] * 3
        dense_grid = logits.reshape(grid_shape)
        return marching_cubes(dense_grid, threshold)
    
    else:
        # Initialize MISE extractor
        # [S1, unofficial]
        # Note: Implementation details of MISE class are assumed to be available
        # as per the source code structure.
        mesh_extractor = MISEExtractor(
            resolution=resolution0,
            upsampling_steps=upsampling_steps,
            threshold=threshold_logit,
            padding=padding
        )
        
        # 3. Adaptive Query Loop
        # [S1, unofficial]
        points = mesh_extractor.query()
        while points.shape[0] != 0:
            # Normalize points to bounding box
            # Assuming points are in [0, 1] or similar normalized space
            # and need to be mapped to the actual bounding box
            normalized_points = normalize_to_bbox(points, bounding_box)
            
            # Evaluate model
            logits = evaluate_model_batch(model, normalized_points, points_batch_size)
            
            # Update MISE structure
            mesh_extractor.update(logits)
            
            # Query next set of points
            points = mesh_extractor.query()
            
        # 4. Convert to dense grid
        # [S1, unofficial]
        dense_grid = mesh_extractor.to_dense()
        
        # 5. Extract mesh
        return marching_cubes(dense_grid, threshold)

def evaluate_model_batch(model, points, batch_size):
    """
    Evaluates the model at given points in batches.
    """
    all_logits = []
    for i in range(0, len(points), batch_size):
        batch_points = points[i:i+batch_size]
        # Model expects specific input format, e.g., (N, 3)
        logits = model(batch_points)
        all_logits.append(logits)
    return np.concatenate(all_logits, axis=0)

def make_3d_grid(bounding_box, resolution):
    """
    Generates a uniform 3D grid of points within the bounding box.
    """
    min_coords, max_coords = bounding_box
    # Create linspace for each dimension
    x = np.linspace(min_coords[0], max_coords[0], resolution)
    y = np.linspace(min_coords[1], max_coords[1], resolution)
    z = np.linspace(min_coords[2], max_coords[2], resolution)
    # Create meshgrid
    xx, yy, zz = np.meshgrid(x, y, z, indexing='ij')
    # Stack and reshape to (N, 3)
    points = np.stack([xx, yy, zz], axis=-1).reshape(-1, 3)
    return points

def normalize_to_bbox(points, bounding_box):
    """
    Normalizes points from [0, 1] or similar to the bounding box.
    Assumes points are in a normalized space relative to the bounding box.
    """
    min_coords, max_coords = bounding_box
    # If points are already in the correct space, this might be identity
    # Based on source, points are normalized to the bounding box before evaluation
    # This function is a placeholder for the specific normalization logic
    # [G] Assuming points are in [0,1] and need to be scaled to bounding box
    return min_coords + points * (max_coords - min_coords)

def marching_cubes(grid, threshold):
    """
    Standard Marching Cubes algorithm.
    """
    # [G] Implementation of Marching Cubes is assumed to be available
    # as a standard utility.
    return mc.grid_to_mesh(grid, threshold)
```

**Edge Cases and Pitfalls:**
*   **Precision:** The conversion to log-odds space is critical for models trained with BCE loss. Using the linear threshold directly will result in incorrect isosurface locations.
*   **Boundaries:** The `padding` parameter ensures that the surface is not clipped at the bounding box edges. Insufficient padding may lead to incomplete meshes.
*   **Degenerate Input:** If the model outputs constant values, the adaptive refinement may not converge or may produce empty grids.

## Subtopics

### FlexiCubes
A differentiable meshing method that uses a flexible cube template to generate meshes directly from neural fields, offering higher quality and watertightness compared to MISE's marching cubes approach. [FlexiCubes](3669-flexicubes.md)

### NeRF Density Field Mesh Extraction
General methods for extracting meshes from NeRF density fields, often relying on uniform grid sampling and marching cubes, which are less efficient than MISE's adaptive approach. [NeRF Density Field Mesh Extraction](4104-nerf-density-field-mesh-extraction.md)

## Operating characteristics
*   **Time Complexity:** The time complexity depends on the number of model evaluations. MISE reduces the number of evaluations compared to uniform high-resolution sampling by focusing on ambiguous regions. However, the adaptive loop adds overhead.
*   **Memory Complexity:** The memory usage is dominated by the dense grid at the final resolution. The sparse MISE structure uses less memory during the iterative process.
*   **Measured Performance:** MISE is significantly faster than uniform high-resolution marching cubes for high-fidelity meshes, as it avoids evaluating the model at points far from the surface.
*   **Strengths:** Adaptive resolution, efficient use of model evaluations, high-fidelity meshes.
*   **Limitations:** Requires a trained DVR model, iterative nature can be slower than single-pass methods for low-resolution meshes.
*   **When to Use:** When high-fidelity meshes are required from a trained DVR model and computational efficiency is a concern.
*   **When Not to Use:** For low-resolution meshes or when a single-pass method is preferred.

## Background
MISE was introduced as part of the Differentiable Volumetric Rendering (DVR) framework by Liao et al. [S1, unofficial]. It addresses the challenge of extracting high-quality meshes from implicit occupancy fields learned by neural networks. The method leverages the adaptivity of the MISE algorithm to efficiently sample the occupancy field, reducing the computational cost of mesh extraction.

## Sources
[S1] autonomousvision/differentiable_volumetric_rendering, https://github.com/autonomousvision/differentiable_volumetric_rendering, fetched 2026-10-26 (unofficial)
