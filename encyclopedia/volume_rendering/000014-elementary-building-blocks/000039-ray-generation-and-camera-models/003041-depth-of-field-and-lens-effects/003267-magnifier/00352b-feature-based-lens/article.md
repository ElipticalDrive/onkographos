# Feature-based lens

Also called: Feature-centric Lens, Feature-sensitive lens

## Overview
The feature-based lens is a focus+context technique for volume rendering that dynamically defines the shape of the magnification region based on the geometry of specific features identified via segmentation, rather than a fixed geometric primitive [S1, book][S2]. It solves the problem of highlighting semantically important structures (e.g., bones, tumors) by dedicating more screen area to them while compressing surrounding context to preserve spatial relationships [S1, book][S2]. Unlike the [Sampling-rate-based lens](6758-sampling-rate-based-lens.md), which varies ray density, the feature-based lens modifies ray directions to pass through a virtual focal point, but restricts this modification to rays intersecting the segmented feature and its immediate transition boundary [S2].

## How it works
The method operates by dynamically determining the direction of each ray cast from the camera based on whether the ray intersects a user-defined feature of interest within a segmented volume dataset [S2]. The core logic distinguishes between three regions on the image plane: the feature interior, the transition region, and the background [S1, book][S2].

### Inputs and Outputs
*   **Inputs:**
    *   Segmented volume dataset $V$, where voxels are labeled by feature ID [S1, book][S2].
    *   Camera parameters (eye position, view direction, image plane dimensions) [S1, book][S2].
    *   Focal point $F$ in world space [S1, book][S2].
    *   Transition region width $l_b$ [S1, book][S2].
*   **Outputs:**
    *   Modified ray directions for each pixel [S2].
    *   Final composited image [S2].

### Data Structures
The GPU implementation uses a multi-pass fragment program pipeline to compute ray directions before the main volume rendering pass [S2]:
1.  **Pass 1 (RayDirection):** Calculates initial ray direction and lens/feature information [S2].
2.  **Pass 1+ (RayLensBorder):** Calculates the distance field for the transition region and modifies ray directions based on the distance [S2].
3.  **Pass 2 (RayTfrontback):** Computes intersections with the volume bounding box ($t_{front}$, $t_{back}$) [S2].
4.  **Pass 3 (RayCasting):** Casts rays and composites color [S2].
5.  **Pass 4 (Rendering):** Outputs the rendering results to the frame buffer [S2].

### Numbered Steps
1.  **Feature Detection & Projection:** Perform ray casting through the volume. Detect if the ray intersects a specific feature of interest (e.g., bone in a segmented dataset) [S2]. For each ray orthogonally incident upon the image plane, if it passes through the feature, mark it as inside the feature [S1, book][S2].
2.  **Transition Region Computation:** Compute the distance $d$ from each pixel's entry point to the boundary of the feature-projected area [S1, book][S2]. This requires a two-pass computation: the first pass defines the feature-projected region, and the second pass computes the distance from a pixel to this region [S2]. Two methods are provided:
    *   *Distance Transform:* Project the feature onto the image plane, fill all interior points with a constant value, and apply a distance transform to mark the transition region field [S1, book].
    *   *GPU Search Circle:* For each pixel outside the feature, search within a circle of maximal radius $l_b$ for the closest neighbor pixel that is part of the projected feature. The smallest distance found is used as $d$ [S1, book][S2].
3.  **Ray Direction Determination:** For each pixel, compute the new direction $\vec{r}'$ [S1, book][S2]:
    *   **Inside Feature:** If the ray passes through the feature, its new direction points directly to the focal point $F$ [S1, book][S2].
    *   **Inside Transition Region:** If the ray does not pass through the feature but is inside the transition region on the image plane, calculate the distance $d$ from its entry point to the boundary of the feature-projected area and compute the new direction using Equations 1-3 [S2].
    *   **Outside Transition Region:** Otherwise, the ray continues along its original direction [S1, book][S2].
4.  **Volume Rendering:** Cast rays with the modified directions through the volume. Perform standard volume rendering using the segmented data and transfer functions [S2].
5.  **Output:** Composite the color and opacity into the frame buffer [S2].

### Formulas
Let $P$ be the pixel entry point on the image plane. Let $F$ be the focal point in world space. Let $d$ be the distance from $P$ to the boundary of the feature-projected area. Let $l_b$ be the transition region width [S1, book][S2].

The new ray direction $\vec{r}'$ is computed as follows [S1, book][S2]:

1.  **Inside Feature:**
    $$ \vec{r}' = \text{normalize}(F - P) $$
    *(Note: The sources state "new direction is pointing to F". In a standard pinhole camera, rays originate from the eye. The "direction" here refers to the vector from the image plane point towards the focal point, effectively bending the ray path.)*

2.  **Inside Transition Region:**
    The sources reference "Equations 1-3" for computing the new direction based on $d$ [S2]. The exact algebraic form of Equations 1-3 is not transcribed in the provided text, but the direction is computed using $d$ to smoothly transition from the focal direction to the original direction [S2].

3.  **Outside Transition Region:**
    $$ \vec{r}' = \vec{r}_{\text{original}} $$

### Parameters
*   **$F$ (Focal Point):** The 3D point in space to which rays inside the feature are bent [S1, book][S2].
*   **$l_b$ (Transition Width):** The width of the transition region, used as the maximal radius for the searching circle in the distance computation [S1, book][S2].
*   **$d$ (Distance):** The distance from the ray's entry point to the boundary of the feature-projected area [S1, book].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_feature_lens_directions(image_width, image_height, eye_pos, focal_point, feature_mask_3d, transition_width_lb):
    """
    Computes modified ray directions for a feature-based lens.
    Args:
        image_width, image_height: Image plane dimensions.
        eye_pos: 3D tuple, camera eye position.
        focal_point: 3D tuple, focal point F.
        feature_mask_3d: 3D list of lists of bools, segmented feature mask.
        transition_width_lb: Float, transition region width lb.
    Returns:
        ray_directions: 2D list of 3D tuples, modified directions per pixel.
    """
    def normalize(v):
        length = (v[0]**2 + v[1]**2 + v[2]**2)**0.5
        if length == 0: return (0, 0, 0)
        return (v[0]/length, v[1]/length, v[2]/length)

    def ray_intersects_feature(origin, direction, mask, bounds):
        # [G] Simple ray-marching against 3D mask. 
        # In practice, this uses GPU fragment programs or optimized marching.
        step = 0.05
        pos = list(origin)
        for _ in range(int(1.0 / step)):
            ix, iy, iz = int(pos[0]), int(pos[1]), int(pos[2])
            if bounds[0] <= ix < bounds[1] and bounds[2] <= iy < bounds[3] and bounds[4] <= iz < bounds[5]:
                if mask[ix][iy][iz]: return True
            pos[0] += direction[0] * step
            pos[1] += direction[1] * step
            pos[2] += direction[2] * step
        return False

    ray_directions = [[None for _ in range(image_width)] for _ in range(image_height)]
    feature_projected_mask = [[False for _ in range(image_width)] for _ in range(image_height)]
    
    # [G] Volume bounds are assumed known or derived from mask dimensions
    volume_bounds = (0, 1, 0, 1, 0, 1) 

    for y in range(image_height):
        for x in range(image_width):
            # [G] Standard pinhole camera projection to get 3D point on image plane
            u = (x / image_width) * 2 - 1
            v = (y / image_height) * 2 - 1
            pixel_pos = (eye_pos[0] + u, eye_pos[1] + v, eye_pos[2] + 1.0)
            original_dir = normalize((pixel_pos[0]-eye_pos[0], pixel_pos[1]-eye_pos[1], pixel_pos[2]-eye_pos[2]))
            
            # Detect if the ray intersects a specific feature of interest [S2]
            if ray_intersects_feature(eye_pos, original_dir, feature_mask_3d, volume_bounds):
                feature_projected_mask[y][x] = True

    # Compute distance field using search circle method [S1, book][S2]
    distance_field = [[float('inf') for _ in range(image_width)] for _ in range(image_height)]
    
    for y in range(image_height):
        for x in range(image_width):
            if feature_projected_mask[y][x]:
                distance_field[y][x] = 0.0
            else:
                min_dist = float('inf')
                y_min = max(0, y - int(transition_width_lb))
                y_max = min(image_height - 1, y + int(transition_width_lb))
                x_min = max(0, x - int(transition_width_lb))
                x_max = min(image_width - 1, x + int(transition_width_lb))
                
                for dy in range(y_min, y_max + 1):
                    for dx in range(x_min, x_max + 1):
                        if feature_projected_mask[dy][dx]:
                            dist = ((x - dx)**2 + (y - dy)**2)**0.5
                            if dist < min_dist:
                                min_dist = dist
                distance_field[y][x] = min_dist

    # Determine final directions
    for y in range(image_height):
        for x in range(image_width):
            u = (x / image_width) * 2 - 1
            v = (y / image_height) * 2 - 1
            pixel_pos = (eye_pos[0] + u, eye_pos[1] + v, eye_pos[2] + 1.0)
            original_dir = normalize((pixel_pos[0]-eye_pos[0], pixel_pos[1]-eye_pos[1], pixel_pos[2]-eye_pos[2]))
            
            d = distance_field[y][x]
            
            if feature_projected_mask[y][x]:
                # Inside feature: point to F [S1, book][S2]
                ray_directions[y][x] = normalize((focal_point[0]-pixel_pos[0], focal_point[1]-pixel_pos[1], focal_point[2]-pixel_pos[2]))
            elif d < transition_width_lb:
                # Inside transition region: compute direction based on d [S2]
                # [G] Exact interpolation function not specified in sources; standard blending used here
                t = d / transition_width_lb
                dir_to_focal = normalize((focal_point[0]-pixel_pos[0], focal_point[1]-pixel_pos[1], focal_point[2]-pixel_pos[2]))
                blended = (original_dir[0]*(1-t) + dir_to_focal[0]*t,
                           original_dir[1]*(1-t) + dir_to_focal[1]*t,
                           original_dir[2]*(1-t) + dir_to_focal[2]*t)
                ray_directions[y][x] = normalize(blended)
            else:
                # Outside: original direction [S1, book][S2]
                ray_directions[y][x] = original_dir
                
    return ray_directions
```

### Edge Cases and Pitfalls
*   **Aliasing in Transition Region:** The sources note that the ray sampling rate is relatively low in the transition region, which can cause aliasing [S2].
    *   *Mitigation:* Use volume texture mip-mapping to adaptively choose the appropriate resolution of the volume data for rendering. Lower resolution volumes are chosen for regions sampled at a lower rate [S2].
*   **Precision:** The distance computation relies on pixel-level accuracy. Sub-pixel features may be lost if the resolution is insufficient [S1, book].

## Operating characteristics
*   **Time Complexity:** The pre-computation of the distance field is $O(W \cdot H \cdot l_b^2)$ if using a naive search circle, where $W, H$ are image dimensions and $l_b$ is the transition width. GPU acceleration using fragment programs allows this to be performed efficiently in parallel [S2]. The main rendering pass is $O(N)$ where $N$ is the number of voxels sampled along rays.
*   **Memory Complexity:** Requires storage for the feature projected mask, distance field, and ray direction buffer, all of size $O(W \cdot H)$ [S2].
*   **Performance:** The sources provide performance data for various datasets (lobster, engine, bonsai, foot, aneurism, frog) but do not explicitly list the frame rates for the feature-based lens specifically, only mentioning that the four-pass fragment program implementation is efficient [S2]. The addition of Pass 1+ (RayLensBorder) adds overhead compared to a standard magnifier but is necessary for the dynamic distance calculation [S2].
*   **Strengths:**
    *   Allows feature-centric enlargement, highlighting semantically important structures [S1, book][S2].
    *   Does not interfere with the composition of voxels with different properties because of segmentation [S2].
    *   Retains spatial context via the transition region [S1, book][S2].
*   **Limitations:**
    *   Requires segmented volume data [S1, book][S2].
    *   Potential for aliasing in the transition region without mip-mapping [S2].

## Background
The feature-based lens was introduced by S. Bruckner, M. E. Gröller, K. Mueller, B. Preim, and D. Silver as part of the "Magic Volume Lens" framework [S1, book][S2]. It extends the traditional magnifier concept by incorporating semantic information from segmentation to define the lens shape dynamically [S1, book][S2]. The method is designed for interactive volume visualization, allowing users to highlight portions of interest in volume objects by dedicating more screen area to them [S1, book]. It is particularly useful for medical visualization (e.g., brain datasets, frog skeletons) where specific anatomical structures need to be examined in detail while maintaining context [S1, book][S2].

## Sources
[S1] Illustrative Focus+Context Approaches in Interactive Volume Visualization, https://doi.org/10.4230/dfu.sciviz.2010.136, fetched 2026-10-26 (book)
[S2] PDF The Magic Volume Lens: An Interactive Focus+Context Technique for ..., https://www3.cs.stonybrook.edu/~mueller/papers/lens_highres.pdf, fetched 2026-10-27
