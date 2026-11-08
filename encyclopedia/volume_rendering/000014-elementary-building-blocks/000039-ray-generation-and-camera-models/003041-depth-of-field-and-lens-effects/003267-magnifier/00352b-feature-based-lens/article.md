# Feature-based lens

Also called: Feature-centric Lens, Feature-sensitive lens

## Overview
The feature-based lens is a focus+context technique for volume rendering that dynamically defines the shape of the magnification region based on the geometry of specific features identified via segmentation, rather than a fixed geometric primitive like a sphere or cylinder [S1, book][S2]. It solves the problem of highlighting semantically important structures (e.g., bones, tumors) by dedicating more screen area to them while compressing surrounding context to preserve spatial relationships [S1, book][S2]. Unlike the [Sampling-rate-based lens](6758-sampling-rate-based-lens.md), which varies ray density, the feature-based lens modifies ray directions to pass through a virtual focal point, but restricts this modification to rays intersecting the segmented feature and its immediate transition boundary [S2].

## How it works
The method operates by dynamically determining the direction of each ray cast from the camera based on whether the ray intersects a user-defined feature of interest within a segmented volume dataset. The core logic distinguishes between three regions on the image plane: the feature interior, the transition region, and the background.

### Inputs and Outputs
*   **Inputs:**
    *   Segmented volume dataset $V$, where voxels are labeled by feature ID.
    *   Camera parameters (eye position, view direction, image plane dimensions).
    *   Focal point $F$ in world space.
    *   Transition region width $l_b$.
*   **Outputs:**
    *   Modified ray directions for each pixel.
    *   Final composited image.

### Data Structures
The implementation relies on a two-pass pre-computation on the GPU to determine ray directions before the main volume rendering pass [S2].

1.  **Feature Projected Region Map:** A 2D texture (image plane resolution) where pixels corresponding to the projection of the selected feature are marked with a constant value (e.g., 1), and others are 0.
2.  **Distance Field Texture:** A 2D texture storing the distance $d$ from each pixel to the boundary of the feature-projected region. This is computed using a distance transform or a local search [S1, book][S2].
3.  **Ray Direction Buffer:** Stores the modified ray direction vector for each pixel.

### Numbered Steps

1.  **Feature Projection:** Project the 3D segmented feature onto the 2D image plane. For each pixel, if the ray from the camera eye through the pixel intersects any voxel belonging to the target feature, mark the pixel as "inside feature" [S2].
2.  **Transition Region Computation:** Compute the distance field for the transition region surrounding the projected feature [S1, book][S2].
    *   *Method A (Distance Transform):* Fill the interior of the projected feature with a constant value. Apply a distance transform to generate a field where values represent the distance to the feature boundary [S1, book].
    *   *Method B (GPU Search Circle):* For each pixel outside the feature, search within a circle of maximal radius $l_b$ for the closest neighbor pixel that is part of the projected feature. The distance to this neighbor is the value $d$ for the current pixel [S1, book][S2].
3.  **Ray Direction Determination:** For each pixel $(u, v)$ on the image plane, determine the new ray direction $\vec{r}'$ based on the computed distance $d$ and feature membership:
    *   **Case 1: Inside Feature.** If the ray intersects the feature, the new direction points directly to the focal point $F$.
    *   **Case 2: Inside Transition Region.** If the ray does not intersect the feature but is within the transition region (distance $d < l_b$), compute the new direction using Equations 1-3 (see below).
    *   **Case 3: Outside Transition Region.** If the ray is outside the transition region, the direction remains unchanged (orthogonal to the image plane or as defined by the base camera model) [S1, book][S2].
4.  **Volume Rendering:** Cast rays with the modified directions through the volume. Perform standard volume rendering (marching cubes, ray casting, etc.) using the segmented data and transfer functions [S2].
5.  **Output:** Composite the color and opacity into the frame buffer [S2].

### Formulas
Let $P$ be the pixel entry point on the image plane. Let $F$ be the focal point in world space. Let $d$ be the distance from $P$ to the boundary of the feature-projected area. Let $l_b$ be the transition region width.

The new ray direction $\vec{r}'$ is computed as follows [S1, book][S2]:

1.  **Inside Feature:**
    $$ \vec{r}' = \text{normalize}(F - P) $$
    *(Note: The sources state "new direction is pointing to F". In a standard pinhole camera, rays originate from the eye. The "direction" here refers to the vector from the image plane point towards the focal point, effectively bending the ray path.)*

2.  **Inside Transition Region:**
    The sources reference "Equations 1-3" for computing the new direction based on $d$ [S2]. While the explicit algebraic form of Equations 1-3 is not fully transcribed in the provided text snippets, the logic is defined by the distance $d$. Typically, this involves interpolating between the original direction and the focal direction based on the normalized distance $d/l_b$.
    *   If $d=0$ (at the boundary), the direction is likely continuous with the feature interior direction.
    *   If $d=l_b$ (at the edge of the transition), the direction is the original unmodified direction.
    *   *Implementation Note:* The exact interpolation function is not specified in the provided text. A standard linear interpolation of the direction vector or the projection offset is implied by the "transition region" concept [S1, book].

3.  **Outside Transition Region:**
    $$ \vec{r}' = \vec{r}_{original} $$

### Parameters
*   **$F$ (Focal Point):** The 3D point in space to which rays inside the feature are bent. [S1, book][S2]
*   **$l_b$ (Transition Width):** The width of the transition region. It serves as the maximal radius for the GPU-accelerated search circle in Method B [S1, book][S2]. Typical values are not explicitly given in the sources, but it must be large enough to provide a smooth visual transition [S1, book].
*   **$d$ (Distance):** The distance from the ray's entry point to the boundary of the feature-projected area. [S1, book][S2]

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_feature_lens_directions(image_width, image_height, eye_pos, focal_point, feature_mask_3d, transition_width_lb):
    """
    Computes the modified ray directions for a feature-based lens.
    
    Args:
        image_width, image_height: Dimensions of the image plane.
        eye_pos: 3D vector, camera eye position.
        focal_point: 3D vector, the focal point F.
        feature_mask_3d: 3D array, boolean mask of the segmented feature.
        transition_width_lb: Float, width of the transition region.
        
    Returns:
        ray_directions: 2D array of shape (image_height, image_width, 3), 
                        containing normalized direction vectors for each pixel.
    """
    
    # Step 1: Project feature onto image plane
    # Create a 2D mask where True indicates the ray intersects the feature
    feature_projected_mask = np.zeros((image_height, image_width), dtype=bool)
    
    for y in range(image_height):
        for x in range(image_width):
            # Calculate original ray direction from eye to pixel
            pixel_pos = image_plane_to_world(x, y, eye_pos) # Helper to get 3D point on image plane
            ray_dir = normalize(pixel_pos - eye_pos)
            
            # Check if ray intersects feature in 3D volume
            # This requires ray-casting against the 3D feature_mask_3d
            if ray_intersects_volume(eye_pos, ray_dir, feature_mask_3d):
                feature_projected_mask[y, x] = True

    # Step 2: Compute distance field for transition region
    # Initialize distance field with infinity
    distance_field = np.full((image_height, image_width), np.inf)
    
    # Method B: GPU-accelerated search circle approach
    # For each pixel, find the closest neighbor in the projected feature region
    # within a circle of radius transition_width_lb (in pixels)
    
    # Precompute pixel coordinates for distance calculation
    pixel_coords = np.array(np.meshgrid(np.arange(image_width), np.arange(image_height))).T.reshape(-1, 2)
    
    # Get coordinates of all pixels inside the feature
    feature_pixel_coords = np.argwhere(feature_projected_mask)
    
    # For efficiency, use a spatial index or local search
    # Here we describe the local search logic for each pixel outside the feature
    for y in range(image_height):
        for x in range(image_width):
            if feature_projected_mask[y, x]:
                distance_field[y, x] = 0.0
            else:
                # Search in a circle of radius lb around (x, y)
                min_dist = np.inf
                search_radius = transition_width_lb
                
                # Define search bounds
                y_min = max(0, y - int(search_radius))
                y_max = min(image_height - 1, y + int(search_radius))
                x_min = max(0, x - int(search_radius))
                x_max = min(image_width - 1, x + int(search_radius))
                
                for dy in range(y_min, y_max + 1):
                    for dx in range(x_min, x_max + 1):
                        if feature_projected_mask[dy, dx]:
                            dist = np.sqrt((x - dx)**2 + (y - dy)**2)
                            if dist < min_dist:
                                min_dist = dist
                
                distance_field[y, x] = min_dist

    # Step 3: Determine ray directions
    ray_directions = np.zeros((image_height, image_width, 3))
    
    for y in range(image_height):
        for x in range(image_width):
            pixel_pos = image_plane_to_world(x, y, eye_pos)
            original_dir = normalize(pixel_pos - eye_pos)
            
            d = distance_field[y, x]
            
            if feature_projected_mask[y, x]:
                # Case 1: Inside feature, point to focal point
                direction_to_focal = normalize(focal_point - pixel_pos)
                ray_directions[y, x] = direction_to_focal
            elif d < transition_width_lb:
                # Case 2: Inside transition region
                # Compute new direction based on distance d
                # The sources reference Equations 1-3. 
                # Standard interpolation: blend between original direction and focal direction
                # based on normalized distance t = d / l_b.
                # At d=0 (boundary), direction is towards F.
                # At d=l_b (edge), direction is original.
                
                t = d / transition_width_lb
                direction_to_focal = normalize(focal_point - pixel_pos)
                
                # Linear interpolation of direction vectors
                # Note: Vector interpolation should be done carefully to maintain normalization
                blended_dir = (1 - t) * original_dir + t * direction_to_focal
                ray_directions[y, x] = normalize(blended_dir)
            else:
                # Case 3: Outside transition region
                ray_directions[y, x] = original_dir
                
    return ray_directions

def ray_intersects_volume(origin, direction, volume_mask):
    """
    Checks if a ray intersects the 3D feature mask.
    This is a placeholder for the actual ray-marching or ray-casting implementation.
    """
    # Implementation details depend on the specific volume rendering engine.
    # [G] In a typical implementation, this would involve marching along the ray
    # and checking if any voxel coordinate falls within the volume_mask.
    return False # Placeholder
```

### Edge Cases and Pitfalls
*   **Aliasing in Transition Region:** The sources note that the ray sampling rate is relatively low in the transition region, which can cause aliasing [S2].
    *   *Mitigation:* Use volume texture mip-mapping to adaptively choose the appropriate resolution of the volume data for rendering. Lower resolution volumes are chosen for regions sampled at a lower rate [S2].
*   **Dynamic Viewport:** The shape of the magnifier is defined dynamically by the feature shape within the *current* viewport. If the camera moves, the projected feature shape changes, requiring recomputation of the distance field and ray directions [S2].
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
    *   Computationally more expensive than fixed-shape lenses due to the dynamic distance field calculation [S2].
    *   Potential for aliasing in the transition region without mip-mapping [S2].

## Background
The feature-based lens was introduced by S. Bruckner, M. E. Gröller, K. Mueller, B. Preim, and D. Silver as part of the "Magic Volume Lens" framework [S1, book][S2]. It extends the traditional magnifier concept by incorporating semantic information from segmentation to define the lens shape dynamically [S1, book][S2]. The method is designed for interactive volume visualization, allowing users to highlight portions of interest in volume objects by dedicating more screen area to them [S1, book]. It is particularly useful for medical visualization (e.g., brain datasets, frog skeletons) where specific anatomical structures need to be examined in detail while maintaining context [S1, book][S2].

## Sources
[S1] Illustrative Focus+Context Approaches in Interactive Volume Visualization, https://doi.org/10.4230/dfu.sciviz.2010.136, fetched 2026-10-26 (book)
[S2] PDF The Magic Volume Lens: An Interactive Focus+Context Technique for ..., https://www3.cs.stonybrook.edu/~mueller/papers/lens_highres.pdf, fetched 2026-10-27
