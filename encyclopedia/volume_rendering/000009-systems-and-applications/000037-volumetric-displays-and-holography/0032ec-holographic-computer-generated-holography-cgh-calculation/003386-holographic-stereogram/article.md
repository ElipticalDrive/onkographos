# Holographic Stereogram

Also called: HPO Stereogram, Horizontal-Parallax-Only Stereogram, Stereogram

## Overview
A Holographic Stereogram is a type of computer-generated hologram composed of a series of discrete 2D perspective views of an object scene [S1]. It produces a view-dependent image that presents the corresponding perspective view in each horizontally displaced direction, functioning similarly to lenticular or parallax barrier displays [S1]. Unlike full-parallax methods that compute complex interference patterns for all viewing angles, this method approximates the holographic fringe by combining these discrete views, allowing for computation at nearly interactive rates on specialized hardware [S1].

## How it works
The method generates a hologram by synthesizing a fringe pattern from a sequence of rendered perspective views.

**Inputs:**
*   3D object scene data (e.g., point cloud, mesh).
*   Camera parameters: lateral step size, number of views, field of view (FOV).
*   Hologram display parameters: vertical resolution, pixel pitch.

**Outputs:**
*   A 2D holographic fringe pattern (complex field or phase-only map) suitable for display on an SLM.

**Data Structures:**
*   **View Array:** A sequence of 2D images $V_i$, where $i$ is the view index. Each image has height $H$ (matching the hologram's vertical resolution) and width $W$ (determined by the horizontal sampling).
*   **Hologram Plane:** A 2D grid of complex amplitudes or phase values.

**Steps:**
1.  **View Generation:** Generate a sequence of 2D perspective views of the object scene. This is done by moving a virtual camera laterally in discrete steps [S1]. The number of views $N$ is determined by the desired horizontal Field of View (FOV) and the step size [S2].
2.  **Fringe Synthesis:** Combine the generated views to create the holographic fringe. The specific combination algorithm (e.g., interpolation, direct mapping) is not detailed in the primary source, but the result is a fringe pattern that encodes the horizontal parallax information [S1].

**Parameters:**
*   **Number of Views ($N$):** Must be large enough to create a high-quality image free from sampling artifacts [S1]. A larger FOV requires more views and increases computational complexity [S2].
*   **Camera Step Size:** Determines the horizontal displacement between consecutive views.

**Pitfalls:**
*   **Sampling Artifacts:** Using too few perspective views results in visible sampling artifacts and poor image quality [S1].
*   **Computational Cost:** The need for a large number of views limits computation speed, although specialized hardware can mitigate this [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_holographic_stereogram(object_scene, num_views, hologram_height, fov):
    """
    Generates a holographic stereogram from a 3D object scene.
    
    Args:
        object_scene: 3D representation of the object.
        num_views: Number of perspective views to generate.
        hologram_height: Vertical resolution of the hologram (and each view).
        fov: Horizontal field of view in degrees.
        
    Returns:
        hologram_fringe: 2D array representing the holographic fringe.
    """
    # Step 1: Generate sequence of view images
    views = []
    for i in range(num_views):
        # Calculate camera position based on lateral step
        # Step size is derived from fov and num_views
        angle = (i - num_views / 2) * (fov / num_views)
        camera_pos = get_camera_position(angle)
        
        # Render perspective view
        # Height is fixed to hologram_height
        view_image = render_view(object_scene, camera_pos, height=hologram_height)
        views.append(view_image)
        
    # Step 2: Combine images to generate fringe
    # The specific combination method is not detailed in the sources [S1]
    # This step typically involves mapping the views to the hologram plane
    # to form the interference-like pattern.
    hologram_fringe = combine_views_to_fringe(views)
    
    return hologram_fringe
```

## Subtopics

### View Extraction
View extraction is a technique used to retrieve individual perspective views from a pre-computed hologram, often for light field displays. A window with the resolution of the vertical axis of the hologram moves along the horizontal axis to acquire sub-holograms that simulate a pinhole camera view [S2]. The apertures extracted can be chosen densely enough to overlap, as increasing the number of views improves final rendering quality [S2]. However, the targeted Field of View (FOV) is crucial; larger FOVs require more views and increased computational complexity [S2].

## Operating characteristics
*   **Performance:** Allows computation at nearly interactive rates when implemented on specialized hardware [S1].
*   **Complexity:** Computational complexity increases with the number of views required, which is driven by the desired FOV and the need to avoid sampling artifacts [S1, S2].
*   **Quality:** Image quality is dependent on the density of the perspective views; insufficient views lead to sampling artifacts [S1].
*   **Use Case:** Suitable for applications requiring horizontal parallax with high computational efficiency, such as real-time 3D displays where full-parallax computation is too slow [S1].

## Background
The holographic stereogram approach was described in the context of interactive three-dimensional holographic displays [S1]. It is distinguished from interference-based approaches that employ supercomputers and simple images to achieve interactive display [S1]. The method is related to lenticular and parallax barrier displays but generates a holographic fringe [S1].

## Sources
[S1] Interactive three-dimensional holographic displays, https://doi.org/10.1145/271283.271312, fetched 2026-11-05
[S2] Three-dimensional rendering of computer-generated holograms acquired from point-clouds on light field displays, https://doi.org/10.1117/12.2237581, fetched 2026-11-04
