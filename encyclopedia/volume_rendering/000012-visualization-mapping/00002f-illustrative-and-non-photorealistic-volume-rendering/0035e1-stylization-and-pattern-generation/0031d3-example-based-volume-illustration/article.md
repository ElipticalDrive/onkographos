# Example-Based Volume Illustration

Also called: Color Transfer for Illustration, Example-Based Illustrative Visualization, Example-Based Rendering, Volume Illustration Using Wang Cubes

## Overview
Example-Based Volume Illustration is a technique that generates multiple illustrative styles (e.g., pen and ink, watercolor) by combining 3D pattern textures with color transfer from example images [S1][S2, unofficial]. It solves the problem of simulating professional illustrator styles while simplifying user interaction, which previously required manual assignment of textured cubes per object [S2, unofficial]. It sets itself apart from siblings like [Stipple Volume Rendering](6321-stipple-volume-rendering.md) by using Wang Cubes for pattern synthesis and explicit color transfer algorithms rather than procedural stippling or curve generation [S1][S2, unofficial].

## How it works
**Inputs:**
1.  Target volume data (scalar field).
2.  Source illustration example (image).
3.  Segmentation data (optional): Object IDs or Opacity values.

**Outputs:**
*   Stylized volume rendering with transferred colors and patterns.

**Data Structures:**
*   **Wang Cubes:** A set of 3D textured cubes. Each cube has a "cube tiling" (face colors) and "cube contents" (internal pattern) [S1].
*   **Cube Sets:** A collection of Wang Cubes assigned to objects or regions.
    *   For segmented datasets: One cube set per Object ID [S1][S2, unofficial].
    *   For un-segmented datasets: Cube sets defined by opacity ranges [S1][S2, unofficial].

**Steps:**
1.  **Sampling:** The user samples the source illustration example and the target volume data [S2, unofficial].
2.  **Color Transfer:** Apply a color transfer algorithm to map color statistics from the source example to the target volume data. Standard algorithms include those by Reinhard et al. (2001) for color transfer between images and Welse et al. (2002) for transferring color to greyscale images [S1][S2, unofficial].
3.  **Assignment:** For each object or region, assign one set of textured Wang Cubes based on the segmentation (Object ID) or opacity [S1][S2, unofficial].
4.  **Synthesis:** Render the volume using the transferred colors and the assigned Wang Cubes. The rendering uses the cube tiling for pattern similarity and cube contents for style [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def example_based_volume_illustration(volume_data, source_image, segmentation=None):
    # Step 1: Sampling
    # User provides source_image and volume_data
    
    # Step 2: Color Transfer
    # Map color statistics from source_image to volume_data
    # Using Reinhard et al. or Welse et al. algorithm
    transferred_colors = transfer_color(source_image, volume_data)
    
    # Step 3: Assignment
    if segmentation is not None:
        # Segmented datasets: assign one cube set per Object ID
        cube_sets = assign_cube_sets_by_object_id(volume_data, segmentation)
    else:
        # Un-segmented datasets: assign cube sets by opacity
        cube_sets = assign_cube_sets_by_opacity(volume_data)
        
    # Step 4: Synthesis
    # Render using transferred colors and cube sets
    # Wang Cubes provide pattern via tiling and style via contents
    result = render_volume(volume_data, transferred_colors, cube_sets)
    
    return result
```

**Parameters:**
*   **User Interaction:** Limited to sampling the source example and selecting illustration examples [S1][S2, unofficial].
*   **Assumptions:** Simpler representation and similar object distributions are assumed to allow automatic transferring [S1][S2, unofficial].

**Edge Cases:**
*   **Segmentation:** If no segmentation is available, opacity-based assignment is used, which may be less precise for distinct objects [S1][S2, unofficial].
*   **Object Distribution:** The method assumes similar object distributions between the example and target for effective color transfer [S1][S2, unofficial].

## Subtopics

### Wang Cubes
A method for generating 3D patterns and textures using non-periodic indexing and shared cube tiling [S1].

### Color Transfer
Algorithms for mapping color statistics from a source image to a target volume [S1][S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Enables multiple styles (pen and ink, watercolor, etc.) with less storage by sharing one cube tiling [S1]. Simplifies user interaction by requiring only sampling and example selection [S1][S2, unofficial].
*   **Limitations:** Assumes similar object distributions between source and target for automatic color transfer [S1][S2, unofficial].
*   **When to use:** When simulating professional illustrator styles is desired with minimal user interaction [S2, unofficial].
*   **When not to use:** When precise per-object style control is needed without segmentation assumptions [S2, unofficial].

## Background
Originated in the context of medical illustration and flows, aiming to convey object properties through multiple illustrative styles [S1][S2, unofficial]. It builds on previous approaches that required one set of textured cubes per object or segmentation, proposing a simpler representation [S2, unofficial]. The technique is used in medical illustration and surgical simulation [S2, unofficial].

## Sources
[S1] Volume illustration for medicine and flows, https://doi.org/10.1145/1185657.1185694, fetched 2026-10-28
[S2] Illustrative Visualization for Medicine and Science, https://doi.org/10.2312/egt.20061068, fetched 2026-10-26 (unofficial)
