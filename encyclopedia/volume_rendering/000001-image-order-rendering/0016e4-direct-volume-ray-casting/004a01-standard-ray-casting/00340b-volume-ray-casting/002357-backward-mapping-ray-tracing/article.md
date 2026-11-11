# Backward Mapping (Ray Tracing)

Also called: Image-order volume rendering, Ray-tracing volume rendering, Volume Rendering (General)

## Overview
Backward mapping is a volume rendering technique that uses computer graphics ray-tracing to form the image by projecting a ray from each pixel through the data volume, where intersecting voxels influence the pixel value [S2, unofficial]. It is an image-space method that considers all voxels in an object for one view, producing 'fuzzy' images from sampled data rather than 'crisp' surface renderings [S1, unofficial]. This approach helps overcome misinterpretations (false negatives/positives) inherent in surface renderings, albeit at a slower speed [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A volumetric dataset (voxels), a camera/viewing parameters defining the image plane and ray origins/directions.
- Output: A 2D image where each pixel value is accumulated from the voxels intersected by the corresponding ray.

**Data Structures:**
- The volume is stored in its natural voxel format, intrinsically spatially presorted [S2, unofficial].
- No explicit geometric representations (surfaces) are used [S1, unofficial].

**Steps:**
1. For each pixel in the image plane, define a ray originating from the camera center and passing through the pixel [S2, unofficial].
2. Project this ray through the data volume [S2, unofficial].
3. Identify all voxels intersected by the ray [S2, unofficial].
4. Accumulate the contribution of each intersecting voxel to the pixel value [S2, unofficial].
5. The accumulation can be achieved using either "front-to-back" or "back-to-front" ordering [S1, unofficial].

**Formulas:**
Not given in the sources.

**Parameters:**
Not given in the sources.

**Edge Cases and Pitfalls:**
- The rendering is created from sampled 'fuzzy' data, which requires the user to remember this nature when interpreting the image [S1, unofficial].
- The method is slower than surface type renderings [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def backward_mapping_volume_rendering(volume, camera, image_width, image_height):
    """
    Renders a volume using backward mapping (ray tracing).
    
    Args:
        volume: 3D array of voxel values.
        camera: Object containing camera position and orientation.
        image_width: Width of the output image.
        image_height: Height of the output image.
        
    Returns:
        2D array representing the rendered image.
    """
    image = zeros((image_height, image_width))
    
    for y in range(image_height):
        for x in range(image_width):
            # Define ray for current pixel
            ray_origin = camera.position
            ray_direction = compute_ray_direction(camera, x, y, image_width, image_height)
            
            # Initialize pixel value
            pixel_value = 0.0
            
            # Trace ray through volume
            # Determine intersection with volume bounds
            t_min, t_max = compute_ray_volume_intersection(ray_origin, ray_direction, volume_bounds)
            
            # Sample along the ray
            t = t_min
            while t <= t_max:
                # Get sample point in world space
                sample_point = ray_origin + t * ray_direction
                
                # Sample voxel value at sample_point
                # This involves trilinear interpolation or similar resampling
                voxel_value = sample_volume(volume, sample_point)
                
                # Accumulate contribution
                # Specific accumulation function (e.g., front-to-back compositing) is not detailed in sources
                pixel_value = accumulate(pixel_value, voxel_value)
                
                # Advance along ray
                t += step_size
                
            image[y, x] = pixel_value
            
    return image
```

## Operating characteristics
- **Speed:** Slower than surface type renderings [S1, unofficial].
- **Quality:** Produces higher quality images than surface renderings [S1, unofficial].
- **Data Handling:** Uses no geometric representations; considers all voxels [S1, unofficial].
- **Interpretation:** Produces 'fuzzy' images from sampled data, reducing misinterpretations compared to 'crisp' surface renderings [S1, unofficial].

## Background
Volume rendering was developed as an alternative to surface rendering to avoid the ambiguities and misinterpretations (false negatives/positives) that can arise when creating surfaces from neighbor data points [S1, unofficial]. The technique is divided into two main subsections: Backward projection methods (image space/ray casting) and Forward projection methods (object space) [S1, unofficial]. Backward mapping is specifically identified as an image-space method using ray-tracing [S1, S2, unofficial]. The feasibility of volume rendering relies on the intrinsic spatial presorting of volumetric data sets [S2, unofficial].

## Sources
[S1] An Overview of Rendering from Volume Data --- including Surface and Volume Rendering, https://core.ac.uk/download/63614.pdf, fetched 2026-10-30 (unofficial)
[S2] Volumetric rendering of medical data : applications to stereotactic neurosurgery planning, https://doi.org/10.82308/10070, fetched 2026-10-26 (unofficial)
