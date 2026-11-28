# Orthographic Crosshair Projection

Also called: 3D Crosshair, Crosshair Drawing, Slice Cursor Lines

## Overview
Orthographic Crosshair Projection renders horizontal and vertical lines representing the intersection of the current 2D slice plane with the X, Y, and Z axes in world space, projected to the orthographic view [S1, unofficial]. It provides spatial orientation cues for the user by indicating the current slice position relative to the volume's coordinate system. Unlike [2D Volume Cross-Section Rendering](1764-2d-volume-cross-section-rendering.md), which focuses on the slice content, this method overlays geometric indicators. It differs from [Curved Plane Reconstruction](22024-curved-plane-reconstruction.md) by strictly adhering to axis-aligned world-space intersections rather than following anatomical curves.

## How it works
**Inputs:**
- Volume shape: $S = (sh[0], sh[1], sh[2])$ (voxel dimensions in X, Y, Z).
- Voxel spacing: $sp = (sp[0], sp[1], sp[2])$ (world units per voxel).
- Slice orientation: $O \in \{AXIAL, CORONAL, SAGITTAL\}$.
- Current slice index: $idx$ (voxel index along the slicing axis).

**Outputs:**
- A set of line primitives (polygons/lines) representing the crosshair arms, positioned in world coordinates and ready for orthographic projection.

**Data Structures:**
- `vtkPoints`: Stores the 3D coordinates of the line endpoints.
- `vtkCellArray`: Stores the connectivity (pairs of point indices defining lines).
- `vtkPolyData`: Combines points and cells into a renderable geometry object.

**Steps:**
1. **Compute World Extents:** Calculate the total physical dimensions of the volume.
   $$ ex = sh[0] \times sp[0] $$
   $$ ey = sh[1] \times sp[1] $$
   $$ ez = sh[2] \times sp[2] $$
   [S1, unofficial]

2. **Determine Slice Plane Constant:** Identify the axis being sliced and the constant world coordinate for the current slice.
   - If $O = AXIAL$ (scroll Z): The slice is at constant $Z$.
     $$ pz = idx \times sp[2] $$
     The crosshair consists of a line spanning X at $Y=0$ (or center) and Z=0? No, the crosshair represents the intersection with axes. The source specifies: "AXIAL sets lines at constant pz spanning X/Y". This implies two lines: one along X at $Y=0, Z=pz$? Or spanning the full X and Y ranges at the slice plane?
     Re-reading [S1, unofficial]: "Draws horizontal and vertical lines representing the intersection of the current slice plane with the X, Y, and Z axes... AXIAL sets lines at constant pz spanning X/Y".
     Interpretation: For an Axial slice at $Z=pz$, the crosshair shows where the slice plane intersects the X-axis and Y-axis? No, it spans X and Y. This usually means a "plus" sign centered on the slice position in the X-Y plane, but projected in 3D.
     Actually, standard crosshairs in medical viewers show the intersection of the current slice with the other two orthogonal planes.
     Let's stick strictly to [S1, unofficial]'s step description: "AXIAL sets lines at constant pz spanning X/Y". This likely means two lines:
     1. Line along X axis: $Y=0, Z=pz$, $X \in [0, ex]$.
     2. Line along Y axis: $X=0, Z=pz$, $Y \in [0, ey]$.
     However, "spanning X/Y" might mean the lines are drawn across the entire volume extent in the respective dimensions.
     Let's look at the code. The code creates a single line from (0,0,0) to (1,0,0). This is a template. The logic in the step description defines the endpoints.
     
     Refined Step 2:
     - If $O = AXIAL$ (Z-slice):
       - Line 1 (X-axis intersection): From $(0, 0, pz)$ to $(ex, 0, pz)$.
       - Line 2 (Y-axis intersection): From $(0, 0, pz)$ to $(0, ey, pz)$.
       - *Note*: The source says "spanning X/Y". It does not explicitly state the centering. Standard crosshairs are often centered on the slice cursor. However, [S1, unofficial] says "intersection ... with the X, Y, and Z axes". The axes intersect at $(0,0,0)$. If the slice is at $pz$, it intersects the X-axis at $(x, 0, pz)$? No, the X-axis is $y=0, z=0$. The plane $z=pz$ is parallel to the X-axis if $pz \neq 0$. It never intersects the X-axis unless $pz=0$.
       
     Let's re-read carefully: "intersection of the current slice plane with the X, Y, and Z axes".
     If the slice is Axial ($Z=pz$), it is parallel to the XY plane. It intersects the Z-axis at $(0,0,pz)$. It does *not* intersect the X or Y axes (unless $pz=0$).
     
     Perhaps "intersection with the axes" refers to the *projection* or the *grid lines*?
     Or perhaps it means the lines drawn *on* the slice that correspond to the X and Y axes directions?
     
     Let's look at the "AXIAL sets lines at constant pz spanning X/Y" again.
     If it spans X, it is a line $Y=const, Z=pz$. If it spans Y, it is $X=const, Z=pz$.
     Usually, crosshairs are centered on the current view. But the source doesn't mention a center point. It mentions "voxel indices (i,j,k)".
     
     Let's assume the standard implementation where the crosshair indicates the slice position in the other two views.
     Actually, [S1, unofficial] code snippet is generic. The text description is the key.
     "AXIAL sets lines at constant pz spanning X/Y".
     This implies:
     - One line parallel to X-axis at $Y=0$ (or some reference) and $Z=pz$.
     - One line parallel to Y-axis at $X=0$ (or some reference) and $Z=pz$.
     
     Given the ambiguity of "spanning X/Y" without a center, and the fact that it's called "intersection with axes", if $pz \neq 0$, there is no intersection with X/Y axes.
     
     *Correction*: In many viewers, the crosshair is a "plus" sign centered on the slice. But the source text says "intersection ... with the X, Y, and Z axes".
     If the source is literal, and $pz \neq 0$, the plane $Z=pz$ only intersects the Z-axis.
     
     However, the code creates a line from 0,0,0 to 1,0,0. This is a unit vector.
     
     Let's look at [S2, unofficial]. It mentions "in-plane crosshair arm endpoints". This implies the crosshair lies *in* the slice plane.
     
     Let's assume the "intersection" description in [S1, unofficial] is slightly loose and refers to the *grid lines* or *axis-aligned lines* passing through the slice center or origin.
     
     *Decision*: I will describe the method as generating lines parallel to the axes within the slice plane, based on the "spanning" description. I will note the coordinate calculation.
     
     Step 2 (Revised for Implementation):
     Calculate the world coordinate of the slice:
     - If $O = AXIAL$: $z_{slice} = idx \times sp[2]$.
     - If $O = CORONAL$: $y_{slice} = idx \times sp[1]$.
     - If $O = SAGITTAL$: $x_{slice} = idx \times sp[0]$.
     
     Generate lines:
     - If $O = AXIAL$:
       - Line X: $(0, 0, z_{slice})$ to $(ex, 0, z_{slice})$? Or centered?
       - The source says "spanning X/Y". I will assume spanning the full extent for simplicity unless a center is provided. *However*, standard crosshairs are centered.
       - Let's look for clues in [S2, unofficial]. "Projects in-plane crosshair arm endpoints...".
       - Without a center coordinate, I cannot implement a centered crosshair.
       - *Gap Fill [G]*: In standard medical imaging viewers (like ITK-SNAP, referenced by color), crosshairs are centered on the current slice position in the *other* two dimensions? No, the crosshair *is* the slice position.
       - Actually, the "Orthographic Crosshair" often refers to the lines drawn on the *other* two orthogonal views to show where the current slice is. But this article is under "2D Volume Cross-Section Rendering", implying it's drawn *on* the current slice.
       - If drawn on the current slice, it usually shows the X and Y axes of the slice.
       - I will implement the lines spanning the full extent of the volume in the respective dimensions, anchored at the origin (0,0) of the slice plane, as implied by "intersection with axes" (if we consider the slice plane's local axes).
       - *Wait*, if it's the intersection with the *world* axes, and the slice is at $z=pz$, it intersects the Z-axis.
       
       Let's stick to the most robust interpretation of "spanning X/Y":
       For AXIAL:
       - Line 1: Parallel to X, at $Y=0$ (or center?), $Z=z_{slice}$.
       - Line 2: Parallel to Y, at $X=0$ (or center?), $Z=z_{slice}$.
       
       Given the lack of center info, I will define the lines from $(0,0,z_{slice})$ to $(ex,0,z_{slice})$ and $(0,0,z_{slice})$ to $(0,ey,z_{slice})$. This represents the intersection of the slice plane with the planes $Y=0$ and $X=0$? No.
       
       Let's look at the code again. `pts.InsertNextPoint(0, 0, 0)` to `(1, 0, 0)`. This is a line along X.
       
       I will provide the formula for the endpoints based on the "spanning" description, assuming the lines start from the origin of the volume's coordinate system within the slice plane.
       
       *Refined Step 2*:
       - If $O = AXIAL$:
         - Point A: $(0, 0, pz)$
         - Point B: $(ex, 0, pz)$
         - Point C: $(0, 0, pz)$
         - Point D: $(0, ey, pz)$
       - If $O = CORONAL$:
         - $py = idx \times sp[1]$
         - Line along X: $(0, py, 0)$ to $(ex, py, 0)$
         - Line along Z: $(0, py, 0)$ to $(0, py, ez)$
       - If $O = SAGITTAL$:
         - $px = idx \times sp[0]$
         - Line along Y: $(px, 0, 0)$ to $(px, ey, 0)$
         - Line along Z: $(px, 0, 0)$ to $(px, 0, ez)$

3. **Construct Geometry:**
   - Create a `vtkPoints` object.
   - Insert the calculated endpoints.
   - Create a `vtkCellArray`.
   - Insert lines connecting the points (e.g., Point 0 to Point 1, Point 0 to Point 2).
   - Create a `vtkPolyData` object and set the points and cells.

4. **Style Properties:**
   - Set line width to $1.2$ [S1, unofficial].
   - Set line stipple pattern to `0xF0F0` for a dashed appearance [S1, unofficial].
   - Set line stipple repeat factor to $1$ [S1, unofficial].
   - Set color (e.g., blue `(0.2, 0.5, 1.0)` as per example) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def create_crosshair_lines(shape, spacing, orientation, slice_index):
    """
    shape: tuple (sh_x, sh_y, sh_z)
    spacing: tuple (sp_x, sp_y, sp_z)
    orientation: 'AXIAL', 'CORONAL', or 'SAGITTAL'
    slice_index: int
    """
    sh_x, sh_y, sh_z = shape
    sp_x, sp_y, sp_z = spacing
    
    ex = sh_x * sp_x
    ey = sh_y * sp_y
    ez = sh_z * sp_z
    
    points = []
    
    if orientation == 'AXIAL':
        pz = slice_index * sp_z
        # Line along X at Y=0, Z=pz
        points.append((0, 0, pz))
        points.append((ex, 0, pz))
        # Line along Y at X=0, Z=pz
        points.append((0, 0, pz))
        points.append((0, ey, pz))
        
    elif orientation == 'CORONAL':
        py = slice_index * sp_y
        # Line along X at Y=py, Z=0
        points.append((0, py, 0))
        points.append((ex, py, 0))
        # Line along Z at X=0, Y=py
        points.append((0, py, 0))
        points.append((0, py, ez))
        
    elif orientation == 'SAGITTAL':
        px = slice_index * sp_x
        # Line along Y at X=px, Z=0
        points.append((px, 0, 0))
        points.append((px, ey, 0))
        # Line along Z at X=px, Y=0
        points.append((px, 0, 0))
        points.append((px, 0, ez))
        
    return points

def build_polydata(points):
    """
    Converts a list of 4 points (2 lines sharing a start point) into VTK-like structure.
    """
    # In a real implementation, this would use vtk.vtkPoints and vtkCellArray
    # Here we return the raw data for the renderer
    return {
        'points': points,
        'connectivity': [(0, 1), (0, 2), (0, 3)] # Indices of points for lines
    }
```

**Parameters:**
- Line width: $1.2$ [S1, unofficial].
- Line stipple pattern: `0xF0F0` [S1, unofficial].
- Line stipple repeat factor: $1$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** Floating point errors in coordinate calculation may cause gaps in the crosshair at the origin if not handled.
- **Boundaries:** The lines span the full volume extent. If the view is zoomed in, the lines may extend beyond the visible area, which is expected.
- **Degenerate Input:** If spacing is zero, extents are zero, resulting in no visible lines.

## Subtopics

### Crosshair Segment Fitting to Slice Plane
**Definition:** A variant that projects in-plane crosshair arm endpoints along the tile's viewing axis onto the displayed slice plane [S2, unofficial]. This prevents depth-test occlusion (where arms might taper, vanish, or show spikes due to depth buffering) while preserving exact screen coordinates [S2, unofficial].

**When to Choose:** Use this variant when rendering oblique volumes or using WebGL2/WebGPU renderers where depth testing might incorrectly hide parts of the crosshair arms that are technically "behind" the slice surface in depth but should be visible on screen [S2, unofficial].

**Key Differences:** Unlike the basic projection which places lines in world space, this method adjusts the Z-depth of the line endpoints to match the slice plane equation used by the slice shader and picking logic [S2, unofficial].

**Implementation Details:**
1. Retrieve the `slicePlaneEquation` used by the slice shader [S2, unofficial].
2. For each crosshair arm endpoint, project it along the tile's viewing axis onto the slice plane [S2, unofficial].
3. Update geometry buffers with the fitted endpoints [S2, unofficial].
4. Pass the tile's slice type to the `update()` function for both renderers [S2, unofficial].

**Pitfalls:**
- Must explicitly check for 2D slice types to leave untilted (axis-aligned) planes bit-for-bit unchanged. This ensures that axis-aligned tiles continue to share a single WebGL upload, optimizing performance [S2, unofficial].

**Usage:** Implemented in NiiVue WebGL2 and WebGPU crosshair renderers (`packages/niivue/src/gl/crosshair.ts`, `packages/niivue/src/wgpu/crosshair.ts`) [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per frame for generating the crosshair geometry, as it involves a fixed number of arithmetic operations and point insertions regardless of volume size.
- **Memory Complexity:** $O(1)$, storing only a few points and lines.
- **Performance:** The VTK-based implementation [S1, unofficial] is efficient for desktop applications. The WebGL2/WebGPU implementation [S2, unofficial] optimizes further by sharing geometry uploads for axis-aligned tiles.
- **Strengths:** Provides clear spatial orientation; low computational overhead; integrates easily with existing slice rendering pipelines.
- **Limitations:** Basic version may suffer from depth-test artifacts on oblique slices without the fitting variant [S2, unofficial].
- **When to Use:** Interactive medical imaging viewers where users need to understand the 3D position of the 2D slice.
- **When Not to Use:** Non-interactive batch rendering where orientation cues are unnecessary, or in scenes with extremely high polygon counts where even simple overlays might contribute to draw call overhead (though negligible here).

## Background
Orthographic crosshair projection is a standard feature in medical imaging software (e.g., ITK-SNAP, referenced by the color scheme in [S1, unofficial]) to aid in spatial navigation of 3D volume datasets. It allows users to correlate the current 2D slice with the orthogonal planes. The method described in [S1, unofficial] uses VTK for rendering, while [S2, unofficial] describes a modern WebGL2/WebGPU adaptation for web-based viewers (NiiVue), addressing depth-testing issues specific to GPU rendering pipelines.

## Sources
[S1] caitlin-leonard/medviewer, https://github.com/caitlin-leonard/medviewer, fetched 2026-10-27 (unofficial)
[S2] fix(niivue): draw the crosshair arms a 2D slice hides faintly, https://github.com/niivue/mono/pull/344, fetched 2026-10-27 (unofficial)
