# Distance Transformation (Proximity Clouds)
Also called: Proximity Clouds

## Overview
Distance Transformation, also known as Proximity Clouds, is a method to accelerate ray traversal in a regular grid filled with empty zones by pre-calculating distance information to the nearest full element in the grid [S1, unofficial]. It differs from hierarchical structures by storing distance data in a flat 3D volume aligned with the original voxel indices, allowing for $O(1)$ lookup during traversal without tree navigation overhead [S2, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A binary volume where cells are classified as transparent (empty) or opaque (full) [S2, unofficial].
*   **Output:** A distance volume (distance map) where each cell contains the distance to the nearest non-empty cell [S2, unofficial].

**Data Structures**
*   **Distance Volume:** A 3D grid with the same dimensions as the input binary volume. Each cell stores a scalar value representing the Euclidean distance between the center of the current cell and the center of the nearest non-empty cell [S2, unofficial].

**Algorithm Steps**
1.  **Generate Input Binary Volume:** Encode the scene geometry into a binary volume, marking cells as transparent or opaque [S2, unofficial].
2.  **Compute Distance Map:** Calculate the distance from every cell to the nearest non-empty cell. This can be done using exact metrics (City Block, Euclidean, Chessboard) or approximate methods like Chamfering [S2, unofficial].
    *   *Chamfering:* Uses a mask of local distances propagated through the volume. Larger masks yield higher accuracy but increase preprocessing time [S2, unofficial].
3.  **Ray Casting with Adaptive Traversal:** For each ray, determine the current cell index using fixed-point arithmetic and integer division [S2, unofficial].
    *   **Case A (Vicinity of Object):** If the current sample is near an object (typically implied by a small distance value or specific threshold), perform simple incremental ray traversal (step-by-step sampling) [S2, unofficial].
    *   **Case B (Empty Space):** If the current sample is in empty space, use the distance value $d$ stored in the current cell to skip the empty region [S2, unofficial].
4.  **Skip Calculation:** Calculate the new sample location to jump to the edge of the empty zone.
    *   Formula: $$ \text{New Sample} = \text{Current Sample} + \text{Unit Ray Direction} \times (d - 1) $$
    *   The term $(d - 1)$ is used instead of $d$ because the distance $d$ is calculated from the center of the current cell. Stepping by $d$ would skip beyond the free zone into the object [S2, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_distance_map(binary_volume):
    """
    Computes the distance map from a binary volume.
    binary_volume: 3D array where 1 is opaque, 0 is transparent.
    Returns: 3D array of distances.
    """
    # Implementation depends on chosen metric (Euclidean, Chamfering, etc.)
    # Not given in the sources. [G]
    pass

def ray_cast_proximity_clouds(ray_origin, ray_direction, distance_volume, resolution):
    """
    Casts a ray using Proximity Clouds acceleration.
    """
    # Initialize current position
    t = 0.0
    current_pos = ray_origin
    
    # Determine initial cell index
    # Using fixed-point arithmetic and integer division as per source [S2, unofficial]
    # Not given in the sources. [G]
    current_cell = get_cell_index(current_pos, resolution)
    
    while t < max_distance:
        # Get distance to nearest object from the precomputed map
        d = distance_volume[current_cell]
        
        # Check if we are in the vicinity of an object
        # The source implies a threshold or logic to switch strategies.
        # If d is small, we are near an object.
        if d < threshold:
            # Simple incremental traversal
            step_size = 1.0 / resolution # Or voxel size
            current_pos = ray_origin + ray_direction * (t + step_size)
            t += step_size
            current_cell = get_cell_index(current_pos, resolution)
        else:
            # Skip empty region
            # Formula: New sample = Current + direction * (d - 1)
            # Note: 'Current' here refers to the sample location within the cell,
            # but the source simplifies this to adding to the current sample location.
            # To be precise, we jump to the boundary of the empty space.
            
            # The source states: "The new sample is determined by adding the unit 
            # direction of the given ray multiplied by d − 1 to the current sample location."
            skip_distance = d - 1.0
            
            # Update position
            current_pos = current_pos + ray_direction * skip_distance
            t += skip_distance
            
            # Update cell index
            current_cell = get_cell_index(current_pos, resolution)
            
            # If we jumped past the object or out of bounds, break
            if not is_inside_volume(current_pos, resolution):
                break
```

**Parameters**
*   **Distance Metric:** Choice between City Block, Euclidean, Chessboard, or Chamfering approximation. A compromise between preprocessing time and rendering accuracy is required [S2, unofficial].
*   **Threshold:** The value distinguishing "vicinity of an object" from "empty space" is not explicitly defined in the sources [G].

**Edge Cases and Pitfalls**
*   **Stepping Distance:** The stepping distance must be $d-1$, not $d$, to avoid skipping beyond the free zone because the distance is calculated from the center of the current cell [S2, unofficial].
*   **Dynamic Scenes:** The method is too costly for dynamic scenes because the distance map must be recomputed for every frame [S1, unofficial].

## Operating characteristics
*   **Preprocessing Time:** Depends on the distance metric. Chamfering is efficient, but higher accuracy requires larger masks and longer preprocessing [S2, unofficial].
*   **Rendering Speed:** Fast for scenes with large empty regions due to skipping. The source notes it is considered the optimal structure on GPU for scenes with uniform data distribution [S1, unofficial].
*   **Memory:** Requires storing an additional 3D volume for distances, in addition to the binary volume [S2, unofficial].
*   **Limitations:** Not suitable for dynamic scenes due to high preprocessing cost [S1, unofficial]. Hierarchical structures may have additional traversal costs for cells at different levels, which this method avoids [S2, unofficial].

## Background
*   **Origin:** Presented by Cohen and Sheffer in 1994 [S1, unofficial].
*   **History:** The technique was introduced to accelerate ray traversal in regular grids with empty zones. It is particularly effective for static scenes [S1, unofficial].
*   **Usage:** Used in medical volume rendering and other domains where static scenes with significant empty space are common [S2, unofficial].

## Sources
[S1] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
[S2] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
