# Holographic Stereogram

Also called: HPO Stereogram, Horizontal-Parallax-Only Stereogram, Stereogram

## Overview
A Holographic Stereogram is a type of computer-generated hologram composed of a series of discrete 2D perspective views of an object scene [S1]. It produces a view-dependent image that presents the corresponding perspective view in each horizontally displaced direction, functioning similarly to lenticular or parallax barrier displays [S1]. Unlike full-parallax methods, it restricts depth cues to horizontal motion, significantly reducing computational load compared to full 3D wavefront reconstruction [S1].

## How it works
**Inputs:** A 3D object scene (represented as points, voxels, or meshes) and a set of camera positions.
**Outputs:** A 1D fringe pattern (or 2D array if vertical parallax is included, but typically 1D for HPO) representing the hologram plane.

**Data Structures:**
*   **View Images:** A sequence of 2D images $I_v(x, y)$, where $v$ indexes the view.
*   **Hologram Plane:** A 1D array (for HPO) or 2D array of complex amplitudes or phase values.

**Steps:**
1.  **View Generation:** Generate a sequence of perspective views by moving a virtual camera laterally in discrete steps across the object scene [S1]. The number of views $N_v$ depends on the desired Field of View (FOV) and the resolution of the display [S2].
2.  **Fringe Combination:** Combine these individual view images to generate the final holographic fringe for display [S1]. This involves encoding the perspective information into the interference pattern.

**Parameters:**
*   **Camera Step Size:** Determines the angular resolution. Smaller steps increase quality but increase computation [S1].
*   **Number of Views ($N_v$):** Must be large enough to create a high-quality image free from sampling artifacts [S1].

**Edge Cases and Pitfalls:**
*   **Sampling Artifacts:** A small number of perspective views leads to visible sampling artifacts (e.g., moiré patterns or discrete view jumps) [S1].
*   **Computational Load:** The need for a large number of views limits computation speed, although specialized hardware can achieve nearly interactive rates [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_holographic_stereogram(object_scene, camera_positions, hologram_resolution):
    """
    Generates a HPO Holographic Stereogram.
    
    Args:
        object_scene: 3D representation of the object.
        camera_positions: List of (x, y, z) coordinates for lateral camera movement.
        hologram_resolution: Resolution of the final hologram fringe.
        
    Returns:
        hologram_fringe: 1D array representing the hologram.
    """
    views = []
    
    # Step 1: Generate perspective views
    for pos in camera_positions:
        # Render 2D image from current camera position
        view_image = render_view(object_scene, pos)
        views.append(view_image)
        
    # Step 2: Combine views into fringe
    # Note: Specific encoding algorithm (e.g., multiplexing, interference calculation) 
    # is not detailed in sources [S1, S2]. This step combines the views.
    hologram_fringe = combine_views_into_fringe(views, hologram_resolution)
    
    return hologram_fringe
```

## Subtopics

### View Extraction
A technique used to extract different views from a computed hologram, often for display on light field displays. A window with the resolution of the vertical axis of the hologram moves along the horizontal axis to acquire sub-holograms that simulate a pinhole camera view [S2]. The apertures extracted can be chosen densely enough to overlap, as increasing the number of views improves final rendering [S2]. However, the Field of View (FOV) is crucial; larger FOVs require more views and increased computational complexity [S2].

## Operating characteristics
*   **Time Complexity:** Scales with the number of views $N_v$ and the resolution of each view. The need for a large number of views to avoid artifacts limits speed [S1].
*   **Memory Complexity:** Proportional to the number of views stored during generation.
*   **Performance:** Allows computation at nearly interactive rates when implemented on specialized hardware [S1].
*   **Strengths:** Significantly faster than full-parallax CGH methods; suitable for real-time applications with hardware acceleration [S1].
*   **Limitations:** Limited to horizontal parallax; requires many views for high quality, which can be computationally expensive without specialized hardware [S1].
*   **When to Use:** When horizontal depth cues are sufficient and real-time or near-real-time performance is required.
*   **When Not to Use:** When full 3D parallax (vertical and horizontal) is necessary.

## Background
The Holographic Stereogram approach was introduced as a method to achieve interactive 3D holographic displays by breaking from traditional simulation of optical holographic interference [S1]. It leverages computer graphics to generate perspective views, which are then combined to form the hologram [S1]. This approach was noted for its potential to achieve interactive rates with specialized hardware [S1].

## Sources
[S1] Interactive three-dimensional holographic displays, https://doi.org/10.1145/271283.271312, fetched 2026-11-05
[S2] Three-dimensional rendering of computer-generated holograms acquired from point-clouds on light field displays, https://doi.org/10.1117/12.2237581, fetched 2026-11-04
