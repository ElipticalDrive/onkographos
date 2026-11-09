# 3D Cursor

Also called: Cross-hair Cursor, Cross-plane Cursor, virtual cursor

## Overview
The 3D Cursor is a direct interaction method that specifies X, Y, and Z coordinates within a volume dataset to enable spatial selection and exploration [S1, S2, unofficial]. It distinguishes itself from clipping or deformation techniques by acting as a localized probe or marker rather than a global modifier, allowing users to identify specific spatial relationships or select discrete features [S2, unofficial]. While siblings like Cropping modify the visible domain, the 3D Cursor provides a reference point for data querying or targeted selection, often implemented as a hardware-generated, software-controlled overlay [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** User position/orientation data (e.g., from a tracker, mouse, or haptic device) defining the cursor's location in world or volume space [S1, unofficial][S3].
*   **Output:** A visual representation of the cursor blended into the rendered volume, and optionally, data values (scalar, gradient) at the cursor's tip [S3].

**Data Structures:**
The cursor is defined by its geometric parameters. For a standard cross-hair/cross-plane cursor (as in VolumePro), the structure includes:
*   `origin`: $(x, y, z)$ coordinates in volume space.
*   `dimensions`: $(w_x, w_y, w_z)$ defining the length of the cursor arms along each axis.
*   `attributes`:
    *   `enabled`: Boolean flags for x, y, and z axes independently [S2, unofficial].
    *   `color`: RGB or RGBA value for each enabled axis [S2, unofficial].
    *   `thickness`: Pixel or voxel width for each axis [S2, unofficial].

**Steps:**
1.  **Acquire Position:** Determine the 3D position of the cursor based on user input. In local interaction, this maps directly to the user's hand position [S1, unofficial]. In action-at-a-distance, it may follow a laser beam or virtual drone path [S1, unofficial].
2.  **Define Geometry:** Construct the cursor geometry. For a cross-hair, this involves defining line segments or thin boxes centered at `origin` with lengths `w_x`, `w_y`, `w_z` along the respective axes [S2, unofficial].
3.  **Render Cursor:** Render the cursor geometry into the frame buffer. The cursor is typically rendered after the volume data but before post-processing, or via alpha blending [S2, unofficial].
4.  **Blend:** Apply alpha blending to integrate the cursor into the volume visualization. The blending equation is standard:
    $$ C_{final} = C_{cursor} \cdot \alpha_{cursor} + C_{volume} \cdot (1 - \alpha_{cursor}) $$
    where $C$ is color and $\alpha$ is opacity [S2, unofficial].
5.  **Query Data (Optional):** If the cursor is a probe (e.g., Data Probe Widget), query the volume data $V(x,y,z)$ and its derivatives at the tip position [S3].
6.  **Feedback:** Display queried values in a transfer function widget or UI [S3].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_3d_cursor(cursor_pos, cursor_dims, cursor_attrs, volume_data, camera):
    """
    Renders a 3D cursor into the volume.
    cursor_pos: (x, y, z) float tuple
    cursor_dims: (wx, wy, wz) float tuple
    cursor_attrs: dict with keys 'enabled' (bool list), 'color' (list of tuples), 'thickness' (list of ints)
    volume_data: 3D array or texture
    camera: Camera object with view/projection matrices
    """
    # 1. Define cursor geometry based on enabled axes
    cursor_primitives = []
    for i, axis in enumerate(['x', 'y', 'z']):
        if cursor_attrs['enabled'][i]:
            # Create a thin box or line segment along the axis
            # Centered at cursor_pos
            # Length = cursor_dims[i]
            # Thickness = cursor_attrs['thickness'][i]
            # Color = cursor_attrs['color'][i]
            primitive = create_axis_primitive(axis, cursor_pos, cursor_dims[i], 
                                            cursor_attrs['thickness'][i], 
                                            cursor_attrs['color'][i])
            cursor_primitives.append(primitive)

    # 2. Transform primitives to clip space
    clip_primitives = []
    for prim in cursor_primitives:
        # Transform vertices using camera view and projection matrices
        transformed = transform_to_clip_space(prim, camera)
        clip_primitives.append(transformed)

    # 3. Rasterize and Blend
    # Render primitives with alpha blending enabled
    # Depth test should be enabled to occlude volume behind cursor if desired,
    # or disabled if cursor should always be visible on top.
    for prim in clip_primitives:
        rasterize_and_blend(prim, alpha=1.0) # Alpha handled by primitive color

    # 4. Query Data if Probe Mode
    if cursor_attrs.get('probe_mode', False):
        # Interpolate volume data at cursor_pos
        val = trilinear_interpolate(volume_data, cursor_pos)
        grad = compute_gradient(volume_data, cursor_pos)
        return val, grad
    return None
```

**Parameters:**
*   `width`/`length`: User-definable dimensions for each axis [S2, unofficial].
*   `color`: Independent color per axis [S2, unofficial].
*   `thickness`: Independent thickness per axis [S2, unofficial].

**Edge Cases:**
*   **Precision:** Cursor position must be accurately mapped from input device to volume coordinates, especially in action-at-a-distance scenarios [S1, unofficial].
*   **Occlusion:** The semi-transparent lining of a Silk Cursor helps resolve depth ambiguity (front/inside/behind) [S1, unofficial].

## Subtopics

### Data Probe Widget
A 3D widget shaped like a pencil used to point at features in the volume, linking spatial interaction to transfer function feedback [S3]. It reports its tip's position and a slider value [S3]. The tip tracks the volume space point, querying data values and derivatives which are displayed as a ball in the Transfer Function Widget, with voxel corners represented as connected balls [S3]. It can be linked to haptic devices like the SensAble PHANTOM for direct 3D input [S3].

### Go-Go Technique
A selection technique enhancing 3D cursor range for distant objects using a non-linear arm-growing metaphor [S1, unofficial]. For nearby objects, virtual hand movement maps directly to real hand movement. When the user extends their hand beyond 2/3 of their arm length, the mapping becomes non-linear, and the virtual arm grows to reach remote objects, overcoming the arm's-length limitation of standard cursors [S1, unofficial].

### Silk Cursor
A 3D volume cursor extending 2D area cursors, featuring a rectangular shape lined with a semi-transparent surface [S1, unofficial]. This lining creates a partial occlusion effect, allowing users to determine if a target is in front, inside, or behind the cursor [S1, unofficial]. It increases the effective activation area of targets and was found beneficial in both monoscopic and stereoscopic displays [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Rendering is $O(N_c)$ where $N_c$ is the number of cursor primitives (typically constant, e.g., 3 for cross-hair). Data querying is $O(1)$ with trilinear interpolation.
*   **Memory Complexity:** $O(1)$ for cursor state.
*   **Performance:** Hardware-accelerated cursors (like in VolumePro) are efficient, leveraging dedicated ASIC pipelines [S2, unofficial].
*   **Strengths:** Direct, intuitive selection; supports both local and distant interaction; provides immediate visual feedback on spatial relationships [S1, S2, unofficial].
*   **Limitations:** Range limited by input device or technique (e.g., Go-Go mitigates this); depth ambiguity can occur without features like the Silk Cursor's lining [S1, unofficial].
*   **When to use:** For precise selection of specific points or features within a volume, or for exploring spatial relationships. Not suitable for global volume modification.

## Background
The 3D Cursor concept was suggested by Hinckley et al. [1994], Mine [1995a], and Poupyrev et al. [1996] as a direct interaction method [S1, unofficial]. Mine distinguished between local and action-at-a-distance interactions [S1, unofficial]. The Silk Cursor was introduced by Zhai et al. [1994] as an extension of 2D area cursors [S1, unofficial]. The Go-Go technique was introduced by Poupyrev [1996] to enhance selection range [S1, unofficial]. The VolumePro system implemented a hardware-generated, software-controlled 3D cursor (cross-hair/cross-plane) [S2, unofficial]. The Data Probe Widget was part of a system using multi-dimensional transfer functions [S3].

## Sources
[S1] Interaction with volumetric displays, http://hdl.handle.net/1807/16733, fetched 2026-10-26 (unofficial)
[S2] The VolumePro Real-Time Ray-Casting System, https://www.merl.com/publications/docs/TR99-19.pdf, fetched 2026-10-26 (unofficial)
[S3] Interactive Volume Rendering Using Multi-Dimensional Transfer Functions, https://publications.sci.utah.edu/publications/jmk01/simian-paper.pdf, fetched 2026-10-27
