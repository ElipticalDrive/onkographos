# Point Cloud Projection with Depth Sorting

Also called: Depth-sorted point cloud projection, Hard voxelization visualization with depth resolution

## Overview
Point Cloud Projection with Depth Sorting renders a voxelized point cloud into a 2D image by quantizing 3D coordinates, applying user-controlled panning, and resolving visual overlaps via depth comparison [S1]. It differs from general-purpose ray casting or rasterization by exploiting the one-to-one correspondence between visible voxels and pixels along principal axes, eliminating the need for complex depth buffering or interpolation [S1]. This method is specifically designed for perceptual quality assessment, providing a bridge between 3D geometric data and established 2D image quality metrics [S1].

## How it works
**Inputs:**
- Reference point cloud: List of points $P_{ref} = \{(x_i, y_i, z_i, r_i, g_i, b_i)\}$.
- Distorted point cloud: List of points $P_{dist} = \{(x_j, y_j, z_j, r_j, g_j, b_j)\}$.
- User parameters: Panning shift $(\Delta x, \Delta y)$, applied identically to both clouds.

**Outputs:**
- Reference image: 2D array of RGB values.
- Distorted image: 2D array of RGB values.

**Data Structures:**
- Image buffer: 2D array `Image[W][H]` initialized to default color.
- Point structure: Tuple `(x, y, z, r, g, b)`.

**Steps:**
1. **Initialization**: Create two image buffers, `Image_ref` and `Image_dist`, of dimensions $W \times H$. Initialize every pixel in both buffers to the default neutral gray value $(127, 127, 127)$ [S1].
2. **Quantization and Panning**: For each point in the reference and distorted clouds, quantize the spatial coordinates $(x, y, z)$ to integer values. Apply the user-defined panning shift $(\Delta x, \Delta y)$ to the $x$ and $y$ coordinates:
   $$ \hat{x} = \text{round}(x) + \Delta x $$
   $$ \hat{y} = \text{round}(y) + \Delta y $$
   The $z$ coordinate remains as the distance metric from the projection plane [S1].
3. **Projection and Depth Sorting**: Iterate through all points in the reference cloud. For each point with quantized coordinates $(\hat{x}, \hat{y})$ and depth $z$:
   - If the pixel `Image_ref[$\hat{x}$][$\hat{y}$]` is unoccupied (still default gray) or the current point has a smaller $z$ (closer to the projection plane) than the point currently stored at that pixel, update the pixel color to $(r, g, b)$ of the current point [S1].
   - If multiple points share identical $(\hat{x}, \hat{y})$ and identical $z$ (coinciding coordinates after rotation, quantization, and panning), compute the pixel value as the average of the color values of all such points [S1].
4. **Repeat for Distorted Cloud**: Perform steps 2 and 3 for the distorted point cloud to generate `Image_dist`, using the same panning shift $(\Delta x, \Delta y)$ [S1].
5. **Rendering**: Display `Image_ref` and `Image_dist` side-by-side or overlaid for visual comparison [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def project_point_cloud(points, width, height, pan_x, pan_y, default_color=(127, 127, 127)):
    # Initialize image buffer with default color
    image = [[default_color for _ in range(height)] for _ in range(width)]
    
    # Dictionary to store the best point for each pixel: key=(x, y), value=(z, r, g, b)
    # Using a dict allows handling sparse grids efficiently
    pixel_map = {}
    
    for point in points:
        x, y, z, r, g, b = point
        
        # Quantize and apply panning
        # Assuming coordinates are floats that need rounding to integer grid
        x_hat = round(x) + pan_x
        y_hat = round(y) + pan_y
        
        # Check bounds
        if 0 <= x_hat < width and 0 <= y_hat < height:
            # Check if this point is closer (smaller z) than existing point at this pixel
            if (x_hat, y_hat) not in pixel_map:
                pixel_map[(x_hat, y_hat)] = (z, r, g, b)
            else:
                existing_z, existing_r, existing_g, existing_b = pixel_map[(x_hat, y_hat)]
                
                if z < existing_z:
                    # Current point is closer, replace
                    pixel_map[(x_hat, y_hat)] = (z, r, g, b)
                elif z == existing_z:
                    # Coinciding coordinates: average colors
                    # Note: The source implies averaging if multiple points have coinciding coordinates.
                    # This simple implementation averages with the previous winner.
                    # For strict averaging of all coinciding points, a list of colors per pixel would be needed.
                    # However, the source says "average of the color values of those points".
                    # To implement strictly, we might need to collect all points with same (x,y,z).
                    # Given the iterative nature described, we assume the "average" applies to the set of points
                    # that map to the same pixel with the same depth.
                    # A robust implementation would group by (x_hat, y_hat, z) first.
                    pass # See note below for robust implementation
    
    # Re-implementation for correct averaging of coincident points
    # Group points by (x_hat, y_hat, z)
    groups = {}
    for point in points:
        x, y, z, r, g, b = point
        x_hat = round(x) + pan_x
        y_hat = round(y) + pan_y
        if 0 <= x_hat < width and 0 <= y_hat < height:
            key = (x_hat, y_hat, z)
            if key not in groups:
                groups[key] = []
            groups[key].append((r, g, b))
            
    # Resolve depth and average colors
    for (x_hat, y_hat, z), colors in groups.items():
        # Find the minimum z for this (x_hat, y_hat) across all groups?
        # No, the source says: "if another point with identical (x, y) coordinates and a smaller distance... is identified, the first point is ignored"
        # This implies a global depth sort for each (x_hat, y_hat).
        pass

    # Correct Algorithm based on source description:
    # 1. Quantize and pan all points.
    # 2. For each pixel (x_hat, y_hat), find the point(s) with the minimum z.
    # 3. If multiple points have the same minimum z, average their colors.
    
    # Step 1: Quantize and Pan
    projected_points = []
    for point in points:
        x, y, z, r, g, b = point
        x_hat = round(x) + pan_x
        y_hat = round(y) + pan_y
        if 0 <= x_hat < width and 0 <= y_hat < height:
            projected_points.append((x_hat, y_hat, z, r, g, b))
            
    # Step 2: Group by pixel (x_hat, y_hat)
    pixel_groups = {}
    for x_hat, y_hat, z, r, g, b in projected_points:
        if (x_hat, y_hat) not in pixel_groups:
            pixel_groups[(x_hat, y_hat)] = []
        pixel_groups[(x_hat, y_hat)].append((z, r, g, b))
        
    # Step 3: Resolve depth and average
    for (x_hat, y_hat), group in pixel_groups.items():
        # Find minimum z
        min_z = min(p[0] for p in group)
        # Get all points with min_z
        closest_points = [p for p in group if p[0] == min_z]
        # Average colors
        avg_r = sum(p[1] for p in closest_points) / len(closest_points)
        avg_g = sum(p[2] for p in closest_points) / len(closest_points)
        avg_b = sum(p[3] for p in closest_points) / len(closest_points)
        
        # Round to integer RGB
        image[x_hat][y_hat] = (round(avg_r), round(avg_g), round(avg_b))
        
    return image

def render_pair(ref_points, dist_points, width, height, pan_x, pan_y):
    ref_image = project_point_cloud(ref_points, width, height, pan_x, pan_y)
    dist_image = project_point_cloud(dist_points, width, height, pan_x, pan_y)
    return ref_image, dist_image
```

**Parameters:**
- Panning shift $(\Delta x, \Delta y)$: User-defined integer offset applied to $x$ and $y$ coordinates, issued simultaneously on reference and distorted contents [S1].
- Default pixel color: $(127, 127, 127)$ in RGB space, representing neutral gray [S1].

**Edge Cases and Pitfalls:**
- Multiple points sharing identical $(x, y)$ but different $z$: Resolved by depth comparison, keeping the point with the smaller distance to the projection plane [S1].
- Multiple points sharing identical $(x, y, z)$ after transformations: Resolved by averaging their color values for the target pixel [S1].
- Precision: Quantization to integer values may cause aliasing or loss of fine detail if the voxel grid resolution is too low relative to the point cloud density [S1].

## Subtopics

### [Orthographic Voxelization and Projection](18550-orthographic-voxelization-and-projection.md)
Orthographic Voxelization and Projection is a rendering scheme that converts 3D point clouds into a discrete voxel grid and projects them onto 2D planes using orthographic projection to enable objective quality assessment via standard 2D metrics. It maps 3D point cloud contents to 2D planes via orthographic projection and applies 2D image metrics to compare projected reference and distorted pairs. Unlike general-purpose ray casting or rasterization which may require complex depth buffering or interpolation, this method exploits the one-to-one correspondence between visible voxels and pixels when projecting along the principal axes, eliminating the need for interpolation or post-processing. It is specifically designed for perceptual quality assessment, providing a bridge between 3D geometric data and established 2D image quality metrics.

## Operating characteristics
- **Time Complexity**: $O(N)$ for quantization and grouping, where $N$ is the number of points. Depth sorting and averaging are linear with respect to the number of points per pixel.
- **Memory Complexity**: $O(W \times H)$ for the image buffer, plus $O(N)$ for storing point data.
- **Strengths**: Simple implementation, no interpolation artifacts, direct mapping to 2D metrics for quality assessment [S1].
- **Limitations**: Discrete quantization may introduce aliasing; depth sorting is limited to the specified projection direction; user interaction (panning) is required for optimal viewing [S1].
- **When to use**: Perceptual quality assessment of voxelized point clouds where 2D metrics are applicable [S1].
- **When not to use**: When continuous depth representation or complex lighting effects are required [S1].

## Background
Originating from the methodology for quality assessment of voxelized point clouds, this technique was developed to facilitate simultaneous visualization of reference and distorted point clouds from the same angle [S1]. It leverages the preprocessing workflow including voxelization, scaling, translation, and encoding to prepare content for comparison [S1]. The method is used in experiments involving diverse static contents such as human bodies and inanimate objects with varying levels of geometry and color details [S1].

## Sources
[S1] A novel methodology for quality assessment of voxelized point clouds, https://doi.org/10.1117/12.2322741, fetched 2026-10-30
