# Midpoint UV Calculation

Also called: Midpoint UV Vector

## Overview
Midpoint UV Calculation computes the UV coordinates of the midpoint plane within a voxel traversal step to determine height and lighting variations for surface rendering [S1, unofficial]. It sets apart from sibling methods by explicitly calculating a geometric midpoint between entry and exit UV states to drive height maps, whereas siblings like [Corner-based Voxel Volume Calculation](14665-corner-based-voxel-volume-calculation.md) rely on corner interpolation and [Voxel DDA Raymarching with Neighbor-Averaged Entry Interpolation](14668-voxel-dda-raymarching-with-neighbor-averaged-entry-interpola.md) focus on entry-point smoothing [S1, unofficial].

## How it works
**Inputs:**
- `southwest`, `southeast`, `northwest`, `northeast`: UV coordinate vectors (2D) representing the four corners of the current voxel face or traversal step [S1, unofficial].
- `center`: A scalar or vector representing the central reference point of the voxel [S1, unofficial].
- `midpointUV`: A pre-loaded 2D vector containing initial midpoint data [S1, unofficial].

**Outputs:**
- Updated `midpointUV` vector containing the calculated midpoint coordinates [S1, unofficial].
- Derived height values for lighting/terrain generation [S1, unofficial].

**Data Structures:**
- `midpoints`: A 2D vector structure holding X and Y components [S1, unofficial].

**Steps:**
1. Initialize the midpoint vector by loading existing midpoint data:
   $$ \text{midpoints} = \text{load2}() $$
2. Calculate the Southwest midpoint UV by averaging the Southwest and Southeast UV coordinates:
   $$ \text{midpoints}.x_{sw} = \frac{\text{southwest}.x + \text{southeast}.x}{2} $$
   $$ \text{midpoints}.y_{sw} = \frac{\text{southwest}.y + \text{southeast}.y}{2} $$
3. Calculate the Northwest midpoint UV by averaging the Northwest and Northeast UV coordinates:
   $$ \text{midpoints}.x_{nw} = \frac{\text{northwest}.x + \text{ntheast}.x}{2} $$
   $$ \text{midpoints}.y_{nw} = \frac{\text{northwest}.y + \text{ntheast}.y}{2} $$
4. Apply Y min/max constraints to the midpoint Y component. This constraint is reused for both the Southwest and Northwest midpoints to ensure consistency [S1, unofficial].
5. Determine the final midpoint X and Y based on the constrained Y value:
   - If $\text{midpointUV}.y < 0.5$:
     $$ \text{midpointX} = \text{center} + 0.5 $$
   - Otherwise:
     $$ \text{midpointY} = \text{center} $$
6. The resulting `midpointUV` is used to calculate map height and lighting variations [S1, unofficial].

**Pitfalls:**
- Rendering only functions correctly above the calculated midpoint. Below the midpoint, the ray traversal lacks proper height calculations, causing incorrect visual output [S1, unofficial].
- The method assumes a specific traversal direction; entering from above and exiting below without explicit quadrant handling can lead to inaccurate midpoint values [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_midpoint_uv(southwest, southeast, northwest, northeast, center):
    # Step 1: Initialize midpoint vector
    midpoints = load2() # Implementation detail: loads initial state
    
    # Step 2: Calculate Southwest midpoint UV
    midpoints.x_sw = (southwest.x + southeast.x) / 2.0
    midpoints.y_sw = (southwest.y + southeast.y) / 2.0
    
    # Step 3: Calculate Northwest midpoint UV
    midpoints.x_nw = (northwest.x + northeast.x) / 2.0
    midpoints.y_nw = (northwest.y + northeast.y) / 2.0
    
    # Step 4: Apply Y min/max constraints (reused for both)
    # Note: Exact min/max values are not specified in sources [G]
    midpoints.y_sw = clamp(midpoints.y_sw, min_y, max_y)
    midpoints.y_nw = clamp(midpoints.y_nw, min_y, max_y)
    
    # Step 5: Determine final midpoint based on Y constraint
    # The source implies a conditional logic based on midpointUV.y
    # Assuming midpointUV refers to the current state or a specific component
    if midpoints.y_sw < 0.5:
        midpoint_x = center + 0.5
        midpoint_y = midpoints.y_sw # Or center, source phrasing is ambiguous
    else:
        midpoint_x = midpoints.x_sw # Source says "midpoints.x plus Center" then "goes to center"
        midpoint_y = center
        
    # Return the calculated midpoint UV
    return midpoint_x, midpoint_y
```

## Subtopics

### Midpoint Plane Y-Height Interpolation
Calculates the exact Y height where a ray intersects the midpoint plane using linear interpolation between entry and exit Y values [S1, unofficial]. It determines the entry Y value ($Y_{entry}$) and exit Y value ($Y_{exit}$) for the current voxel step [S1, unofficial]. The Y difference is calculated as $\text{diff} = Y_{entry} - Y_{exit}$ [S1, unofficial]. The Y offset at the midpoint is found by multiplying the difference by the normalized X position ($t$, typically $0.5$) [S1, unofficial]. The intersection height is obtained by subtracting the offset from the entry Y height: $Y_{intersect} = Y_{entry} - (Y_{entry} - Y_{exit}) \times t$ [S1, unofficial]. A key pitfall is that midpoint accuracy degrades if the ray enters from above and exits below without explicit quadrant handling or proper step direction tracking [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Constant time $O(1)$ per voxel step, involving basic arithmetic operations (addition, division, comparison) [S1, unofficial].
- **Memory Complexity:** Constant space $O(1)$, requiring storage for a few scalar values and vectors [S1, unofficial].
- **Strengths:** Provides a geometrically intuitive midpoint for height and lighting calculations, simplifying the transition between voxel faces [S1, unofficial].
- **Limitations:** Fails below the calculated midpoint, requiring additional logic for full volume traversal [S1, unofficial]. Accuracy is sensitive to ray entry/exit angles [S1, unofficial].
- **When to use:** Suitable for terrain rendering or surface approximation where height variation at the midpoint of a voxel step is critical [S1, unofficial].
- **When not to use:** Not suitable for full volumetric density integration or cases requiring accurate height calculation below the midpoint plane without additional quadrant handling [S1, unofficial].

## Background
Originating from informal discussions on voxel DDA raymarching techniques, this method was documented in a video tutorial by an unofficial source [S1, unofficial]. It is part of a broader effort to improve surface rendering in voxel-based environments by introducing geometric midpoints for height mapping [S1, unofficial].

## Sources
[S1] Smooth terrain with voxel DDA raymarching, https://www.youtube.com/watch?v=CxoC0E7inUg, fetched 2026-10-25 (unofficial)
