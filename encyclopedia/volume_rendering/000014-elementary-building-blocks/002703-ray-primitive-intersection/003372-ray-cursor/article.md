# Ray Cursor

Also called: Ray Cursor Selection, laser gun selection

## Overview
Ray Cursor is a 3D selection technique where a virtual ray is emitted from a user-controlled source (typically the hand or finger) to identify and select objects in a scene [S1][S2, unofficial]. It solves the problem of selecting specific models in volumetric or immersive 3D environments where traditional 2D input devices are insufficient [S1][S2, unofficial]. Unlike [Ray Occlusion Test](5670-ray-occlusion-test.md), which defines the ray by the vector between the eye and hand, Ray Cursor allows the user to independently control the ray's origin and orientation, offering greater flexibility but requiring higher angular accuracy [S2, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** User hand position $\mathbf{p}_{hand}$ and orientation (defining direction $\mathbf{d}$); scene geometry (list of objects).
*   **Output:** Selected object identifier; visual feedback (highlighting).

**Data Structures**
*   **Ray:** Defined by origin $\mathbf{o} = \mathbf{p}_{hand}$ and normalized direction $\mathbf{d}$ [S2, unofficial].
*   **Intersection List:** A list of intersection points $\{ \mathbf{i}_1, \mathbf{i}_2, \dots, \mathbf{i}_n \}$ sorted by distance from $\mathbf{o}$ [S1][S2, unofficial].

**Steps**
1.  **Emit Ray:** The system generates a ray starting at the user's hand position $\mathbf{p}_{hand}$ with direction $\mathbf{d}$ determined by hand orientation [S2, unofficial].
2.  **Intersect:** Perform ray-object intersection tests against all scene objects [S1][S2, unofficial].
3.  **Sort:** Sort intersection points by distance from the ray origin $\mathbf{o}$ [S1][S2, unofficial].
4.  **Highlight:** Highlight the closest intersected object (first in the sorted list) by changing its color (e.g., to yellow) [S1].
5.  **Cycle (Optional):** If multiple objects are intersected, allow the user to cycle through them by moving the hand forward (selecting further objects) or backward (selecting closer objects) [S1].
6.  **Select:** Upon a trigger gesture, select the currently highlighted object (turning it blue) [S1].

**Formulas**
*   Ray Equation: $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$, where $t \ge 0$ [G].
*   Distance Metric for Cone Disambiguation (Spotlight Selection): If using a cone instead of a ray, select the object based on the distance from the target to the apex and central axis of the cone [S2, unofficial].

**Parameters**
*   **Ray Color:** Yellow for highlighting, Blue for selected [S1].
*   **Cone Apex:** Located at the user's hand [S2, unofficial].

**Edge Cases and Pitfalls**
*   **Angular Accuracy:** Selecting small and distant objects is difficult due to the required precision in hand orientation [S2, unofficial].
*   **Multiple Intersections:** A ray may intersect multiple objects. The system must disambiguate which object to select [S1][S2, unofficial].
*   **Occlusion:** If the user cannot see the hand or ray clearly, selection may fail [S1][S2, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_cursor_select(hand_position, hand_orientation, scene_objects):
    # Define ray origin and direction
    ray_origin = hand_position
    ray_direction = normalize(hand_orientation)
    
    # Find all intersections
    intersections = []
    for obj in scene_objects:
        hit_point, hit_distance = intersect_ray_object(ray_origin, ray_direction, obj)
        if hit_point is not None:
            intersections.append((hit_distance, obj, hit_point))
    
    # Sort by distance
    intersections.sort(key=lambda x: x[0])
    
    if not intersections:
        return None
    
    # Highlight the closest object
    closest_distance, closest_obj, closest_point = intersections[0]
    highlight_object(closest_obj, color="yellow")
    
    # If cycling is enabled, allow user to select further objects
    # This is handled by user input loop, not shown here
    return closest_obj
```

## Subtopics

### Spotlight Selection
Uses a cone instead of a ray to accommodate angular error. Selects the object closest to the cone's central axis and apex. Choose when high angular precision is difficult [S2, unofficial].

### Occlusion Selection
Defines the ray by the vector between the eye and hand. Choose when the user wants to select objects behind their hand [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ for intersection tests against $N$ objects, plus $O(M \log M)$ for sorting $M$ intersections [G].
*   **Memory Complexity:** $O(M)$ to store intersection points [G].
*   **Strengths:** Intuitive for 3D environments; allows selection of near and distant objects [S2, unofficial].
*   **Limitations:** High angular accuracy required for small/distant objects; multiple intersections require disambiguation [S2, unofficial].
*   **When to Use:** Immersive 3D environments with volumetric displays or VR [S1][S2, unofficial].
*   **When Not to Use:** 2D interfaces; scenes with high object density where disambiguation is complex [S2, unofficial].

## Background
Ray Cursor selection was implemented by Liang and Green in 1994 as "laser gun" selection for immersive 3D environments [S2, unofficial]. It was designed to allow users to select both near and distant objects by emitting a ray from the user's hand [S2, unofficial]. The technique was later explored in volumetric displays, where it was found to be more effective than other techniques for static selection tasks [S1]. Hinckley suggested augmenting the technique with cycling through intersection points [S2, unofficial].

## Sources
[S1] Multi-finger gestural interaction with 3d volumetric displays, https://doi.org/10.1145/1029632.1029644, fetched 2026-10-28
[S2] Interaction with volumetric displays, http://hdl.handle.net/1807/16733, fetched 2026-10-26 (unofficial)
