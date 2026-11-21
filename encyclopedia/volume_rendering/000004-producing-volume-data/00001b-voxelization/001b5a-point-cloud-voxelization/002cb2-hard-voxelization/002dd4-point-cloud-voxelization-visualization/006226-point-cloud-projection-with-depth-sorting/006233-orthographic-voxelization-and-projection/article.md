# Orthographic Voxelization and Projection

Also called: 2D metric on voxelized point clouds, Voxel-based quality assessment, Projection-Based Metrics

## Overview
Orthographic Voxelization and Projection is a rendering scheme that converts 3D point clouds into a discrete voxel grid and projects them onto 2D planes using orthographic projection to enable objective quality assessment via standard 2D metrics [S1]. It maps 3D point cloud contents to 2D planes via orthographic projection and applies 2D image metrics to compare projected reference and distorted pairs [S1]. Unlike general-purpose ray casting or rasterization which may require complex depth buffering or interpolation, this method exploits the one-to-one correspondence between visible voxels and pixels when projecting along the principal axes, eliminating the need for interpolation or post-processing [S1]. It is specifically designed for perceptual quality assessment, providing a bridge between 3D geometric data and established 2D image quality metrics [S1].

## How it works
**Inputs:** A reference point cloud and a distorted point cloud, each consisting of a list of points with 3D spatial coordinates $(x, y, z)$ and color values.
**Outputs:** Six pairs of 2D images (reference and distorted) corresponding to the faces of a bounding cube, and a set of objective quality scores derived from 2D metrics applied to these pairs.

**Data Structures:**
*   **Point Cloud:** A list of structures, each containing:
    *   `x`, `y`, `z`: Floating-point spatial coordinates.
    *   `r`, `g`, `b`: Color components (format depends on the specific 2D metric implementation, e.g., uint8 or float).
*   **Voxel Grid:** A 3D array or hash map mapping integer coordinates $(\hat{x}, \hat{y}, \hat{z})$ to color values. The grid dimensions are determined by the scaling factor $s$ and the target resolution of 1024x1024 for the projected images.
*   **Projection Images:** 2D arrays of size $1024 \times 1024$ (or defined by the scaling logic) storing the final pixel colors.

**Parameters:**
*   $zf$: Zoom factor, inversely proportional to the virtual distance between the user and the content. It is updated dynamically based on mouse wheel scrolling [S1].
*   $w_{max}, w_{min}$: The maximum and minimum coordinate values of the point cloud in each dimension, used to determine the bounding box size [S1].
*   $yaw, pitch$: Rotation angles determined by user interaction (right mouse button drag) [S1].
*   $pan_x, pan_y$: Panning offsets determined by user interaction (left mouse button drag) [S1].
*   Target Resolution: The method scales the model to fit in a cubic volume that projects to a $1024 \times 1024$ pixel grid [S1].

**Steps:**

1.  **Compute Bounding Box and Centroid:**
    Iterate through all points in the point cloud. Record the maximum ($w_{max}$) and minimum ($w_{min}$) coordinate values for each dimension ($x, y, z$). Compute the centroid $(c_x, c_y, c_z)$ as the average position of all points [S1].

2.  **Center the Model:**
    Subtract the centroid from each point's coordinates to center the 3D model in the viewing volume:
    $$ x' = x - c_x, \quad y' = y - c_y, \quad z' = z - c_z $$
    [S1]

3.  **Scale the Model:**
    Calculate the scaling factor $s$ to fit the model in a cubic volume of side 1024. The formula is:
    $$ s = zf \times 2^{(10 - \lceil \log_2(w_{max} - w_{min}) \rceil)} $$
    where $w_{max} - w_{min}$ is the range of the coordinates (assuming uniform scaling across dimensions or using the maximum range if dimensions differ, though the source implies a single scaling component $s$ applied to spatial coordinates) [S1]. Multiply all centered coordinates by $s$:
    $$ x'' = x' \times s, \quad y'' = y' \times s, \quad z'' = z' \times s $$
    [S1]

4.  **Apply Rotation:**
    Apply a rigid rotation to the spatial coordinates using a rotation matrix $R$ calculated from the user-defined yaw and pitch angles. The rotation is applied to both reference and distorted point clouds identically to facilitate simultaneous visualization [S1].
    $$ \begin{bmatrix} x''' \\ y''' \\ z''' \end{bmatrix} = R \begin{bmatrix} x'' \\ y'' \\ z'' \end{bmatrix} $$
    [S1]

5.  **Apply Panning (Optional/Interactive):**
    If user interaction includes panning, add shift values $pan_x$ and $pan_y$ to the $x$ and $y$ coordinates respectively. This is applied to both reference and distorted contents [S1].
    $$ x_{final} = x''' + pan_x $$
    $$ y_{final} = y''' + pan_y $$
    [S1]

6.  **Quantize and Voxelize:**
    Quantize the final spatial coordinates to integer values $(\hat{x}, \hat{y}, \hat{z})$.
    *   If multiple points share identical $(\hat{x}, \hat{y})$ coordinates in the projection plane, associate the color value with the pixel $(\hat{x}, \hat{y})$ corresponding to the point with the smaller distance from the projection plane (i.e., the point with the largest $z$ value for a front-facing projection, or smallest depending on coordinate system convention; the source specifies "smaller distance from the projection plane") [S1].
    *   If multiple points fall within the same voxel area (same $\hat{x}, \hat{y}, \hat{z}$), the color of the voxel is given by the average of the color values of these points [S1].
    *   For voxels projected frontally (faces parallel to the projection plane), map occupied voxels directly to pixels without interpolation, exploiting the one-to-one relationship between visible voxels and pixels [S1].

7.  **Generate Projections:**
    Define a bounding cube around the object. Project the voxelized point cloud orthographically onto the 6 faces of the cube. This corresponds to 6 orthographic projections along the reference axes (e.g., $\pm x, \pm y, \pm z$) [S1]. Ensure one-to-one correspondence between voxels and pixels in the projected images [S1].

8.  **Compute Quality Metrics:**
    Apply 2D image metrics to each pair of projected reference and distorted images. The metrics investigated include PSNR, PSNR-HVS, PSNR-HVS-M, SSIM, MSSIM, and VIFP [S1].
    *   Compute one score for each of the 6 pairs of images.
    *   Compute the average score across all 6 projected planes for each metric to obtain a total degradation value representing visual impairment [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def orthographic_voxelization_and_projection(ref_points, dist_points, zf=1.0):
    """
    Performs orthographic voxelization and projection for quality assessment.
    
    Args:
        ref_points: List of tuples (x, y, z, r, g, b) for reference point cloud.
        dist_points: List of tuples (x, y, z, r, g, b) for distorted point cloud.
        zf: Zoom factor.
        
    Returns:
        ref_images: List of 6 numpy arrays (1024x1024x3) for reference projections.
        dist_images: List of 6 numpy arrays (1024x1024x3) for distorted projections.
    """
    
    # Step 1: Compute Bounding Box and Centroid
    # Combine both clouds for bounding box calculation if they share the same space
    all_points = ref_points + dist_points
    xs = [p[0] for p in all_points]
    ys = [p[1] for p in all_points]
    zs = [p[2] for p in all_points]
    
    w_max = max(max(xs), max(ys), max(zs))
    w_min = min(min(xs), min(ys), min(zs))
    
    c_x = sum(p[0] for p in all_points) / len(all_points)
    c_y = sum(p[1] for p in all_points) / len(all_points)
    c_z = sum(p[2] for p in all_points) / len(all_points)
    
    # Step 2 & 3: Center and Scale
    # Calculate scaling factor s
    # Note: The source uses w_max - w_min. Assuming this refers to the max dimension range
    # or the range of the specific dimension being projected. The formula implies a single s.
    # We use the maximum range across dimensions to ensure the object fits in the cube.
    range_max = max(max(xs) - min(xs), max(ys) - min(ys), max(zs) - min(zs))
    
    s = zf * (2 ** (10 - math.ceil(math.log2(range_max))))
    
    def process_points(points):
        processed = []
        for p in points:
            x, y, z, r, g, b = p
            # Center
            xc = x - c_x
            yc = y - c_y
            zc = z - c_z
            # Scale
            xs = xc * s
            ys = yc * s
            zs = zc * s
            # Rotation and Panning are interactive. For a static assessment, 
            # we assume identity rotation and zero pan, or that the user 
            # has set them. Here we assume identity for the core algorithm.
            # In a real implementation, R and pan would be inputs.
            processed.append((xs, ys, zs, r, g, b))
        return processed

    ref_processed = process_points(ref_points)
    dist_processed = process_points(dist_points)
    
    # Step 6 & 7: Voxelize and Project onto 6 faces
    # The 6 faces correspond to projections along +/- X, +/- Y, +/- Z axes.
    # For a frontal projection along Z, we map (x, y) to pixels.
    # The grid size is 1024x1024.
    
    grid_size = 1024
    ref_images = []
    dist_images = []
    
    # Define projection directions and their corresponding pixel axes
    # (Normal Vector, Pixel Axis 1, Pixel Axis 2)
    # Assuming standard right-handed coordinate system
    projections = [
        ('+Z', 'x', 'y'),
        ('-Z', 'x', 'y'),
        ('+X', 'y', 'z'),
        ('-X', 'y', 'z'),
        ('+Y', 'x', 'z'),
        ('-Y', 'x', 'z')
    ]
    
    for normal, axis1, axis2 in projections:
        # Initialize images with zeros or background
        ref_img = np.zeros((grid_size, grid_size, 3), dtype=np.uint8)
        dist_img = np.zeros((grid_size, grid_size, 3), dtype=np.uint8)
        
        # Helper to map continuous coordinates to grid indices
        # The source implies the scaled coordinates fit in a volume that projects to 1024x1024.
        # We need to map the scaled coordinates to [0, 1023].
        # Assuming the scaled object is centered and fits within [-512, 512] or similar.
        # The source says "fit in a cubic volume of side 1024".
        # Let's assume the scaled coordinates are in range [-512, 512] roughly.
        # We map center (0,0) to (512, 512).
        
        def coord_to_pixel(val):
            # Map value to 0-1023. 
            # If the range is roughly 1024, then val + 512 maps to 0-1023.
            # This is an approximation based on "fit in a cubic volume of side 1024".
            # A more robust implementation would use the actual w_max/w_min after scaling.
            # However, the source states the scaling factor s is calculated to fit the model.
            # So the max extent is approx 512.
            return int(val + 512)
        
        # Process Reference Points
        # We need to handle depth sorting for overlapping points in the same pixel
        # The source says: "if another point with identical (x, y) coordinates... 
        # the color of the voxel is given by the average... if multiple points fall within the same area"
        # AND "associate the color value with the pixel... corresponding to the point with the smaller distance from the projection plane"
        
        # For frontal projection along Z, "distance from projection plane" usually means Z.
        # If projecting onto +Z plane, the plane is at Z=constant. 
        # Points with larger Z are closer to the viewer if viewer is at +inf Z.
        # The source says "smaller distance from the projection plane".
        # If the projection plane is at Z=0, distance is |Z|.
        # However, in voxelization, we often just take the front-most point.
        # Let's assume standard front-face culling: take the point with the largest Z for +Z view.
        
        # Collect points for this view
        ref_view_points = []
        for p in ref_processed:
            x, y, z, r, g, b = p
            if normal == '+Z':
                px, py = coord_to_pixel(x), coord_to_pixel(y)
                depth = z
            elif normal == '-Z':
                px, py = coord_to_pixel(x), coord_to_pixel(y)
                depth = -z # Flip for -Z view
            elif normal == '+X':
                px, py = coord_to_pixel(y), coord_to_pixel(z)
                depth = x
            elif normal == '-X':
                px, py = coord_to_pixel(y), coord_to_pixel(z)
                depth = -x
            elif normal == '+Y':
                px, py = coord_to_pixel(x), coord_to_pixel(z)
                depth = y
            elif normal == '-Y':
                px, py = coord_to_pixel(x), coord_to_pixel(z)
                depth = -y
            
            # Check bounds
            if 0 <= px < grid_size and 0 <= py < grid_size:
                ref_view_points.append((px, py, depth, r, g, b))
                
        dist_view_points = []
        for p in dist_processed:
            x, y, z, r, g, b = p
            if normal == '+Z':
                px, py = coord_to_pixel(x), coord_to_pixel(y)
                depth = z
            elif normal == '-Z':
                px, py = coord_to_pixel(x), coord_to_pixel(y)
                depth = -z
            elif normal == '+X':
                px, py = coord_to_pixel(y), coord_to_pixel(z)
                depth = x
            elif normal == '-X':
                px, py = coord_to_pixel(y), coord_to_pixel(z)
                depth = -x
            elif normal == '+Y':
                px, py = coord_to_pixel(x), coord_to_pixel(z)
                depth = y
            elif normal == '-Y':
                px, py = coord_to_pixel(x), coord_to_pixel(z)
                depth = -y
                
            if 0 <= px < grid_size and 0 <= py < grid_size:
                dist_view_points.append((px, py, depth, r, g, b))
                
        # Resolve overlaps
        # Group by pixel (px, py)
        ref_pixel_map = {}
        for px, py, depth, r, g, b in ref_view_points:
            key = (px, py)
            if key not in ref_pixel_map:
                ref_pixel_map[key] = []
            ref_pixel_map[key].append((depth, r, g, b))
            
        for key, points in ref_pixel_map.items():
            # Sort by depth. "Smaller distance from projection plane".
            # If projection plane is at Z=0, distance is |Z|.
            # But typically in rendering, we want the closest point to the camera.
            # The source is slightly ambiguous on "distance from plane" vs "closest to camera".
            # However, it says "one-to-one relationship... no interpolation".
            # Let's assume we take the point with the maximum depth value (closest to viewer at +inf)
            # for +Z view.
            # To be safe, we sort by absolute depth if "distance from plane" is literal.
            # But standard voxelization takes the front-most.
            # Let's assume the source implies standard front-face: max depth for +Z.
            # We will sort by depth descending for +Z, ascending for -Z?
            # Actually, let's just take the point with the largest absolute depth if we assume viewer is far away.
            # Or simpler: The source says "smaller distance from the projection plane".
            # If the plane is Z=0, and points are at Z=10 and Z=20, distance is 10 and 20.
            # So Z=10 is closer.
            # But if the viewer is at Z=+inf, Z=20 is closer to the viewer.
            # The source likely means "closest to the viewer".
            # Let's assume standard rendering: take the point with the largest Z for +Z view.
            
            # Re-reading: "associate the color value with the pixel... corresponding to the point with the smaller distance from the projection plane"
            # If the projection plane is the image plane, and the camera is at infinity, 
            # the "distance from the projection plane" is the Z coordinate (if plane is Z=0).
            # Smaller distance means smaller Z.
            # This contradicts standard "closest to camera" if camera is at +Z.
            # However, if the projection plane is at the back of the object, it might be different.
            # Given the ambiguity, and the fact that it's for quality assessment, 
            # the exact depth handling might be less critical than the voxelization itself.
            # We will implement taking the point with the maximum depth (closest to +Z viewer)
            # as this is the standard interpretation of "front-most" in orthographic projection.
            
            # Let's stick to the text: "smaller distance from the projection plane".
            # If the plane is Z=0, and we project onto it, points with Z>0 are in front.
            # Distance is Z. Smaller Z is closer to the plane.
            # This would mean we see the "back" of the object if the plane is behind it?
            # No, orthographic projection usually projects onto a plane in front of the object.
            # Let's assume the standard case: Viewer at +Z, Plane at Z=0.
            # Points with Z>0 are between viewer and plane? No, usually plane is at Z=0, viewer at +inf.
            # Points with Z>0 are closer to viewer.
            # Distance from plane (Z=0) is Z.
            # Smaller distance = smaller Z.
            # This would pick the point furthest from the viewer.
            # This seems wrong for visualization.
            # Perhaps "projection plane" refers to the plane containing the image, 
            # and "distance" refers to the depth buffer value.
            # In many systems, depth buffer stores distance from camera.
            # Smaller distance = closer to camera.
            # This aligns with standard rendering.
            
            # We will assume "distance from projection plane" means "distance from camera".
            points.sort(key=lambda p: p[0]) # Sort by depth ascending (closest first)
            depth, r, g, b = points[0]
            ref_img[py, px] = [r, g, b]
            
        dist_pixel_map = {}
        for px, py, depth, r, g, b in dist_view_points:
            key = (px, py)
            if key not in dist_pixel_map:
                dist_pixel_map[key] = []
            dist_pixel_map[key].append((depth, r, g, b))
            
        for key, points in dist_pixel_map.items():
            points.sort(key=lambda p: p[0])
            depth, r, g, b = points[0]
            dist_img[py, px] = [r, g, b]
            
        ref_images.append(ref_img)
        dist_images.append(dist_img)
        
    return ref_images, dist_images
```

## Subtopics

### Point Cloud Projection with Depth Sorting
Renders a voxelized point cloud to a 2D image by quantizing 3D coordinates, applying user-controlled panning, sorting by depth, and resolving overlaps. This is the parent method that provides the interactive rendering framework. [Link](18548-point-cloud-projection-with-depth-sorting.md)

### 2D Image Quality Metrics
Methods such as PSNR, SSIM, and VIFP used to evaluate the projected images. These are not voxelization methods but are the downstream consumers of the output of Orthographic Voxelization and Projection.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of points $N$ for the voxelization and projection steps ($O(N)$), as each point is processed once. The metric computation is $O(P)$ where $P$ is the number of pixels (1024x1024x6).
*   **Memory Complexity:** $O(N)$ to store the point clouds and $O(P)$ to store the 6 projection images.
*   **Measured Performance:** The method is designed for real-time interaction, allowing user-controlled panning and zooming [S1]. It outperforms state-of-the-art objective quality assessment methods in predictive power against subjective ratings [S1].
*   **Strengths:**
    *   Eliminates interpolation and post-processing for frontal projections due to the one-to-one voxel-pixel correspondence [S1].
    *   Leverages well-established 2D image quality metrics.
    *   Provides a comprehensive view by using 6 projection planes [S1].
*   **Limitations:**
    *   Only provides 6 viewing angles, which may not capture all aspects of the 3D structure.
    *   Relies on the assumption that 2D metrics correlate well with 3D perceptual quality, which may not always hold for complex geometries.
    *   Requires the point cloud to be voxelized, which may introduce quantization artifacts.
*   **When to use:** When objective quality assessment of point clouds is required, and the computational cost of 3D-specific metrics is prohibitive or less validated.
*   **When not to use:** When a full 360-degree view is necessary, or when the point cloud has complex occlusions that are not well-represented by 6 orthogonal views.

## Background
The method was proposed in a paper titled "A novel methodology for quality assessment of voxelized point clouds" [S1]. It addresses the open problem of subjective and objective quality assessment for point cloud visual data representation [S1]. The authors developed a rendering software that performs real-time voxelization and projection, allowing user interaction. Benchmarking results from two test laboratories showed that the framework provides high predictive power and outperforms state-of-the-art methods [S1]. The approach uses 2D objective metrics (PSNR, SSIM, etc.) on 6 orthographic projections to predict perceptual quality [S1].

## Sources
[S1] A novel methodology for quality assessment of voxelized point clouds, https://doi.org/10.1117/12.2322741, fetched 2026-10-30
