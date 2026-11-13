# Voxel Face Visibility Check

Also called: Back-Face Culling, Camera-relative Block Culling, renderBlock

## Overview
Voxel Face Visibility Check determines which of the six faces of a voxel block are visible to the viewer by evaluating the geometric relationship between face normals and a reference direction. It solves the problem of rendering unnecessary geometry that faces away from the camera, thereby reducing the number of triangles submitted to the graphics pipeline. Unlike its parent [Block Visibility Culling](5334-block-visibility-culling.md), which operates at the coarse block level based on camera movement, this method operates at the fine-grained face level within a block.

## How it works
**Inputs:**
- `cube_center`: The 3D position of the voxel block's center ($\mathbf{c}_{block}$).
- `cube_size`: The side length of the cubic voxel block ($s$).
- `view_direction`: The direction vector from the block towards the viewer (or camera position, depending on implementation).

**Outputs:**
- A boolean list or bitmask indicating which of the six faces are visible.

**Data Structures:**
The method evaluates six faces. Each face is defined by an outward normal $\mathbf{n}_i$ and a center point $\mathbf{c}_i$.
- Normals:
  - Front: $(0, 0, 1)$
  - Back: $(0, 0, -1)$
  - Right: $(-1, 0, 0)$
  - Left: $(1, 0, 0)$
  - Top: $(0, 1, 0)$
  - Bottom: $(0, -1, 0)$

**Steps:**
1. Calculate the half-size of the cube: $h = s / 2$.
2. Determine the center of each face $\mathbf{c}_i$ by adding the offset vector $\pm h \cdot \mathbf{n}_i$ to `cube_center`.
   $$ \mathbf{c}_i = \mathbf{c}_{block} + h \cdot \mathbf{n}_i $$
3. For each face $i$, compute the dot product between the face center $\mathbf{c}_i$ and the outward normal $\mathbf{n}_i$. Note: Some implementations use the vector from the face center to the camera instead of the normal, but the cited source [S1, unofficial] uses the face center vector relative to the origin.
   $$ d_i = \mathbf{c}_i \cdot \mathbf{n}_i $$
   *Correction/Clarification based on [S1, unofficial] and [S2, unofficial]:* Source [S2, unofficial] states the standard back-face culling formula is a dot product between the **face normal** and the **viewing direction**. Source [S1, unofficial] implements a specific variant using the face center and normal relative to the world origin. The standard geometric interpretation for "visible if facing camera" requires the dot product of the normal and the vector to the camera. If the dot product is positive, the angle between the normal and the view vector is less than 90 degrees, meaning the face points towards the viewer.
   
   Let $\mathbf{v}$ be the vector from the face center to the camera (or the viewing direction).
   $$ v_i = \mathbf{n}_i \cdot \mathbf{v} > 0 $$
   
   However, adhering strictly to the provided notes from [S1, unofficial], the implementation computes:
   $$ v_i = \mathbf{c}_i \cdot \mathbf{n}_i > 0 $$
   This specific formula in [S1, unofficial] is noted as a pitfall if the camera is not at the world origin.

4. Mark face $i$ as visible if the computed value is positive.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_face_visibility_check(cube_center, cube_size, camera_position):
    """
    Determines visible faces of a voxel block.
    Note: This implementation follows the logic in [S1, unofficial] but corrects the 
    pitfall identified in the notes by using the camera position for 
    accurate visibility, or uses the standard back-face culling logic from [S2, unofficial].
    """
    half_size = cube_size / 2.0
    
    # Define normals for the 6 faces
    # Order: Front, Back, Right, Left, Top, Bottom
    normals = [
        (0, 0, 1),   # Front
        (0, 0, -1),  # Back
        (-1, 0, 0),  # Right
        (1, 0, 0),   # Left
        (0, 1, 0),   # Top
        (0, -1, 0)   # Bottom
    ]
    
    visible_faces = []
    
    for i, normal in enumerate(normals):
        # Calculate face center
        # Offset is half_size in the direction of the normal
        face_center = (
            cube_center[0] + normal[0] * half_size,
            cube_center[1] + normal[1] * half_size,
            cube_center[2] + normal[2] * half_size
        )
        
        # Vector from face center to camera
        # [G] Standard back-face culling uses the vector to the viewer.
        # Source [S1, unofficial] uses face_center dot normal, which is flawed for arbitrary cameras.
        # We implement the robust version described in [S2, unofficial] and implied by correct graphics principles.
        view_vector = (
            camera_position[0] - face_center[0],
            camera_position[1] - face_center[1],
            camera_position[2] - face_center[2]
        )
        
        # Dot product of normal and view vector
        dot_product = (normal[0] * view_vector[0] + 
                       normal[1] * view_vector[1] + 
                       normal[2] * view_vector[2])
        
        # If dot product > 0, the face is pointing towards the camera
        if dot_product > 0:
            visible_faces.append(i)
            
    return visible_faces
```

**Parameters:**
- `GL_CULL_FACE`: An OpenGL flag that enables hardware-accelerated back-face culling [S2, unofficial].

**Pitfalls:**
- **Reference Point Error:** Source [S1, unofficial] implements the visibility test using the world origin $(0,0,0)$ as the reference point for the dot product (i.e., using the face center vector itself rather than the vector to the camera). This marks faces as visible only if they point away from the world origin, which is incorrect for arbitrary camera positions [S1, unofficial].
- **Precision:** Floating-point precision issues may cause faces exactly parallel to the view direction (dot product $\approx 0$) to be incorrectly culled or rendered.

## Operating characteristics
- **Time Complexity:** $O(1)$ per block, as it involves a fixed number of arithmetic operations (6 dot products).
- **Memory Complexity:** $O(1)$, requiring only temporary storage for face centers and normals.
- **Strengths:** Extremely fast; reduces the number of vertices and indices sent to the GPU.
- **Limitations:** Only culls faces facing away from the camera. It does not cull faces that are occluded by other geometry (requires depth testing or further culling). The naive implementation in [S1, unofficial] is incorrect for non-origin-centered cameras.

## Background
Back-face culling is a fundamental technique in computer graphics. It is implemented in hardware via the `GL_CULL_FACE` flag in OpenGL [S2, unofficial]. The concept is widely used in out-of-core visualization systems to manage large datasets [S2, unofficial].

## Sources
[S1] anb2473/Python-Voxel-Rendering-Engine, https://github.com/anb2473/Python-Voxel-Rendering-Engine, fetched 2026-10-27 (unofficial)
[S2] New Techniques for Out-Of-Core Visualization of Large Datasets, https://ctsilva.github.io/thesis/wagner-correa-thesis.pdf, fetched 2026-10-30 (unofficial)
