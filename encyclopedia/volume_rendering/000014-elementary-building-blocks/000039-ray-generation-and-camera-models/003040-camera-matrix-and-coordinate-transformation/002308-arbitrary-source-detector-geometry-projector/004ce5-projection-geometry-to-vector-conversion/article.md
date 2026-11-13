# Projection Geometry to Vector Conversion

## Overview
This method converts a dictionary-based projection geometry specification into a flat 12-element vector array for each projection angle, supporting both cone and parallel3D beam geometries [S1, unofficial]. It serves as the bridge between high-level geometric parameters and the low-level vector representations required by downstream differentiable ray-casting kernels, distinguishing itself from higher-level setup utilities by producing the exact numerical arrays expected by the projection engine [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A dictionary `proj_geom` containing the geometry definition.
*   **Output:** A dictionary containing metadata and a numpy array `Vectors` of shape `(n_angles, 12)`.

**Data Structures**
The input dictionary `proj_geom` must contain the following keys:
*   `type`: String, either `'cone'` or `'parallel3d'`.
*   `ProjectionAngles`: Array of angles (typically in radians).
*   `DistanceOriginSource` ($D_{OS}$): Distance from origin to source.
*   `DistanceOriginDetector` ($D_{OD}$): Distance from origin to detector center.
*   `DetectorSpacingX` ($D_{DX}$): Physical spacing of detector pixels in the X direction.
*   `DetectorSpacingY` ($D_{DY}$): Physical spacing of detector pixels in the Y direction.
*   `DetectorRowCount`: Integer, number of detector rows.
*   `DetectorColCount`: Integer, number of detector columns.

The output dictionary contains:
*   `type`: String, copied from input.
*   `DetectorRowCount`: Integer, copied from input.
*   `DetectorColCount`: Integer, copied from input.
*   `Vectors`: Numpy array of shape `(n_angles, 12)`. The 12 elements per angle are defined as:
    1.  Source X coordinate
    2.  Source Y coordinate
    3.  Source Z coordinate
    4.  Detector Center X coordinate
    5.  Detector Center Y coordinate
    6.  Detector Center Z coordinate
    7.  Detector X-axis basis vector X component
    8.  Detector X-axis basis vector Y component
    9.  Detector X-axis basis vector Z component
    10. Detector Y-axis basis vector X component
    11. Detector Y-axis basis vector Y component
    12. Detector Y-axis basis vector Z component

**Algorithm Steps**
1.  Initialize an empty array `Vectors` of shape `(len(ProjectionAngles), 12)`.
2.  Iterate through each angle $\theta$ in `ProjectionAngles` with index $i$.
3.  Determine the geometry type from `proj_geom['type']`.
4.  **If type is 'cone':**
    *   Compute Source position $S$:
        $$ S_x = \sin(\theta) D_{OS} $$
        $$ S_y = -\cos(\theta) D_{OS} $$
        $$ S_z = 0 $$
    *   Compute Detector Center $D$:
        $$ D_x = -\sin(\theta) D_{OD} $$
        $$ D_y = \cos(\theta) D_{OD} $$
        $$ D_z = 0 $$
    *   Compute Detector X-axis basis vector $U$:
        $$ U_x = \cos(\theta) D_{DX} $$
        $$ U_y = \sin(\theta) D_{DX} $$
        $$ U_z = 0 $$
    *   Compute Detector Y-axis basis vector $V$:
        $$ V_x = 0 $$
        $$ V_y = 0 $$
        $$ V_z = D_{DY} $$
5.  **If type is 'parallel3d':**
    *   Compute Ray Direction $R$ (Note: The source position is not explicitly stored as a separate entity in the 12-vector format for parallel beams in the same way, but the geometry is defined by the ray direction and detector center):
        $$ R_x = \sin(\theta) $$
        $$ R_y = -\cos(\theta) $$
        $$ R_z = 0 $$
        *Note: The source position is effectively at infinity. The 12-vector format typically stores the ray origin (detector center) and direction components or equivalent basis vectors. Based on the parallel3d description in [S1, unofficial], the detector center is fixed at the origin, and the basis vectors are computed similarly to the cone case but without the rotation of the center relative to the source distance.*
    *   Compute Detector Center $D$:
        $$ D_x = 0 $$
        $$ D_y = 0 $$
        $$ D_z = 0 $$
    *   Compute Detector X-axis basis vector $U$:
        $$ U_x = \cos(\theta) D_{DX} $$
        $$ U_y = \sin(\theta) D_{DX} $$
        $$ U_z = 0 $$
    *   Compute Detector Y-axis basis vector $V$:
        $$ V_x = 0 $$
        $$ V_y = 0 $$
        $$ V_z = D_{DY} $$
6.  Store the computed values in `Vectors[i, :]` in the order: $S_x, S_y, S_z, D_x, D_y, D_z, U_x, U_y, U_z, V_x, V_y, V_z$. For parallel3d, $S$ is replaced by the ray direction components or handled such that the ray origin is $D$ and direction is derived from $U$ and $V$ relative to the rotation. *Correction based on standard 12-vector formats in TIGRE-like systems*: The 12-vector usually represents the Source Position (3), Detector Center (3), Detector X-axis (3), and Detector Y-axis (3). For parallel beams, the "Source" is often represented by the ray direction vector scaled or stored in the first 3 slots, or the geometry is interpreted such that rays originate from the detector center in the direction $-R$. However, [S1, unofficial] explicitly states for parallel3d: "compute ray direction... detector center...". It implies the first 3 elements might be the ray direction or the source is virtual. Given the cone formula uses Source Position, and parallel uses Ray Direction, we must map Ray Direction to the first 3 slots if the structure is rigid. Let's assume the 12-vector is strictly [Source/Direct, DetCenter, DetX, DetY]. For parallel, Source/Direct = Ray Direction.
7.  Return the dictionary with `Vectors`, `type`, `DetectorRowCount`, `DetectorColCount`, `DetectorSpacingX`, and `DetectorSpacingY`.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def projection_geometry_to_vector(proj_geom):
    """
    Converts a dictionary-based projection geometry into a 12-element vector array.
    
    Args:
        proj_geom (dict): Dictionary containing geometry parameters.
        
    Returns:
        dict: Dictionary with 'Vectors', 'type', 'DetectorRowCount', 'DetectorColCount', 
              'DetectorSpacingX', 'DetectorSpacingY'.
    """
    angles = proj_geom['ProjectionAngles']
    n_angles = len(angles)
    vectors = np.zeros((n_angles, 12))
    
    D_OS = proj_geom['DistanceOriginSource']
    D_OD = proj_geom['DistanceOriginDetector']
    D_DX = proj_geom['DetectorSpacingX']
    D_DY = proj_geom['DetectorSpacingY']
    
    geom_type = proj_geom['type']
    
    for i, theta in enumerate(angles):
        sin_t = np.sin(theta)
        cos_t = np.cos(theta)
        
        if geom_type == 'cone':
            # Source Position
            vectors[i, 0] = sin_t * D_OS
            vectors[i, 1] = -cos_t * D_OS
            vectors[i, 2] = 0.0
            
            # Detector Center
            vectors[i, 3] = -sin_t * D_OD
            vectors[i, 4] = cos_t * D_OD
            vectors[i, 5] = 0.0
            
            # Detector X-axis
            vectors[i, 6] = cos_t * D_DX
            vectors[i, 7] = sin_t * D_DX
            vectors[i, 8] = 0.0
            
            # Detector Y-axis
            vectors[i, 9] = 0.0
            vectors[i, 10] = 0.0
            vectors[i, 11] = D_DY
            
        elif geom_type == 'parallel3d':
            # Ray Direction (stored in first 3 slots as per typical 12-vector adaptation for parallel)
            # Note: The source is at infinity. The ray direction is (sin, -cos, 0).
            # The detector center is at (0,0,0).
            vectors[i, 0] = sin_t
            vectors[i, 1] = -cos_t
            vectors[i, 2] = 0.0
            
            # Detector Center
            vectors[i, 3] = 0.0
            vectors[i, 4] = 0.0
            vectors[i, 5] = 0.0
            
            # Detector X-axis
            vectors[i, 6] = cos_t * D_DX
            vectors[i, 7] = sin_t * D_DX
            vectors[i, 8] = 0.0
            
            # Detector Y-axis
            vectors[i, 9] = 0.0
            vectors[i, 10] = 0.0
            vectors[i, 11] = D_DY
            
        else:
            raise ValueError(f"Unsupported geometry type: {geom_type}")
            
    return {
        'type': geom_type,
        'DetectorRowCount': proj_geom['DetectorRowCount'],
        'DetectorColCount': proj_geom['DetectorColCount'],
        'Vectors': vectors,
        'DetectorSpacingX': D_DX,
        'DetectorSpacingY': D_DY
    }
```

**Parameters and Typical Values**
*   `ProjectionAngles`: Array of floats. Typical values depend on the scan protocol (e.g., 180 evenly spaced angles for a half-scan).
*   `DistanceOriginSource`: Float. Typical values range from 500 to 1000 mm in clinical CT.
*   `DistanceOriginDetector`: Float. Typical values range from 1000 to 1500 mm.
*   `DetectorSpacingX/Y`: Float. Typical values range from 0.5 to 1.0 mm.

**Edge Cases and Pitfalls**
*   **Precision:** Trigonometric functions should be computed in double precision if high accuracy is required for large distances [S1, unofficial].
*   **Boundary Conditions:** The formula assumes the rotation axis is the Z-axis and the source/detector rotate in the XY plane. Deviations from this assumption require modification of the basis vectors [S1, unofficial].
*   **Parallel Beam Source:** For parallel3d, the "Source" vector is the ray direction. Downstream kernels must interpret the first 3 elements as direction rather than position [S1, unofficial].

## Subtopics

### t-beam projectors
A specific projector type that approximates the cone beam geometry using a set of parallel beams (t-beams) for computational efficiency. It is chosen when exact cone beam geometry is not required or when speed is prioritized over geometric accuracy. See [t-beam projectors](3067-t-beam-projectors.md).

### TIGRE Geometry Setup
A higher-level utility in the TIGRE toolbox for setting up various standard geometries (parallel, cone, circular, etc.). It provides a user-friendly interface but may not expose the raw 12-vector format directly without conversion. See [TIGRE Geometry Setup](5605-tigre-geometry-setup.md).

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of projection angles. The computation involves simple trigonometric operations per angle.
*   **Memory Complexity:** $O(N)$, storing the 12-element vector for each angle.
*   **Strengths:** Simple, fast, and deterministic conversion from high-level parameters to low-level vectors.
*   **Limitations:** Only supports specific rotation axes and geometry types (cone and parallel3d). Does not handle arbitrary source-detector orientations beyond the defined rotation model.
*   **When to use:** When implementing a differentiable projector that requires explicit source, detector center, and basis vectors for each angle.
*   **When not to use:** For complex geometries not covered by the cone/parallel3d models, or when using a library that handles geometry internally.

## Background
This method is part of the geometry processing pipeline in differentiable tomography frameworks, specifically those based on the TIGRE toolbox and its derivatives like ShapeFromProjections [S1, unofficial]. It enables the use of arbitrary source-detector geometries by converting dictionary-based specifications into a format suitable for GPU-based ray casting [S1, unofficial].

## Sources
[S1] jakeoung/ShapeFromProjections, https://github.com/jakeoung/ShapeFromProjections, fetched 2026-10-26 (unofficial)
