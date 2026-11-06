# Volume Raycasting with Surface Extraction and Phong Shading

Also called: MexFunction Volume Rendering, Threshold-based Surface Raycasting

## Overview
This method extracts surfaces from 3D volume data using a scalar threshold and renders them via first-hit raycasting with Phong illumination [S1, unofficial]. It differs from standard volume rendering by accumulating color only at the first voxel exceeding the threshold, rather than integrating opacity and color along the entire ray path [S1, unofficial].

## How it works
**Inputs:**
- `volume`: 1D array of doubles representing the scalar field. Dimensions `size[0]`, `size[1]`, `size[2]`.
- `objectVolume`: 1D array of doubles representing object IDs or masks. Same dimensions as `volume`.
- `threshold`: Scalar value for surface extraction.
- `view`: Array `[imageWidth, imageHeight]`.
- `alpha`: Array of visibility weights for objects.
- `specularity`: Scalar weight for specular component.
- `rotation`: Array `[yaw, pitch, roll]` in degrees.
- `colors`: Array of RGB colors for objects.

**Outputs:**
- `outputBuffer`: 1D array of doubles of size `imageWidth * imageHeight * 3`. Stored in row-major order: index `w + h * imageWidth + channel * (imageWidth * imageHeight)`, where `channel` is 0 (R), 1 (G), or 2 (B).

**Data Structures:**
- `Grid`: Bounding box defined by min `(0,0,0)` and max `(size[0], size[1], size[2])`.
- `Camera`: Position `(size[0]/2 + viewOffset, size[1]/2, size[2]/2)`, target `(size[0]/2, size[1]/2, size[2]/2)`.
- `Ray`: Origin and direction vectors.

**Parameters:**
- `fov`: 52.51 degrees.
- `viewOffset`: 200.
- `shininess`: 16.0.
- `littleStep`: 1.0.
- `maxStep`: $\sqrt{size[0]^2 + size[1]^2 + size[2]^2}$.
- `ambientColor`: $(0.5, 0.5, 0.5)$.
- `diffuseColor`: $(0.6, 0.6, 0.6)$ (initially, overwritten per object).
- `specularColor`: $(0.7, 0.7, 0.7)$.

**Steps:**
1.  **Initialize Output:** Set all elements of `outputBuffer` to 0.
2.  **Identify Visible Objects:** Iterate through `alpha`. If `alpha[j] > 0.0`, add `j` to `visibleObj` and `alpha[j]` to `visibleAlpha`. Increment `visibleObjSize`.
3.  **Handle Empty Objects:** If `numberOfObjects` is 0, set `visibleObjSize = 1`, `visibleAlpha[0] = 1`, `visibleObj[0] = -1` (representing the main volume).
4.  **Setup Camera:**
    -   Calculate `scale = tan(fov * 0.5 * \pi / 180)`.
    -   Set camera position to `(size[0]/2 + viewOffset, size[1]/2, size[2]/2)`.
    -   Set look-at target to `(size[0]/2, size[1]/2, size[2]/2)`.
    -   Apply rotations (yaw, pitch, roll) to the camera orientation.
    -   Compute the look-at matrix.
5.  **Iterate Pixels:** For each pixel `(h, w)`:
    -   Calculate normalized coordinates: `x = w - imageWidth/2`, `y = h - imageHeight/2`.
    -   Transform `(x, y, 0)` and `(x, y, 1)` using the camera's look-at matrix to get `rayStart` and `rayEnd`.
    -   Create `Ray` from `rayStart` to `rayEnd`.
    -   Compute ray-AABB intersection with the volume grid. Get `tmin` and `tmax`. If no intersection, skip to next pixel.
    -   Initialize `start = ray.orig + ray.dir * tmin`.
    -   **Ray Marching:** For `t` from 0 to `maxStep` with step `littleStep`:
        -   Calculate voxel indices: `ix = floor(start.x)`, `iy = floor(start.y)`, `iz = floor(start.z)`.
        -   Clamp indices: if `ix < 0` set `ix = 0` (same for `iy`, `iz`). [G: The source code uses `floor(start.x) - 1` if positive, else 0. This implies sampling the voxel *behind* the current point or a specific indexing convention. We follow the source logic: `ix = floor(start.x) > 0 ? floor(start.x) - 1 : 0`].
        -   Calculate linear index: `linearIndex = ix + iy * size[0] + iz * (size[0] * size[1])`.
        -   Check bounds: If `grid.isInsideGrid(ix, iy, iz)` is false, continue.
        -   Check Surface Condition: If `volume[linearIndex] >= threshold` AND (`visibleObj[iObj] == -1` OR `objectVolume[linearIndex] == visibleObj[iObj]`):
            -   **Gradient Calculation:** Compute gradient `grad` at `(ix, iy, iz)` using central differences or trilinear interpolation derivatives.
            -   **Normal:** `normal = -grad / ||grad||`. [G: Guard against zero gradient norm to avoid division by zero].
            -   **Lighting Vectors:**
                -   `lightDir = (camera.position - lightPosition).normalize()`. Note: Source sets `lightPosition` identical to camera position, so `lightDir` is effectively `camera.from.normalize()` (vector from light to surface, normalized).
                -   `viewDir = rayStart.normalize()` (vector from surface to camera, normalized).
                -   `halfDir = (lightDir + viewDir).normalize()`.
            -   **Specular:** `specAngle = dot(halfDir, normal)`. `specular = specAngle ^ shininess`.
            -   **Illumination:**
                -   Get object-specific `diffuseColor` from `colors` array based on `visibleObj[iObj]`.
                -   `IlluminationI = ambientColor + diffuseColor * specularity + specularColor * specular * (1 - specularity)`.
                -   **Color Space Conversion:** Convert `IlluminationI` to YUV. Replace Y component with object-specific YUV diffuse color's Y. Convert back to RGB.
                -   **Accumulate:** `outputBuffer[w + h * imageWidth + channel * frameDimension] += visibleAlpha[iObj] * IlluminationI[channel]`.
            -   **Break:** Exit ray marching loop after first hit.
        -   Update `start = start + ray.dir * littleStep`.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def volume_raycasting_phong(volume, object_volume, size, threshold, view, alpha, specularity, rotation, colors):
    image_width, image_height = view
    frame_dimension = image_height * image_width
    
    # Initialize output buffer
    output_buffer = [0.0] * (image_width * image_height * 3)
    
    # Identify visible objects
    visible_obj = []
    visible_alpha = []
    visible_obj_size = 0
    
    if len(alpha) > 0:
        for j in range(len(alpha)):
            if alpha[j] > 0.0:
                visible_obj.append(j)
                visible_alpha.append(alpha[j])
                visible_obj_size += 1
    else:
        visible_obj_size = 1
        visible_alpha.append(1.0)
        visible_obj.append(-1)
        
    # Setup parameters
    fov = 52.51
    view_offset = 200.0
    shininess = 16.0
    little_step = 1.0
    max_step = math.sqrt(size[0]**2 + size[1]**2 + size[2]**2)
    
    half_width = abs(image_width / 2.0)
    half_height = abs(image_height / 2.0)
    
    # Camera setup
    cam_pos = (size[0]/2 + view_offset, size[1]/2, size[2]/2)
    look_at = (size[0]/2, size[1]/2, size[2]/2)
    
    # Light position is same as camera
    light_pos = cam_pos
    
    # Ambient, Diffuse, Specular colors (initial)
    ambient_color = (0.5, 0.5, 0.5)
    specular_color = (0.7, 0.7, 0.7)
    
    # Rotation application (simplified placeholder for matrix math)
    # In implementation, apply yaw, pitch, roll to camera orientation matrix
    
    # Precompute scale
    scale = math.tan(math.radians(fov * 0.5))
    
    # Helper: Get gradient
    def get_gradient(ix, iy, iz, vol, sz):
        # Central difference or trilinear derivative
        # Placeholder for actual gradient calculation logic from source
        # Source uses getGradient function which likely computes partial derivatives
        # Assuming standard central difference for simplicity if not provided
        # Note: Source code snippet shows linearIndex calculation for gradient neighbors
        # We assume a standard gradient function exists
        pass 

    # Helper: Vector operations
    def normalize(v):
        l = math.sqrt(v[0]**2 + v[1]**2 + v[2]**2)
        if l == 0: return (0,0,0)
        return (v[0]/l, v[1]/l, v[2]/l)
    
    def dot(v1, v2):
        return v1[0]*v2[0] + v1[1]*v2[1] + v1[2]*v2[2]
    
    def add(v1, v2):
        return (v1[0]+v2[0], v1[1]+v2[1], v1[2]+v2[2])
    
    def sub(v1, v2):
        return (v1[0]-v2[0], v1[1]-v2[1], v1[2]-v2[2])
    
    def scale_vec(v, s):
        return (v[0]*s, v[1]*s, v[2]*s)
    
    # YUV Conversion Matrices (from source)
    rgb_to_yuv = [
        [0.2126, 0.7152, 0.0722],
        [0, -0.09991, -0.33609],
        [0.436, 0, 0.615]
    ]
    yuv_to_rgb = [
        [1, 0, 1.28033],
        [1, -0.21482, -0.38059],
        [1, 2.12798, 0]
    ]
    
    def rgb_to_yuv_vec(rgb):
        y = rgb[0]*rgb_to_yuv[0][0] + rgb[1]*rgb_to_yuv[0][1] + rgb[2]*rgb_to_yuv[0][2]
        u = rgb[0]*rgb_to_yuv[1][0] + rgb[1]*rgb_to_yuv[1][1] + rgb[2]*rgb_to_yuv[1][2]
        v = rgb[0]*rgb_to_yuv[2][0] + rgb[1]*rgb_to_yuv[2][1] + rgb[2]*rgb_to_yuv[2][2]
        return (y, u, v)
        
    def yuv_to_rgb_vec(yuv):
        r = yuv[0]*yuv_to_rgb[0][0] + yuv[1]*yuv_to_rgb[0][1] + yuv[2]*yuv_to_rgb[0][2]
        g = yuv[0]*yuv_to_rgb[1][0] + yuv[1]*yuv_to_rgb[1][1] + yuv[2]*yuv_to_rgb[1][2]
        b = yuv[0]*yuv_to_rgb[2][0] + yuv[1]*yuv_to_rgb[2][1] + yuv[2]*yuv_to_rgb[2][2]
        return (r, g, b)

    # Iterate pixels
    for h in range(image_height):
        for w in range(image_width):
            for i_obj in range(visible_obj_size):
                x = w - half_width
                y = h - half_height
                
                # Ray generation (simplified lookAt matrix multiplication)
                # In real implementation, use the camera's lookAt matrix
                # rayStart = lookAt * (x, y, 0)
                # rayEnd = lookAt * (x, y, 1)
                # For this pseudocode, we assume rayStart and rayEnd are computed correctly
                ray_start = (0,0,0) # Placeholder
                ray_end = (0,0,0)   # Placeholder
                
                ray_dir = normalize(sub(ray_end, ray_start))
                ray_orig = ray_start
                
                # Ray-AABB Intersection
                # Placeholder for computeRayABBoxIntersection
                tmin = 0.0
                tmax = 0.0
                hit = False # Placeholder
                
                if hit:
                    start = add(ray_orig, scale_vec(ray_dir, tmin))
                    
                    for t in range(0, int(max_step), int(little_step)):
                        # Voxel Indexing
                        ix = math.floor(start[0])
                        iy = math.floor(start[1])
                        iz = math.floor(start[2])
                        
                        # Clamp logic from source
                        if ix > 0: ix = ix - 1
                        else: ix = 0
                        if iy > 0: iy = iy - 1
                        else: iy = 0
                        if iz > 0: iz = iz - 1
                        else: iz = 0
                        
                        linear_index = ix + iy * size[0] + iz * (size[0] * size[1])
                        
                        # Check bounds
                        if 0 <= ix < size[0] and 0 <= iy < size[1] and 0 <= iz < size[2]:
                            # Surface Condition
                            vol_val = volume[linear_index]
                            obj_match = (visible_obj[i_obj] == -1) or (object_volume[linear_index] == visible_obj[i_obj])
                            
                            if vol_val >= threshold and obj_match:
                                # Gradient
                                grad = get_gradient(ix, iy, iz, volume, size)
                                
                                # Normal
                                grad_norm = math.sqrt(grad[0]**2 + grad[1]**2 + grad[2]**2)
                                if grad_norm == 0:
                                    start = add(start, scale_vec(ray_dir, little_step))
                                    continue
                                normal = scale_vec(grad, -1.0 / grad_norm)
                                
                                # Lighting Vectors
                                light_dir = normalize(sub(light_pos, (0,0,0))) # Source uses camera.from.normalize()
                                view_dir = normalize(sub((0,0,0), ray_start)) # Source uses rayStart.normalize()
                                half_dir = normalize(add(light_dir, view_dir))
                                
                                # Specular
                                spec_angle = dot(half_dir, normal)
                                specular = spec_angle ** shininess
                                
                                # Diffuse Color
                                obj_idx = visible_obj[i_obj]
                                if obj_idx == -1:
                                    obj_idx = 0 # Default color index
                                diffuse_color = colors[obj_idx]
                                
                                # Illumination
                                spec_weight = specularity
                                illumination = (
                                    ambient_color[0] + diffuse_color[0] * spec_weight + specular_color[0] * specular * (1 - spec_weight),
                                    ambient_color[1] + diffuse_color[1] * spec_weight + specular_color[1] * specular * (1 - spec_weight),
                                    ambient_color[2] + diffuse_color[2] * spec_weight + specular_color[2] * specular * (1 - spec_weight)
                                )
                                
                                # YUV Conversion
                                yuv_ill = rgb_to_yuv_vec(illumination)
                                yuv_diff = rgb_to_yuv_vec(diffuse_color)
                                yuv_ill = (yuv_diff[0], yuv_ill[1], yuv_ill[2]) # Replace Y
                                illumination = yuv_to_rgb_vec(yuv_ill)
                                
                                # Accumulate
                                frame_dim = frame_dimension
                                output_buffer[w + h * image_width] += visible_alpha[i_obj] * illumination[0]
                                output_buffer[w + h * image_width + frame_dim] += visible_alpha[i_obj] * illumination[1]
                                output_buffer[w + h * image_width + frame_dim * 2] += visible_alpha[i_obj] * illumination[2]
                                
                                break # First hit
                        
                        start = add(start, scale_vec(ray_dir, little_step))

    return output_buffer
```

## Operating characteristics
- **Time Complexity:** $O(W \cdot H \cdot N_{rays} \cdot S)$, where $W, H$ are image dimensions, $N_{rays}$ is the number of visible objects per pixel (typically 1 due to break), and $S$ is the number of steps per ray.
- **Memory Complexity:** $O(W \cdot H \cdot 3)$ for the output buffer, plus $O(V)$ for volume data.
- **Performance:** The method is CPU-bound due to the nested loops and lack of GPU acceleration in the reference implementation [S1, unofficial]. The use of `double` precision and YUV conversion adds overhead.
- **Strengths:** Simple implementation, direct surface extraction, Phong shading provides realistic specular highlights.
- **Limitations:** No transparency or internal structure visualization (first-hit only). Sensitive to threshold selection. No anti-aliasing.

## Background
This method originates from a MATLAB MEX function implementation for volume rendering [S1, unofficial]. It is part of a family of threshold-based surface rendering techniques that prioritize speed and simplicity over physical accuracy.

## Sources
[S1] thoniorf/volumeraycasting, https://github.com/thoniorf/volumeraycasting, fetched 2026-10-26 (unofficial)
