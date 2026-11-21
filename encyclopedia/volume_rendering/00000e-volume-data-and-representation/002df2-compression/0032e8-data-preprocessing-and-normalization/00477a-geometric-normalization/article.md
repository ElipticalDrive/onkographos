# Geometric Normalization

Also called: Data Normalization, Viewport-Fitting Grid Normalization, Volume Centering and Scaling

## Overview
Geometric Normalization is a preprocessing step that translates and scales the input grid to prevent screen clipping during volume rendering [S1, unofficial][S2]. It centers the object at the origin and scales it to fit within a specific display resolution [S2]. It differs from [Volume Normalization](1823-volume-normalization.md), which typically adjusts scalar values (intensity/color) rather than spatial coordinates [G].

## How it works
**Inputs:** A set of 3D points or voxels with coordinates $(x, y, z)$ and associated data (e.g., RGB color) [S2].
**Outputs:** Transformed coordinates $(x', y', z')$ centered at the origin and scaled to fit a target viewport [S1, unofficial][S2].

**Data Structures:**
The input is typically stored as a list of tuples or arrays containing vertex coordinates and color data, e.g., $(x, y, z, r, g, b)$ [S2].

**Steps:**
1. **Compute Bounds:** Determine the minimum and maximum coordinates along each axis: $x_{min}, x_{max}, y_{min}, y_{max}, z_{min}, z_{max}$ [S2].
2. **Compute Center:** Calculate the center of the bounding box for each axis [S2]:
   $$x_{center} = \frac{x_{max} + x_{min}}{2}$$
   $$y_{center} = \frac{y_{max} + y_{min}}{2}$$
   $$z_{center} = \frac{z_{max} + z_{min}}{2}$$
3. **Translate:** Shift all coordinates to the origin [S2]:
   $$x' = x - x_{center}$$
   $$y' = y - y_{center}$$
   $$z' = z - z_{center}$$
4. **Scale:** Scale the translated grid so that it fits completely within a target screen resolution at any viewing angle [S1, unofficial]. The target resolution is typically 1024 x 768 pixels [S1, unofficial][S2]. The exact scaling formula for 3D-to-viewport projection is not specified in the sources; a standard uniform scale factor is applied [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def geometric_normalization(points, target_width=1024, target_height=768):
    """
    Normalize 3D points to fit a target screen resolution.
    points: list of tuples (x, y, z, r, g, b)
    """
    if not points:
        return []
    
    coords = [(p[0], p[1], p[2]) for p in points]
    xs = [c[0] for c in coords]
    ys = [c[1] for c in coords]
    zs = [c[2] for c in coords]
    
    x_min, x_max = min(xs), max(xs)
    y_min, y_max = min(ys), max(ys)
    z_min, z_max = min(zs), max(zs)
    
    x_center = (x_max + x_min) / 2.0
    y_center = (y_max + y_min) / 2.0
    z_center = (z_max + z_min) / 2.0
    
    width = x_max - x_min
    height = y_max - y_min
    depth = z_max - z_min
    max_dim = max(width, height, depth)
    
    # Guard against zero extent to avoid division by zero [G]
    if max_dim == 0:
        return [(p[0], p[1], p[2], p[3], p[4], p[5]) for p in points]
        
    # Exact scaling formula not specified in sources; standard uniform scale applied [G]
    scale_factor = min(target_width, target_height) / max_dim
    
    normalized_points = []
    for p in points:
        x, y, z, r, g, b = p
        x_norm = (x - x_center) * scale_factor
        y_norm = (y - y_center) * scale_factor
        z_norm = (z - z_center) * scale_factor
        normalized_points.append((x_norm, y_norm, z_norm, r, g, b))
        
    return normalized_points
```

**Parameters:**
- Target screen resolution: 1024 x 768 pixels [S1, unofficial][S2].
- Input data format: $(x, y, z, r, g, b)$ [S2].

**Edge Cases:**
- Zero-width/height/depth: If the bounding box has zero extent along any axis, scaling is undefined [G].
- Precision: Floating point precision issues may arise with very large coordinate ranges before normalization [G].

## Subtopics

### [Volume Normalization](1823-volume-normalization.md)
Normalizes scalar values (opacity/intensity) rather than geometry. Choose when data intensity ranges vary but geometry is fixed [G].

### [Volume Utility Functions](2677-volume-utility-functions.md)
General helper functions for volume data, including but not limited to normalization. Choose when reusable spatial or attribute transformations are needed [G].

### [Preprocessing Pipeline](6733-preprocessing-pipeline.md)
Orchestrates multiple preprocessing steps, including geometric normalization, into a single workflow. Choose when automating data preparation for rendering [G].

### [TetSplat Pre-processing and Data Layout](7221-tetsplat-pre-processing-and-data-layout.md)
Specialized preprocessing for tetrahedral splatting data structures, handling non-uniform spatial distributions. Choose when working with tetrahedral or unstructured volumetric meshes [G].

### [Dataset Preprocessing (DeepLesion)](7228-dataset-preprocessing-deeplesion.md)
Specific preprocessing pipeline for medical datasets, which may include geometric normalization. Choose when preparing clinical imaging data for standardized analysis [G].

## Operating characteristics
- **Time and memory complexity:** Not given in the sources [G].
- **Measured performance:** Not given in the sources [G].
- **Scaling:** Linear with respect to the number of input points or voxels [G].
- **Strengths:** Prevents clipping during rendering [S1, unofficial].
- **Limitations:** Assumes a rectangular bounding box [S2].
- **When to use:** When rendering arbitrary 3D grids that may exceed the viewport [S1, unofficial].

## Background
Originating from early volume visualization systems like VIPER, this method was developed to handle scattered data on rectangular grids [S1, unofficial]. It was later adopted in 3D swept-volume volumetric displays to fit display parameters [S2].

## Sources
[S1] DTIC ADA215533: Sampling Scattered Data Onto Rectangular Grids for Volume Visualization, https://archive.org/details/DTIC_ADA215533, fetched 2026-10-26 (unofficial)
[S2] PDF An Improved Design of 3D Swept-Volume Volumetric Display, https://jcomputers.us/vol9/jcp0901-34.pdf, fetched 2026-10-28
