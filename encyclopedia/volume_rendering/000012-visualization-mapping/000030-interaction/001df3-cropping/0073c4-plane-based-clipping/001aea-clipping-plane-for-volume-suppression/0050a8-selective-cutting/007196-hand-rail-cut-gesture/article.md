# Hand-rail cut gesture
## Overview
The hand-rail cut gesture is a whole-hand interaction technique that partitions volume data along a plane defined by the orientation and movement direction of the user's hand rail [S1, unofficial]. It builds on [Selective Cutting](14293-selective-cutting.md) by allowing the user to cut the volume layer by layer, excluding internal parts from being cut [S1, unofficial]. It sets itself apart from standard clipping by treating the cut as a physical incision via a sliding hand motion, providing immediate visual feedback through a drawn line [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: 2D touch coordinates of the hand rail (defined by the user's hand orientation and sliding trajectory) on a tabletop surface.
- Output: A partitioned volume object, split into separate pieces along the detected cutting plane.

**Data Structures:**
Not given in the sources. The interface maintains a list of volume pieces. When a cut is applied, the first intersected piece is split, and the resulting sub-pieces are added to the list.

**Steps:**
1. The user touches the tabletop surface with a hand, forming a "hand rail" (defined by the orientation of the hand).
2. The user slides the hand rail over the surface. The interface detects the orientation and movement direction of the hand rail [S1, unofficial].
3. The interface draws a red line on the surface as visual feedback, indicating the detected cutting direction [S1, unofficial].
4. The user lifts their hand.
5. The volume object is partitioned according to the plane defined by the red line [S1, unofficial].
6. Successive cuts can be applied by repeating steps 1-5, adding additional cutting planes [S1, unofficial].

**Pitfalls:**
- If a cut trajectory crosses multiple existing volume pieces, only the first piece crossed is partitioned. This is done to prevent unintentional cuts of deeper layers [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def handle_hand_rail_cut(hand_trajectory, volume_pieces):
    """
    Handles the hand-rail cut gesture.
    
    Args:
        hand_trajectory: List of (x, y) points representing the sliding hand rail.
        volume_pieces: List of volume objects currently in the scene.
        
    Returns:
        Updated list of volume pieces.
    """
    if not hand_trajectory:
        return volume_pieces

    # Step 2 & 3: Detect direction and draw feedback
    # The orientation and movement of the hand rail suggest the incision direction.
    # The interface draws a red line as visual feedback.
    cut_line = compute_cut_line_from_trajectory(hand_trajectory)
    draw_red_line(cut_line)

    # Step 4: User lifts hand (triggered by end of trajectory or explicit lift event)
    # Step 5: Partition the volume
    new_pieces = []
    cut_applied = False
    
    for piece in volume_pieces:
        # Check if the cut line intersects the piece
        if intersects(piece, cut_line):
            # Step 6: Partition only the first piece crossed
            # To avoid unintentional cuts of deeper layers
            if not cut_applied:
                # Split the piece along the cut plane
                p1, p2 = split_volume_along_plane(piece, cut_line)
                new_pieces.append(p1)
                new_pieces.append(p2)
                cut_applied = True
            else:
                # Subsequent pieces are not cut
                new_pieces.append(piece)
        else:
            new_pieces.append(piece)
            
    return new_pieces
```

## Subtopics

### Bimanual 'V'-shape cut
A gesture allowing simultaneous use of two hand rails to create a 'V'-shaped cut, centering on internal structure while retaining surroundings [S1, unofficial]. It is chosen when the user wants to focus on internal details while keeping context visible, a style often appearing in exploded-view visualization [S1, unofficial]. The cut locations are directly defined by hand positions, allowing flexible control over the angular opening [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
Not given in the sources. The complexity depends on the volume representation (e.g., voxel grid, octree) and the intersection testing algorithm. Splitting a volume piece likely involves traversing the data structure and duplicating or reassigning data points.

**Measured Performance:**
Not given in the sources.

**Strengths and Limitations:**
- Strengths: Natural interaction mimicking a surgical knife; immediate visual feedback; allows for layer-by-layer exploration [S1, unofficial].
- Limitations: Only cuts the first intersected piece, which may limit complex multi-layer cuts in a single gesture; requires precise hand tracking for accurate cut placement [S1, unofficial].

**When to Use:**
Use when the user needs to inspect internal structures of a volume by removing outer layers sequentially.

**When Not to Use:**
Not given in the sources.

## Background
The hand-rail cut gesture was proposed in the paper "Exploring Volume Visualization with Whole-hand Multitouch Gestures" [S1, unofficial]. It is part of a whole-hand multitouch interface designed for tabletop surfaces. The gesture is inspired by the natural motion of a surgical knife [S1, unofficial]. It is used in conjunction with other gestures such as the open palm gesture (to nullify cuts) and the hand-rail explode gesture (to create exploded views) [S1, unofficial].

## Sources
[S1] Exploring Volume Visualization with Whole-hand Multitouch Gestures, https://doi.org/10.2312/pe.pg.pg2013short.007-010, fetched 2026-10-26 (unofficial)
