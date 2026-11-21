# Synthetic Phantom Generation
Also called: Sphere Phantom
## Overview
Synthetic Phantom Generation creates idealized 3D volume data, specifically consisting of spheres, to serve as ground truth for testing and validating CT reconstruction algorithms [S1, unofficial]. It differs from general voxelization by providing analytically defined, noise-free reference volumes rather than discretizing real-world scans, and from primitive-based generation by focusing specifically on spherical geometries for algorithmic benchmarking [S1, unofficial][18236-primitive-based-phantom-generation.md].
## How it works
### Inputs and Outputs
- **Inputs**: Phantom size (dimensions), number of spheres, sphere radii, and spatial coordinates for each sphere.
- **Outputs**: A 3D numpy array (volume) representing the synthetic CT phantom, typically exported as a raw volume or `.npy` file [S1, unofficial].

### Data Structures
- **Volume Grid**: A 3D array of shape $(D, D, D)$ where $D$ is the voxel count per dimension. Data type is `float32` [S1, unofficial].
- **Sphere Definition**: Each sphere is defined by its center coordinates $(x, y, z)$ and radius $r$.

### Steps
1. **Initialize Volume**: Create a 3D array of zeros with shape $(D, D, D)$ [S1, unofficial].
2. **Generate Spheres**: For each sphere $i$ with center $C_i$ and radius $r_i$:
   - Iterate through all voxel coordinates $V$ in the volume.
   - Calculate the Euclidean distance $d$ from $V$ to $C_i$.
   - If $d \le r_i$, set the voxel value to 1 (or a specific attenuation coefficient); otherwise, leave it at 0.
3. **Export**: Save the resulting volume array for use in reconstruction pipelines [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def generate_sphere_phantom(size, spheres):
    """
    Generate a synthetic CT phantom consisting of spheres.
    
    Args:
        size (int): The dimension of the cubic volume (size x size x size).
        spheres (list of tuples): Each tuple contains (x, y, z, radius).
        
    Returns:
        np.ndarray: 3D volume array representing the phantom.
    """
    # Initialize volume with zeros
    volume = np.zeros((size, size, size), dtype=np.float32)
    
    # Iterate over each sphere definition
    for x, y, z, radius in spheres:
        # Create coordinate grids for efficient distance calculation
        # Using meshgrid to avoid explicit loops over voxels
        xx, yy, zz = np.meshgrid(
            np.arange(size), 
            np.arange(size), 
            np.arange(size), 
            indexing='ij'
        )
        
        # Calculate squared Euclidean distance from each voxel to sphere center
        dist_sq = (xx - x)**2 + (yy - y)**2 + (zz - z)**2
        
        # Set voxels inside the sphere to 1
        # Using radius squared to avoid square root operations
        volume[dist_sq <= radius**2] = 1.0
        
    return volume
```

### Parameters
- **Size**: Resolution of the volume. Typical values depend on the target reconstruction resolution [S1, unofficial].
- **Spheres**: List of `(x, y, z, r)` tuples. Coordinates are in voxel units [S1, unofficial].

### Edge Cases and Pitfalls
- **Boundary Conditions**: Ensure sphere centers and radii are defined such that spheres do not extend beyond the volume boundaries, or handle clipping explicitly [G].
- **Precision**: Use `float32` for consistency with typical CT reconstruction pipelines using PyTorch [S1, unofficial].
- **Overlap**: If spheres overlap, the current implementation sets overlapping voxels to 1. For distinct materials, a different accumulation strategy is required [G].

## Subtopics

### [Primitive-based Phantom Generation](18236-primitive-based-phantom-generation.md)
Generates 3D CT phantoms by combining mathematical primitives (spheres, spheroids, cuboids, cylinders, checkers) within a specified volume shape and geometry, using voxel coordinates in millimeters. Choose this when you need varied shapes beyond spheres for more complex scene testing.

## Operating characteristics
- **Time Complexity**: $O(N \cdot V)$ where $N$ is the number of spheres and $V$ is the number of voxels. Optimized with vectorized operations (meshgrid) to reduce constant factors [S1, unofficial].
- **Memory Complexity**: $O(V)$ for storing the volume array [S1, unofficial].
- **Strengths**: Fast generation, analytically precise ground truth, easy to integrate with PyTorch-based reconstruction pipelines [S1, unofficial].
- **Limitations**: Limited to spherical shapes; does not model complex anatomical structures or material variations [S1, unofficial].
- **When to Use**: Validating reconstruction algorithms (FDK, iterative methods) where ground truth is required [S1, unofficial].
- **When Not to Use**: When realistic tissue heterogeneity or complex geometries are needed [18236-primitive-based-phantom-generation.md].

## Background
Synthetic phantom generation is a standard technique in CT research for validating reconstruction algorithms. The specific implementation described here is part of a project exploring hybrid reconstruction combining physics and learning, utilizing a cone-beam forward projector and FDK implementation [S1, unofficial]. It serves as a foundational step in generating synthetic CT training data by providing clean input-target pairs for comparison with noisy reconstructions [S1, unofficial].

## Sources
[S1] ayush-chauhan-1202/physics-aware-ct-reconstruction, https://github.com/ayush-chauhan-1202/physics-aware-ct-reconstruction, fetched 2026-10-27 (unofficial)
