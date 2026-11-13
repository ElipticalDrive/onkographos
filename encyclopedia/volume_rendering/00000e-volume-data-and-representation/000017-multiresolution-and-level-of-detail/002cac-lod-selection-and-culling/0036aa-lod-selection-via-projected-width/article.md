# LoD Selection via Projected Width

Also called: PathTracer MipMap selection

## Overview
This method selects the appropriate resolution level for volume rendering by estimating the projected width of the data on the camera plane, analogous to Mipmap selection in path tracing [S1]. It solves the problem of aliasing and performance inefficiency by ensuring that the sampling rate matches the visual importance of the geometry relative to the camera [S1]. Unlike distance-based selection, which assumes uniform scaling, this approach accounts for perspective distortion and viewing angle, making it particularly effective for complex microgeometry like vegetation and hair [S1].

## How it works
**Inputs:**
*   Camera parameters: position, orientation, field of view, and image resolution.
*   Volume hierarchy: A set of pre-computed LoD volumes (one per level) [S1].
*   Geometry bounds: The spatial extent of the volume data.

**Outputs:**
*   A selected LoD index for each voxel or sample point during rendering [S1].

**Data Structures:**
*   **LoD Volumes:** A list of volume datasets, where each dataset corresponds to a specific resolution level. The number of levels is determined by the user based on memory constraints and input sparsity [S1].
*   **Voxel Grid:** The fundamental unit of lookup. Each voxel maps to a specific LoD volume based on the calculated metric [S1].

**Steps:**
1.  **Load Volumes:** Load all required LoD volumes into memory simultaneously, one per level [S1].
2.  **Calculate Projected Width:** For each voxel (or representative point of the voxel), compute its projected size on the image plane. This involves transforming the voxel's world-space position to camera space and projecting it using the camera's intrinsic parameters.
3.  **Determine LoD Level:** Compare the projected width against a threshold or use a logarithmic mapping to select the appropriate LoD level. The selection follows the "usual PathTracer MipMap selection" logic, which typically aims to keep the projected size within a specific range relative to the pixel size [S1].
4.  **Lookup Data:** During rendering, for each sample, use the calculated LoD level to determine which volume dataset to query for the density or color value [S1].

**Formula:**
Let $P$ be the projected width of a voxel on the image plane. Let $L$ be the selected LoD level. The selection is based on the metric:
$$ L = f(P) $$
where $f$ is the Mipmap selection function [S1]. The exact mathematical form of $f$ is not detailed in the source, but it relies on the projected width on the camera [S1].

**Parameters:**
*   **First Level Resolution:** Controls the finest detail [S1].
*   **Second Level Resolution:** Controls the coarser detail [S1].
*   **Sample Number:** Affects memory usage and quality [S1].

**Edge Cases:**
*   **Memory Limits:** Combining high resolution with a high number of samples can lead to out-of-memory errors [S1]. Users must balance these parameters based on available GPU memory [S1].
*   **Sparsity:** The resolution levels should be adjusted to match the sparsity of the input model [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def select_lod_level(voxel_center, camera, image_width, image_height, fov):
    """
    Selects the LoD level for a voxel based on its projected width.
    Note: The exact Mipmap selection function is not specified in [S1].
    This pseudocode assumes a standard logarithmic mapping based on projected size.
    """
    # Transform voxel center to camera space
    voxel_cam = camera.world_to_camera(voxel_center)
    
    # Project to normalized device coordinates (NDC)
    # Assuming pinhole camera model
    fx = image_width / (2.0 * tan(fov / 2.0))
    fy = image_height / (2.0 * tan(fov / 2.0))
    
    ndc_x = voxel_cam.x * fx / voxel_cam.z
    ndc_y = voxel_cam.y * fy / voxel_cam.z
    
    # Calculate projected size in pixels (approximate)
    # This requires knowing the voxel's physical size in world space
    voxel_size = get_voxel_size() 
    projected_size_x = (voxel_size * fx / voxel_cam.z) / image_width * image_width
    projected_size_y = (voxel_size * fy / voxel_cam.z) / image_height * image_height
    
    # Average projected size in pixels
    projected_pixels = (projected_size_x + projected_size_y) / 2.0
    
    # Select LoD level based on projected pixels
    # Standard Mipmap selection: level = log2(projected_pixels)
    # This is a placeholder for the "usual PathTracer MipMap selection" [S1]
    # The exact thresholding or mapping function is not provided in [S1]
    if projected_pixels < 1.0:
        return 0 # Finest level
    elif projected_pixels < 2.0:
        return 1
    else:
        return 2 # Coarsest level (example)

def render_with_lod(camera, volumes, voxels):
    """
    Renders the volume using the selected LoD for each voxel.
    """
    for voxel in voxels:
        lod_level = select_lod_level(voxel.center, camera, ...)
        # Lookup data from the specific volume level [S1]
        data = volumes[lod_level].lookup(voxel.center)
        # Proceed with ray marching or voxel traversal using 'data'
```

## Subtopics

### [Distance-Based Level Selection](14162-distance-based-level-selection.md)
A mechanism that selects the octree resolution level for each anchor point based on its Euclidean distance from the camera, ensuring closer areas use higher detail and distant areas use simplified representations.

## Operating characteristics
*   **Complexity:** The selection process adds a per-voxel or per-sample calculation overhead based on camera projection [S1].
*   **Memory:** Requires loading multiple volume levels simultaneously, which increases memory usage [S1]. The number of levels and their resolutions must be tuned to fit within GPU memory limits [S1].
*   **Performance:** Improves rendering performance for distant or small-scale features by using coarser resolutions, while maintaining quality for prominent features [S1].
*   **Limitations:** The pipeline exposes three main parameters (first level resolution, second level resolution, sample number) that require user tuning to balance quality and memory [S1].

## Background
This method is described in the context of fast voxelization and level of detail for microgeometry rendering, specifically applied to vegetation and hair models [S1]. It leverages established Mipmap selection techniques from path tracing [S1].

## Sources
[S1] Fast Voxelization and Level of Detail for Microgeometry Rendering, https://doi.org/10.48550/arxiv.2604.13191, fetched 2026-10-26
