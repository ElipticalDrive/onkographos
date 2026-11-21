# IDVR-Compositing

Also called: Importance-Driven Volume Rendering Compositing

## Overview
IDVR-Compositing is a post-local compositing step that assigns a view-dependent **sparseness** value to each object along a ray to modulate its visual prominence before global blending [S1, unofficial]. It solves the problem of displaying multiple objects with different importance levels by making occluded or less important objects sparse (transparent) while keeping important ones dense (visible) [S1, unofficial]. Unlike standard compositing which treats all samples equally, IDVR-Compositing modifies the color and opacity values based on an object's importance and its occlusion status within the image footprint [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of local compositing results $(c_i, \alpha_i)$ for each object $i$ along a ray, where $c_i$ is the object's color and $\alpha_i$ is its accumulated opacity. An importance value $I_i$ for each object. A pre-computed image footprint for the current view direction.
*   **Output:** Modified color and opacity values $(c'_i, \alpha'_i)$ ready for global compositing.

**Data Structures:**
*   **Footprint:** A 2D array or map corresponding to the image pixels, storing occlusion/visibility information for the current view. This must be recalculated whenever the view direction changes [S1, unofficial].
*   **Sparseness Map:** A value $s_i \in [0, 1]$ assigned to each object $i$ per ray. $s_i=1$ means the object is almost completely transparent; $s_i=0$ means it is completely visible [S1, unofficial].

**Steps:**
1.  **Create Footprint:** Generate a footprint of the image for the current view direction. This involves determining which objects intersect each ray and their relative positions [S1, unofficial].
2.  **Assign Sparseness:** For each ray, assign a sparseness value $s_i$ to each object $i$ based on its importance $I_i$ and occlusion status within the footprint [S1, unofficial].
    *   If an object is occluded by higher-importance objects, its sparseness increases (becomes more transparent).
    *   If an object is not occluded or has the highest importance, its sparseness decreases (becomes more visible) [S1, unofficial].
3.  **Modulate Values:** Modify the local compositing results $(c_i, \alpha_i)$ using the sparseness value $s_i$. The exact modulation formula is not explicitly defined in the provided notes, but it implies scaling the opacity and potentially the color contribution based on $s_i$ [S1, unofficial]. Let the modified values be $(c'_i, \alpha'_i)$.
4.  **Global Compositing:** Blend the modified color and opacity values $(c'_i, \alpha'_i)$ using the standard front-to-back compositing equation to produce the final image [S1, unofficial].

**Parameters:**
*   **Sparseness ($s_i$):** A value between 0 and 1. 1 means the object is almost completely transparent. 0 means the object should be completely visible [S1, unofficial].
*   **Importance ($I_i$):** A scalar value representing the visual importance of object $i$. Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def idvr_compositing(ray_objects, footprint, view_direction):
    """
    IDVR-Compositing step.
    ray_objects: list of dicts, each containing:
        - 'id': object identifier
        - 'importance': float, importance value I_i
        - 'color': vec3, local compositing color c_i
        - 'opacity': float, local compositing opacity alpha_i
    footprint: function or data structure that provides occlusion info for view_direction
    view_direction: vec3, current camera view direction

    Returns:
        list of dicts with 'color' and 'opacity' modified by sparseness.
    """
    # Step 1: Create Footprint (if not already cached for this view)
    # The footprint must be recalculated whenever the view direction changes [S1, unofficial].
    # This step determines occlusion relationships.
    occlusion_map = create_footprint(footprint, view_direction)

    modified_objects = []

    for obj in ray_objects:
        obj_id = obj['id']
        importance = obj['importance']
        color = obj['color']
        opacity = obj['opacity']

        # Step 2: Assign Sparseness
        # Sparseness depends on importance and occlusion within the footprint [S1, unofficial].
        # The specific algorithm for assigning s_i (e.g., Maximum Importance Projection)
        # is not fully detailed in the provided notes, but it uses occlusion_map and importance.
        # [G] Placeholder for the specific sparseness assignment logic (e.g., MImP).
        sparseness = assign_sparseness(obj_id, importance, occlusion_map)

        # Step 3: Modulate Values
        # Modify color and opacity based on sparseness [S1, unofficial].
        # [G] The exact modulation formula is not given. A common approach might be:
        # alpha_prime = opacity * (1 - sparseness)
        # color_prime = color * (1 - sparseness)
        # However, since the formula is not in the sources, we cannot implement it precisely.
        # We will assume a simple linear modulation for the sake of completeness,
        # but note that the exact method is unspecified.
        alpha_prime = opacity * (1.0 - sparseness)
        color_prime = color * (1.0 - sparseness)

        modified_objects.append({
            'id': obj_id,
            'color': color_prime,
            'opacity': alpha_prime
        })

    return modified_objects

def create_footprint(footprint_data, view_direction):
    """
    Placeholder for footprint creation.
    Returns an occlusion map based on view_direction.
    """
    # [G] Implementation details for footprint creation are not provided.
    # This would typically involve ray-object intersection tests.
    return {}

def assign_sparseness(obj_id, importance, occlusion_map):
    """
    Placeholder for sparseness assignment.
    Returns a value between 0 and 1.
    """
    # [G] Implementation details for sparseness assignment (e.g., MImP) are not provided.
    # MImP assigns lowest sparseness (0) to the object with max importance,
    # and highest sparseness (1) to others, if occluded.
    return 0.5 # Placeholder
```

## Subtopics

### Maximum Importance Projection (MImP)
A simple and fast approach to assigning sparseness values [S1, unofficial]. It is similar to Maximum Intensity Projection. Along a ray, the object with the maximum importance value gets the lowest sparseness value (0, most dense), so it is displayed in the most dense way. All other objects get the highest sparseness value (1, most transparent), so they are not rendered or are very faint [S1, unofficial]. A disadvantage is that it can result in only a cylindrical countersink at the position of the most important object, potentially hiding context [S1, unofficial].

### Visibility Preserving Importance Compositing
A method that guarantees constant visibility of a focus object regardless of the thickness of the context volume part in front of it [S2]. It ensures a constant fraction of visual information for the focus object [S2]. The desired level of sparseness for the context object in front of the focus object is denoted as $S_o$ [S2]. The context part in front of the focus is rendered separately using two-level volume rendering [S2]. All ray opacities of the occluding part are summed to compute the average per-ray opacity $\alpha_{avg}$ [S2]. The original accumulated opacity value is corrected such that the average per-ray opacity equals $S_o$ using the formula:
$$
\alpha_{accum\_new}(x) = \alpha_{accum}(x) \cdot \frac{S_o}{\alpha_{avg}}
$$
where $\alpha_{accum\_new}$ is the modified accumulated opacity, $\alpha_{accum}$ is the original density value, $S_o$ is the level of sparseness value of the context object, and $\alpha_{avg}$ is the average accumulated opacity value of the suppressed context part [S2]. Finally, a combination of both context parts takes place in the global compositing step [S2].

## Operating characteristics
*   **Time Complexity:** Depends on the footprint creation and sparseness assignment. MImP is described as "very simple but also very fast" [S1, unofficial]. VPImC requires separate rendering of the occluding part, which adds computational cost [S2].
*   **Memory Complexity:** Requires storage for the footprint and sparseness values per ray [S1, unofficial].
*   **Strengths:** Allows for view-dependent importance visualization, making important objects more visible while suppressing less important or occluded ones [S1, unofficial]. VPImC ensures constant visibility of focus objects [S2].
*   **Limitations:** MImP can lead to loss of context information [S1, unofficial]. Footprint must be recalculated for every view direction change [S1, unofficial]. The exact modulation of color and opacity by sparseness is not fully specified in the sources [S1, unofficial].

## Background
IDVR-Compositing was introduced in the context of Importance-Driven Volume Rendering (IDVR) [S1, unofficial]. The concept of assigning sparseness values to objects based on importance and occlusion was developed to enhance feature visibility in volume visualization [S2]. The method is part of a broader family of compositing techniques that modify local compositing results before global blending [S1, unofficial].

## Sources
[S1] Implementation of Importance-Driven Volume Rendering, https://www.cg.tuwien.ac.at/research/publications/2005/Thurner-2005-IVR/Thurner-2005-IVR-paper.pdf, fetched 2026-10-29 (unofficial)
[S2] Importance-Driven Feature Enhancement in Volume Visualization, https://doi.org/10.1109/tvcg.2005.62, fetched 2026-11-05
