# Appel's Ray Casting Algorithm

Also called: Appel's Algorithm

## Overview
Appel's Ray Casting Algorithm is the foundational image-order rendering method for determining visible surfaces and occlusion in 3D scenes by tracing rays from the viewpoint through the image plane [S1]. It solves the hidden surface removal problem and enables realistic shading by identifying the first intersection between a ray and object surfaces, distinguishing itself from earlier scanline-based approaches by prioritizing ray-object intersection tests [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scene description (geometric primitives defining surfaces), a camera position (viewpoint), a camera orientation, and an image plane resolution.
*   **Output:** A 2D image where each pixel contains the color of the visible surface at that location, including shading and shadow information.

**Data Structures:**
*   **Scene Representation:** Objects are defined by their geometric boundaries (e.g., polygons, spheres). The specific data structure for the scene is not detailed in the primary source, but it must support intersection queries [S1].
*   **Image Buffer:** A 2D array storing the final color for each pixel.

**Algorithm Steps:**
1.  **Ray Generation:** For each pixel $(u, v)$ in the image plane, define a ray $r(t) = \mathbf{o} + t\mathbf{d}$, where $\mathbf{o}$ is the camera origin (viewpoint) and $\mathbf{d}$ is the direction vector from the origin through the pixel center [S1].
2.  **Intersection Testing:** For each generated ray, compute intersections with all objects in the scene. Determine the intersection parameter $t$ for each object.
3.  **Occlusion and Visibility:** Identify the intersection with the smallest positive $t$ (the closest hit). This point represents the visible surface for that pixel. Objects behind this point are occluded and do not contribute to the pixel color [S1].
4.  **Shading:** Compute the color of the hit point using a local lighting model. This involves calculating surface normals and applying illumination models (e.g., Phong shading) to simulate realistic lighting effects [S1].
5.  **Shadow Casting (Optional but implied by "realistic shading"):** To determine if the hit point is in shadow, cast secondary rays from the hit point towards light sources. If an object intersects this shadow ray before the light source, the point is in shadow [S1].
6.  **Color Assignment:** Assign the computed color (including ambient, diffuse, specular components, and shadow attenuation) to the corresponding pixel in the image buffer.

**Formulas:**
*   Ray equation: $r(t) = \mathbf{o} + t\mathbf{d}$ [S1].
*   The specific shading formula is not provided, but it relies on standard local lighting models [S1].

**Parameters:**
*   Image resolution (number of pixels).
*   Camera parameters (field of view, aspect ratio).
*   Shading model parameters (light positions, material properties).

**Edge Cases and Pitfalls:**
*   **Performance:** The algorithm is computationally expensive because it requires testing every ray against every object in the scene. This complexity is $O(N \cdot M)$ where $N$ is the number of pixels and $M$ is the number of objects [S1].
*   **Precision:** Floating-point precision errors in intersection calculations can lead to artifacts like z-fighting or incorrect occlusion.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def appels_ray_casting(scene, camera, image_width, image_height):
    """
    Implements Appel's Ray Casting Algorithm.
    
    Args:
        scene: List of geometric objects supporting intersection tests.
        camera: Object containing origin (o) and direction calculation method.
        image_width: Width of the output image in pixels.
        image_height: Height of the output image in pixels.
        
    Returns:
        image: 2D array of pixel colors.
    """
    image = create_empty_image(image_width, image_height)
    
    # Iterate over each pixel in the image
    for y in range(image_height):
        for x in range(image_width):
            # 1. Generate ray for the current pixel
            # Calculate ray origin and direction based on camera and pixel coordinates
            ray_origin = camera.get_origin()
            ray_direction = camera.get_direction(x, y)
            
            # 2. Find the closest intersection
            closest_t = infinity
            closest_object = None
            hit_point = None
            normal = None
            
            for object in scene:
                # Compute intersection of ray with object
                t, point, n = object.intersect(ray_origin, ray_direction)
                
                # Check if intersection is valid (positive t) and closer than current closest
                if t > 0 and t < closest_t:
                    closest_t = t
                    closest_object = object
                    hit_point = point
                    normal = n
            
            # 3. Determine color based on visibility and shading
            if closest_object is not None:
                # 4. Apply shading model
                # Calculate lighting contributions (ambient, diffuse, specular)
                # Check for shadows by casting shadow rays to light sources
                color = compute_shading(hit_point, normal, scene, camera)
                image[y, x] = color
            else:
                # No object hit, assign background color
                image[y, x] = camera.background_color
                
    return image
```

## Subtopics

### Ray Grouping
A technique to reduce computational complexity in Appel's algorithm by grouping rays originating from similar successive voxels into a single representative ray [S2]. It divides the input slice into blocks, measures intra-block variance, and groups voxels if the variance is below a user-defined threshold, casting only one ray for the group [S2]. If variance exceeds the threshold, voxels are ungrouped and individual rays are cast [S2]. This reduces the number of interpolations and ray casts while preserving image quality [S2]. The user-defined threshold determines the grouping sensitivity [S2].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$, where $N$ is the number of pixels and $M$ is the number of objects in the scene. This makes it computationally expensive for complex scenes [S1].
*   **Memory Complexity:** Depends on the scene representation and image buffer size. No specific memory optimization is mentioned in the primary source [S1].
*   **Strengths:** Conceptually simple, handles hidden surface removal naturally, and produces high-quality images with realistic shading and shadows [S1].
*   **Limitations:** Very slow for large scenes due to the exhaustive intersection testing. Not suitable for real-time rendering without significant optimizations or hardware acceleration [S1].
*   **When to use:** Suitable for offline rendering where image quality is prioritized over speed, or for simple scenes with few objects [S1].
*   **When not to use:** Unsuitable for real-time applications or scenes with a large number of objects due to high computational cost [S1].

## Background
Appel's Ray Casting Algorithm was developed by Arthur Appel at IBM in 1968 [S1]. It was one of the first computational approaches to realistic shading and solved the hidden surface removal problem by tracing rays from the viewpoint through the image plane [S1]. This method laid the foundational principles for later rendering techniques by prioritizing ray-object intersections over the scanline-based methods prevalent at the time [S1].

## Sources
[S1] ray casting, https://grokipedia.com/page/Ray_casting, fetched 2026-10-26
[S2] Different Ray-Casting Algorithm Implementations for Volume Rendering, https://doi.org/10.35940/ijitee.g1011.0597s20, fetched 2026-10-24
