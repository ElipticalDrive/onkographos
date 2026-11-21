# Image Plane Selection

Also called: one-eyed cursor, occlusion selection, ray cursor, laser gun selection

## Overview
Image plane selection is a technique where 3D objects are projected onto a 2D image plane, and a cursor defines a perpendicular ray to select the first intersected object [S1, unofficial]. It extends desktop mouse interaction to 3D environments by using occlusion or specific hand configurations as the selection mechanism [S1, unofficial]. Unlike direct 3D cursors that specify X, Y, and Z coordinates explicitly, this method relies on the visual alignment of the user's input device (hand, finger, or virtual pointer) with the target object [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** User hand position/orientation (tracked via gloves, optical sensors, or head-tracking), camera viewpoint (eye position), and 3D scene geometry.
- **Outputs:** A selected object or set of objects identified by intersection with the selection ray or volume.

**Data Structures:**
- **Hand Pose:** Vector $H$ representing the position of the relevant hand feature (e.g., fingertip, palm center).
- **Eye Position:** Vector $E$ representing the user's eye position.
- **Scene Geometry:** List of 3D primitives or volume data.

**Steps:**
1. **Define Selection Ray/Volume:**
   - **Standard Image Plane:** Define a ray $R$ starting at the eye $E$ and passing through the cursor position $C$ on the image plane. The direction is $D = C - E$.
   - **One-Eyed Cursor (Stereoscopic):** For stereoscopic displays, define the image plane as the viewpoint from only a single eye to create a "one-eyed cursor" [S1, unofficial]. This avoids ambiguity in depth perception [S1, unofficial].
   - **Aperture-Based:** Cast a cone from the eye $E$ through the user's finger. The distance between the finger and the eye controls the cone's divergence angle [S1, unofficial].
   - **Head Crusher:** Define a selection volume (typically a frustum or slab) between the index finger and thumb [S1, unofficial].
   - **Sticky Finger:** Define a ray from the eye $E$ through the tip of the index finger [S1, unofficial].
   - **Lifting Palm:** Define a plane or volume underneath the flattened palm [S1, unofficial].
   - **Framing Hands:** Use both hands to define a bounding box around the object using thumbs and index fingers [S1, unofficial].
   - **Icon Wall:** Use a virtual laser pointer from the hand to a point on the projection screen plane [S2, unofficial].

2. **Intersection Test:**
   - Cast the ray or test the volume against the 3D scene geometry.
   - For **Ray Cursor** (Laser Gun): The first object intersected by the ray is selected [S1, unofficial].
   - For **Occlusion Selection**: Objects visually lying behind the hand-attached cursor are selected [S1, unofficial].
   - For **Aperture-Based**: If the ray intersects multiple objects, use device orientation for disambiguation. Select the object whose orientation is most closely matched by the input device [S1, unofficial].

3. **Selection Confirmation:**
   - Click or gesture confirmation selects the identified object [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def select_object(hand_pose, eye_position, scene):
    # hand_pose includes fingertip position F and orientation O
    # eye_position is E
    
    # Define ray direction based on technique
    # Example: Sticky Finger
    ray_origin = eye_position
    ray_direction = normalize(hand_pose.fingertip - eye_position)
    
    # Cast ray against scene
    intersections = []
    for object in scene.objects:
        t = ray_intersect(ray_origin, ray_direction, object)
        if t is not None and t > 0:
            intersections.append((t, object))
    
    # Sort by distance
    intersections.sort(key=lambda x: x[0])
    
    if not intersections:
        return None
    
    # Handle ambiguity for aperture-based selection
    if len(intersections) > 1 and technique == "aperture":
        # Disambiguate using device orientation
        best_object = None
        best_score = -infinity
        for t, obj in intersections:
            score = orientation_match(obj.normal, hand_pose.orientation)
            if score > best_score:
                best_score = score
                best_object = obj
        return best_object
    
    # Return first intersected object
    return intersections[0][1]
```

**Parameters:**
- **Cone Divergence Angle:** Controlled by the distance between the finger and the eye [S1, unofficial]. Not given in the sources.
- **Disambiguation Threshold:** Not given in the sources.

**Edge Cases and Pitfalls:**
- **Multiple Intersections:** In aperture-based selection, multiple objects may intersect the ray. Orientation matching is used for disambiguation [S1, unofficial].
- **Stereoscopic Ambiguity:** Without the one-eyed cursor technique, stereoscopic displays may lack a well-defined image plane, leading to selection errors [S1, unofficial].
- **Occlusion:** Users must visually align the cursor with the target, which can be difficult in cluttered scenes [S1, unofficial].

## Subtopics

### Icon Wall Interaction
Icons are drawn in the plane of the projection screen to allow both head-tracked and non-headtracked viewers to clearly view them at all times [S2, unofficial]. Users interact with icons by pointing at them with a virtual laser pointer and clicking a button. Check boxes change state on click; dials are changed by clicking and twisting the hand [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of objects in the scene for ray casting $O(N)$, where $N$ is the number of objects.
- **Memory Complexity:** $O(1)$ for the selection ray/volume, plus storage for scene geometry.
- **Strengths:** Intuitive mapping from 2D desktop interaction; effective for 3D environments when combined with occlusion cues [S1, unofficial].
- **Limitations:** Requires precise hand tracking; ambiguous in cluttered scenes without disambiguation mechanisms [S1, unofficial].
- **When to Use:** When users are familiar with 2D mouse interaction; in immersive environments where direct 3D cursors are difficult to use [S1, unofficial].
- **When Not to Use:** When high precision is required in dense scenes without orientation disambiguation [S1, unofficial].

## Background
Originated from desktop mouse interaction, where objects are projected onto a 2D image plane [S1, unofficial]. Extended to 3D displays by Ware and Lowther (1997) with the "one-eyed cursor" technique [S1, unofficial]. Liang and Green (1994) implemented a "laser gun" selection mechanism [S1, unofficial]. Forsberg et al. (1996) explored aperture-based selection [S1, unofficial]. Pierce et al. (1997) explored several occlusion selection techniques including Head Crusher, Sticky Finger, Lifting Palm, and Framing Hands [S1, unofficial].

## Sources
[S1] Interaction with volumetric displays, http://hdl.handle.net/1807/16733, fetched 2026-10-26 (unofficial)
[S2] IEEE Visualization 2004 Contest: User Friendly Volume Data Set Exploration in the Cave, http://vis.computer.org/vis2004contest/brown/, fetched 2026-10-28 (unofficial)
