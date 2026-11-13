# Cross-section Texture Synthesis with Flow Vectors

Also called: Interactive system for designing and browsing volumetric illustrations, Internal Texture Synthesis, Volumetric Illustration Synthesis, Volumometric Illustration System

## Overview
This method synthesizes internal textures for 3D models by generating cross-sectional images on demand using 2D reference images, rather than storing explicit 3D volumetric data [S1]. It solves the problem of manually defining complex internal structures for volumetric illustrations by mapping pixels from a 2D control map (containing flow vectors) to a reference volume, ensuring the synthesized texture matches the local neighborhood structure and orientation defined by the flow [S1]. Unlike [Isotropic Texture Synthesis](13939-isotropic-texture-synthesis.md), which assumes uniform distribution without orientation dependency, this method supports layered and oriented textures by using flow vectors to guide the sampling geometry in the reference volume [S1].

## How it works

### Inputs and Outputs
**Inputs:**
1.  A target 3D surface mesh model.
2.  A 2D reference image (or a selected region thereof).
3.  A texture type selection: Isotropic, Layered, or Oriented [S1].
4.  Guidance information:
    *   For Isotropic: None required [S1].
    *   For Layered/Oriented: A 2D control map (flow vectors) associated with the cross-section [S1].
5.  A 3D reference volume constructed from the reference image based on the texture type [S1].

**Outputs:**
*   A synthesized 2D image representing the internal texture of a specific cross-section of the 3D model [S1].

### Data Structures

**Reference Volume:**
*   A 3D data structure storing texture information derived from the 2D reference image.
*   For **Isotropic** textures: The reference image region is transferred to all faces of a reference cube [S1].
*   For **Layered** textures: Associated with a vertical flow orientation; the volume represents layers perpendicular to the flow [S1].
*   For **Oriented** textures: The volume is sliced at appropriate angles to generate 2D reference images for each pixel [S1].

**Control Map (Flow Vectors):**
*   A 2D array corresponding to the cross-section resolution (e.g., 300x300 pixels) [S1].
*   Each pixel contains a flow vector defining the local orientation and slant angle for texture synthesis [S1].

**Multi-resolution Pyramid:**
*   Used for similarity comparison during synthesis [S1].
*   **Isotropic/Oriented:** Three levels.
    *   Lower resolution: 3x3 square neighbor.
    *   Higher resolution: 5x5 L-shaped neighbor [S1].
*   **Layered:**
    *   Lower resolution: 3x3 square neighbor.
    *   Higher resolution: 5x3 rectangular neighbor [S1].

**Cache:**
*   Stores sampled images from the reference volume.
*   Key: Discretized slant angle.
*   Value: Sampled 2D image from the reference volume [S1].

### Steps

1.  **Model Preparation:** The user cuts the target 3D model using a freeform stroke on the screen. The model opens with animation to reveal the cross-section [S1].
2.  **Region Selection:** The cross-section is divided into closed regions. The user clicks a target region to specify the 3D area to be textured [S1].
3.  **Texture Type Selection:** The user selects one of three types: Isotropic, Layered, or Oriented [S1].
4.  **Reference Image Import:** The user drags a 2D image file into the system. For non-isotropic types, the user may select a specific region via rubber banding [S1].
5.  **Reference Volume Construction:**
    *   **Isotropic:** The selected region is transferred to all faces of a reference cube [S1].
    *   **Layered/Oriented:** The system constructs a 3D reference volume or prepares slicing parameters based on the flow orientation [S1].
6.  **Flow Vector Assignment (Layered/Oriented only):** The user provides guidance information (flow vectors) for the cross-section. For layered textures, this defines the layering direction; for oriented textures, it defines the slice angle [S1].
7.  **Cross-section Synthesis:** For each pixel in the target cross-section:
    a.  Approximate the pixel's neighborhood as a small flat rectangle [S1].
    b.  Determine the slant angle of this rectangle relative to the flow vector [S1].
    c.  Sample the reference volume to find a corresponding neighborhood.
    d.  Compare similarity between the cross-section neighborhood and the sampled reference neighborhood.
    e.  Select the best matching pixel/region from the reference volume.
    f.  Assign the color to the cross-section pixel [S1].

### Formulas and Geometry

**Neighborhood Approximation:**
For a target pixel $p$ on the cross-section, the neighborhood $N_p$ is approximated as a small flat rectangle [S1].

**Slant Angle Matching:**
For a pixel $q$ in the reference volume, the sampled neighborhood $N_q$ is a small flat rectangle. The slant angle $\theta_q$ of $N_q$ (angle between the rectangle and the vertical flow vector) must equal the slant angle $\theta_p$ of $N_p$ on the cross-section [S1]:
$$ \theta_q = \theta_p $$

**Similarity Comparison:**
The system compares the similarity between $N_p$ and $N_q$ using multi-resolution pyramids. The similarity metric is not explicitly defined in the sources, but it relies on standard 2D texture synthesis techniques (e.g., Wei and Levoy [S1]).

**Caching Key:**
The slant angle $\theta$ is discretized with a step size of $\Delta \theta = \pi/32$. The cache key is $k = \lfloor \theta / \Delta \theta \rfloor$ [S1].

### Parameters

*   **Sampling Region Size:** The slanted 2D square region sampled from the reference volume is 45x45 pixels [S1].
*   **Discretization Step:** $\pi/32$ radians for slant angles [S1].
*   **Cross-section Resolution:** Approximately 300x300 pixels [S1].
*   **Neighbor Sizes:**
    *   Isotropic/Oriented (Low): 3x3 square [S1].
    *   Isotropic/Oriented (High): 5x5 L-shaped [S1].
    *   Layered (Low): 3x3 square [S1].
    *   Layered (High): 5x3 rectangular [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def synthesize_cross_section(cross_section_pixels, control_map, reference_volume, texture_type, cache):
    """
    Synthesizes the texture for a cross-section.
    
    Args:
        cross_section_pixels: List of (x, y) coordinates for pixels in the cross-section.
        control_map: 2D array of flow vectors for the cross-section.
        reference_volume: 3D data structure containing reference texture data.
        texture_type: Enum {ISOTROPIC, LAYERED, ORIENTED}.
        cache: Dictionary mapping discretized slant angles to sampled 2D images.
        
    Returns:
        Synthesized 2D image of the cross-section.
    """
    synthesized_image = empty_image(size=cross_section_pixels.shape)
    
    for pixel in cross_section_pixels:
        # 1. Get flow vector for the current pixel
        flow_vector = control_map[pixel.y][pixel.x]
        
        # 2. Approximate neighborhood on cross-section
        # The neighborhood is a small flat rectangle.
        # For isotropic, orientation is ignored. For layered/oriented, use flow vector.
        neighborhood_cross = get_neighborhood_rectangle(pixel, flow_vector, texture_type)
        
        # 3. Determine slant angle
        # Slant angle is the angle between the rectangle and the vertical flow vector.
        # For isotropic, this is effectively 0 or irrelevant.
        slant_angle = compute_slant_angle(neighborhood_cross, flow_vector)
        
        # 4. Cache lookup or sampling from reference volume
        # Discretize slant angle
        discretized_angle = discretize(slant_angle, step=pi/32)
        
        if discretized_angle in cache:
            sampled_reference_image = cache[discretized_angle]
        else:
            # Sample a 45x45 slanted 2D square region from the reference volume
            # centered at the appropriate location in the reference volume.
            # The orientation of this square matches the slant angle.
            sampled_reference_image = sample_reference_volume(
                reference_volume, 
                slant_angle, 
                size=(45, 45)
            )
            cache[discretized_angle] = sampled_reference_image
            
        # 5. Find best match using 2D texture synthesis
        # Compare neighborhood_cross with patches in sampled_reference_image
        # using multi-resolution pyramids.
        best_match_patch = find_best_match(
            neighborhood_cross, 
            sampled_reference_image,
            texture_type
        )
        
        # 6. Assign color
        synthesized_image[pixel.y][pixel.x] = best_match_patch.center_color
        
    return synthesized_image

def get_neighborhood_rectangle(pixel, flow_vector, texture_type):
    """
    Approximates the neighborhood of a pixel as a small flat rectangle.
    """
    if texture_type == ISOTROPIC:
        # For isotropic, rotation about flow vector is ignored.
        # Assume a standard square or circle approximation.
        return create_square_rectangle(pixel, size=5)
    else:
        # For layered/oriented, the rectangle is aligned with the flow vector.
        # The slant angle determines the orientation of the rectangle.
        return create_rectangle_aligned_with_flow(pixel, flow_vector, size=5)

def compute_slant_angle(rectangle, flow_vector):
    """
    Computes the angle between the rectangle's normal/axis and the vertical flow vector.
    """
    # Calculate angle between the rectangle's primary axis and the vertical direction
    # defined by the flow vector.
    rect_axis = get_rectangle_axis(rectangle)
    vertical = get_vertical_direction(flow_vector)
    angle = acos(dot(normalize(rect_axis), normalize(vertical)))
    return angle

def discretize(angle, step=pi/32):
    """
    Discretizes the angle to a grid step of pi/32.
    """
    return floor(angle / step) * step

def sample_reference_volume(reference_volume, slant_angle, size=(45, 45)):
    """
    Samples a slanted 2D square region from the 3D reference volume.
    The region is centered at the current position in the reference volume.
    """
    # Create a 2D image of size 45x45
    image = empty_image(size=size)
    
    # Define the sampling plane in 3D space based on slant_angle
    # The plane is tilted by slant_angle relative to the vertical axis.
    plane = create_tilted_plane(slant_angle)
    
    # Sample pixels from the reference volume onto this plane
    for y in range(size[1]):
        for x in range(size[0]):
            # Map 2D pixel to 3D point on the plane
            point_3d = map_2d_to_3d(x, y, plane, center=reference_volume.center)
            
            # Interpolate color from reference_volume at point_3d
            color = interpolate_volume(reference_volume, point_3d)
            image[y][x] = color
            
    return image

def find_best_match(neighborhood_cross, sampled_reference_image, texture_type):
    """
    Finds the best matching patch in sampled_reference_image for neighborhood_cross.
    Uses multi-resolution pyramids.
    """
    # Build multi-resolution pyramid for neighborhood_cross and sampled_reference_image
    # based on texture_type neighbor sizes.
    pyramid_cross = build_pyramid(neighborhood_cross, texture_type)
    pyramid_ref = build_pyramid(sampled_reference_image, texture_type)
    
    # Perform standard 2D texture synthesis matching (e.g., Wei and Levoy)
    # Iterate from lowest resolution to highest.
    best_match = None
    for level in pyramid_levels(pyramid_cross):
        patch_cross = get_patch_at_level(pyramid_cross, level)
        patch_ref = find_similar_patch(pyramid_ref, patch_cross)
        best_match = patch_ref
        
    return best_match

def build_pyramid(image, texture_type):
    """
    Builds a multi-resolution pyramid.
    """
    if texture_type in [ISOTROPIC, ORIENTED]:
        # 3 levels
        # Low: 3x3 square neighbor
        # High: 5x5 L-shaped neighbor
        return create_multi_resolution_pyramid(image, levels=3, low_neighbor=(3,3), high_neighbor=(5,5))
    else: # LAYERED
        # Low: 3x3 square neighbor
        # High: 5x3 rectangular neighbor
        return create_multi_resolution_pyramid(image, levels=3, low_neighbor=(3,3), high_neighbor=(5,3))
```

### Edge Cases and Pitfalls

*   **Rotation Ambiguity:** Rotation about the flow vector is ignored because the reference volume is assumed to have an isotropic structure on cross-sections perpendicular to the flow orientation [S1]. This may lead to artifacts if the texture has directional features perpendicular to the flow.
*   **Cascading Resampling:** Quality issues arise due to cascading resampling and resulting distortion [S1].
*   **Performance:** Layered and oriented textures require more computation than standard texture synthesis. Layered textures require rotating neighbors, and oriented textures require generating a 2D reference image by slicing the reference volume at an appropriate angle for each pixel [S1].
*   **Boundary Conditions:** The sampling region (45x45) must fit within the reference volume. If the reference volume is too small, sampling may fail [S1].

## Operating characteristics

*   **Time Complexity:** Not explicitly given in Big-O notation. Synthesis times vary by model and texture type. On a Pentium M 1.6-GHz processor with 1 GB RAM:
    *   Meat: 22 sec
    *   Cucumber: 4 sec
    *   Bamboo: 14 sec
    *   Stomach: 18 sec
    *   Tooth: 32 sec [S1]
*   **Memory Complexity:** The data size is significantly smaller than explicit volumetric data. For a 300x300 cross-section, the quality is comparable to 300^3 colored voxels (approx. 80 MB storage), but the actual model data is much smaller (e.g., Meat: 622 kb, Cucumber: 53 kb) [S1].
*   **Strengths:**
    *   Significantly reduces data storage requirements compared to explicit 3D volumetric data [S1].
    *   Allows designers to add 3D internal textures without specifying each voxel manually [S1].
    *   Supports interactive modeling with progressive synthesis [S1].
*   **Limitations:**
    *   Computational cost is higher for layered and oriented textures compared to isotropic [S1].
    *   Quality issues due to cascading resampling and distortion [S1].
    *   Cannot be used for applications requiring real volumetric data, such as translucent rendering or volumetric simulation [S1].
*   **When to Use:** Suitable for creating lightweight volumetric illustrations for communication purposes (e.g., education, medical visualization) where explicit volumetric data is not required [S1].
*   **When Not to Use:** Not suitable for applications requiring accurate volumetric properties like translucency or simulation [S1].

## Background

*   **Origin:** Proposed in 2004 by the authors of "Volumetric Illustration" [S1].
*   **History:** The system builds on previous work in texture synthesis [Heeger and Bergen 1995, Wei 2001] and 3D texture mapping [Turk 2001, Praun et al. 2000]. It addresses the difficulty of designing detailed internal textures of 3D volumes using standard 2D devices [S1].
*   **Applications:** Useful for conveying volumetric information between teachers and students, doctors and patients, or virtual-reality content providers and consumers [S1].

## Sources

[S1] Volumetric illustration, https://doi.org/10.1145/1186562.1015723, fetched 2026-10-30
