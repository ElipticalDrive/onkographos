# Metaball-based Envelope Painting

Also called: 3D Envelope Painting, 3D Painting Metaphor for VOI Definition, Thick Paint Metaphor, Volume Painting

## Overview
Metaball-based Envelope Painting is an interactive visualization technique that defines a Volume of Interest (VOI) by "painting" an implicit envelope around features in unsegmented volumetric data [S1]. It distinguishes itself from sibling methods like [Metaball Field Evaluation](1017-metaball-field-evaluation.md) by integrating the field definition directly into a real-time interaction loop that supports previewing the envelope's effect before commitment, using a "thick paint" metaphor to merge organic shapes [S1]. Unlike [Distance to Hypertextures](1555-distance-to-hypertextures.md) or [Superelliptic Blends](2051-superelliptic-blends.md) which focus on static field composition, this method prioritizes user-guided exploration and editing through two distinct painting modes (3D Surface and 2D Screen) combined with MIDA visualization [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A volumetric dataset $V$, camera parameters, and user interaction events (mouse movement, click).
*   **Output:** A composite image combining the volumetric data visualization and the envelope geometry, along with an updated implicit envelope field representation.

**Data Structures:**
1.  **Metaball Field:** A collection of active metaballs $\{B_i\}$. Each metaball $B_i$ is defined by its center $\mathbf{c}_i$, radius $r_i$, and type (quadric or superquadric). The field value $f(\mathbf{x})$ at a point $\mathbf{x}$ is the sum of contributions from all active metaballs.
2.  **Vertex Buffer:** A GPU memory buffer storing the polygonal mesh extracted from the envelope iso-surface. This buffer is reused across frames to avoid recomputation [S1].
3.  **Position/Normal Maps:** On-the-fly generated textures storing the surface position and normal vectors of the underlying volumetric object, used for 3D Surface Painting mode [S1].
4.  **MIDA Parameters:** Two sets of visualization parameters (one for inside the envelope, one for outside) controlling the Maximum Intensity Difference Accumulation (MIDA) transfer functions [S1].

**Field Definition:**
The envelope is defined by a scalar field $f(\mathbf{x})$. A point is considered inside the envelope if $f(\mathbf{x}) \geq \tau$, where $\tau$ is a threshold (typically 1.0 for standard metaballs). The field is constructed using the six-degree Wyvill polynomial for falloff [S1]. For a single metaball $i$ with center $\mathbf{c}_i$ and radius $r_i$, the contribution $f_i(\mathbf{x})$ is:

$$
f_i(\mathbf{x}) = \begin{cases} 
\frac{1}{r_i^6} (2r_i^2 - d^2)^3 & \text{if } d < r_i \\
0 & \text{otherwise}
\end{cases}
$$

where $d = \|\mathbf{x} - \mathbf{c}_i\|$ is the Euclidean distance from the query point to the metaball center [S1]. The total field is $f(\mathbf{x}) = \sum_i f_i(\mathbf{x})$.

For **superquadric** metaballs (used in 3D Surface Painting for cylinder/disk shapes), the field function is modified to define elongated or flattened shapes oriented along a normal vector $\mathbf{n}_i$. The exact superquadric equation is not provided in the sources, but it replaces the quadric term to create non-spherical blobs [S1].

**Algorithm Steps:**

1.  **Initialization:** Load volumetric data $V$. Initialize an empty list of metaballs. Initialize MIDA parameters for "inside" and "outside" visualization.
2.  **Interaction Loop:**
    a.  **Brush Positioning:** Determine the brush tip position $\mathbf{c}_{new}$ based on user input.
        *   In **3D Surface Painting Mode**: Project the mouse cursor to the scene. Override the depth of $\mathbf{c}_{new}$ using the depth value from the object's surface obtained from the position map. Orient the brush using the surface normal from the normal map [S1].
        *   In **2D Screen Painting Mode**: Set the depth of $\mathbf{c}_{new}$ to a constant value (discarding depth info). The brush is aligned with the view direction [S1].
    b.  **Preview Rendering:**
        *   Temporarily add a metaball $B_{preview}$ at $\mathbf{c}_{new}$ with the selected brush type (quadric or superquadric) to the field.
        *   Compute the envelope iso-surface $f(\mathbf{x}) = \tau$ using a fast GLSL shader implementation of the Marching Cubes algorithm [S1].
        *   Extract the geometry into the memory vertex buffer. If the buffer already contains geometry, update it in place to avoid recomputation [S1].
        *   Render the volume using MIDA. Apply the "inside" MIDA style to voxels where $f(\mathbf{x}) \geq \tau$ and the "outside" style otherwise.
        *   Render the envelope polygon mesh using Front-to-Back Depth Peeling. This technique renders the polygonal envelope geometry layer by layer from the camera position, filling gaps between layers with the volumetric DVR results, blending them seamlessly [S1].
        *   Display the preview to the user.
    c.  **Deposit (Commit):** If the user clicks:
        *   Permanently add $B_{preview}$ to the list of active metaballs.
        *   The envelope is now extended by this blob [S1].
3.  **Volume Editing (Optional):** To carve or peel volume data, set the mapped data intensity range to zero for voxels inside or outside the current envelope $f(\mathbf{x}) \geq \tau$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Global State
metaballs = []  # List of Metaball objects
vertex_buffer = None  # GPU buffer for Marching Cubes output
mida_params_inside = MIDAParameters()
mida_params_outside = MIDAParameters()
is_3d_surface_mode = False
is_2d_screen_mode = False

class Metaball:
    def __init__(self, center, radius, brush_type='quadric', normal=None):
        self.center = center
        self.radius = radius
        self.brush_type = brush_type
        self.normal = normal  # Only for superquadric

def wyvill_field(x, metaball):
    """
    Computes the contribution of a single metaball to the scalar field.
    Uses the six-degree Wyvill polynomial.
    """
    d_squared = dot(x - metaball.center, x - metaball.center)
    r_squared = metaball.radius * metaball.radius
    
    if d_squared < r_squared:
        # Six-degree Wyvill polynomial for quadric
        # f(x) = (1/r^6) * (2*r^2 - d^2)^3
        term = (2 * r_squared - d_squared) ** 3
        return term / (r_squared ** 3)
    else:
        return 0.0

def superquadric_field(x, metaball):
    """
    Computes contribution for superquadric metaball (cylinder/disk).
    Exact formula not provided in sources; assumes orientation along metaball.normal.
    [G] Placeholder for specific superquadric equation implementation.
    """
    # Implementation depends on specific superquadric definition used in [S1]
    pass

def evaluate_field(x):
    total = 0.0
    for mb in metaballs:
        if mb.brush_type == 'quadric':
            total += wyvill_field(x, mb)
        elif mb.brush_type == 'superquadric':
            total += superquadlic_field(x, mb)
    return total

def render_preview(brush_tip, brush_type, brush_normal=None):
    """
    Renders the real-time preview of the painting operation.
    """
    # 1. Determine brush parameters
    radius = get_user_brush_radius()
    preview_mb = Metaball(brush_tip, radius, brush_type, brush_normal)
    
    # 2. Temporarily add to field for evaluation
    metaballs.append(preview_mb)
    
    # 3. Extract Iso-surface using GLSL Marching Cubes
    # The shader evaluates f(x) for voxels in the bounding box of the brush
    # and generates triangles where f(x) >= 1.0
    geometry = glsl_marching_cubes(evaluate_field, threshold=1.0)
    
    # 4. Update Vertex Buffer
    if vertex_buffer is None:
        vertex_buffer = create_gpu_buffer(geometry)
    else:
        update_gpu_buffer(vertex_buffer, geometry)
        
    # 5. Render Volume with MIDA
    # For each sample point along rays:
    #   val = evaluate_field(sample_point)
    #   if val >= 1.0: apply mida_params_inside
    #   else: apply mida_params_outside
    
    # 6. Render Envelope Geometry with Depth Peeling
    # Render vertex_buffer using Front-to-Back Depth Peeling
    # Blend with volume results
    render_depth_peeling(vertex_buffer)
    
    # 7. Remove preview blob
    metaballs.pop()

def render_final():
    """
    Renders the final scene with committed metaballs.
    """
    # Same as render_preview but without temporary addition/removal
    # and potentially using cached geometry if metaballs haven't changed
    pass

def handle_input(mouse_pos, is_click):
    if is_3d_surface_mode:
        # Get depth and normal from position/normal maps
        depth = get_depth_from_map(mouse_pos)
        normal = get_normal_from_map(mouse_pos)
        brush_tip = unproject(mouse_pos, depth)
        brush_type = 'superquadric' # Alternative brush type for surface mode
    elif is_2d_screen_mode:
        # Constant depth, no normal orientation
        depth = constant_depth
        brush_tip = unproject(mouse_pos, depth)
        brush_type = 'quadric' # Conventional spherical blob
    else:
        # Free-hand 3D mode
        brush_tip = get_3d_cursor_position()
        brush_type = 'quadric'
        
    if is_click:
        # Deposit the blob
        metaballs.append(Metaball(brush_tip, get_user_brush_radius(), brush_type, 
                                 brush_normal if is_3d_surface_mode else None))
    else:
        # Preview
        render_preview(brush_tip, brush_type, 
                       brush_normal if is_3d_surface_mode else None)
```

**Parameters:**
*   **Brush Type:** 'quadric' (spherical blob) or 'superquadric' (cylinder/disk). Default is quadric [S1].
*   **Brush Radius:** User-controlled size of the metaball.
*   **Threshold ($\tau$):** Typically 1.0 for the Wyvill polynomial iso-surface [S1].
*   **MIDA Parameters:** Two sets of transfer functions (inside/outside). Specific values are user-defined [S1].

**Edge Cases and Pitfalls:**
*   **Geometry Recomputation:** Re-evaluating Marching Cubes every frame is expensive. The method avoids this by reusing the memory vertex buffer, updating it only when the metaball configuration changes significantly or for the preview blob which is transient [S1].
*   **Precision:** The Wyvill polynomial involves high-degree terms ($d^6$). Care must be taken with floating-point precision for very small or large radii [S1].
*   **2D Mode Depth:** In 2D Screen Painting Mode, depth information is discarded. The envelope acts as a stencil through the entire volume along the ray. This can lead to ambiguity if the user expects a local 3D selection [S1].

## Subtopics

### 3D Surface Painting Mode
A mode where the brush tip automatically follows the surface of the volumetric object. It uses superquadric metaballs (cylinders) oriented by surface normals to create thin-shell envelopes. Choose this when you need to paint along complex curved surfaces [S1].

### 2D Screen Painting Mode
A mode where the brush depth is constant, creating a stencil-like effect through the entire volume. Choose this for quick, global selections or when depth precision is not required [S1].

### MIDA Visualization
The rendering technique used to visualize the volume inside and outside the envelope. It allows for distinct visual styles for the VOI and the context. Choose this when context-preserving exploration is needed [S1].

## Operating characteristics
*   **Time Complexity:** The field evaluation is $O(N \cdot M)$ where $N$ is the number of voxels sampled and $M$ is the number of active metaballs. Marching Cubes is $O(V)$ where $V$ is the number of voxels in the bounding box of the metaballs. Depth Peeling is $O(L \cdot R)$ where $L$ is the number of layers and $R$ is the resolution.
*   **Memory Complexity:** $O(V)$ for the volume data and $O(P)$ for the polygonal mesh, where $P$ is the number of polygons in the envelope. The vertex buffer reuse minimizes memory allocation overhead [S1].
*   **Performance:** The method is designed for real-time interaction. The use of GLSL shaders for Marching Cubes and Depth Peeling allows for GPU acceleration. The preview mechanism adds a slight overhead but is optimized by reusing geometry buffers [S1].
*   **Strengths:** Intuitive interaction, real-time preview, flexible envelope shapes, context-preserving visualization.
*   **Limitations:** Dependent on the quality of the MIDA visualization; may not be suitable for highly detailed internal structures if the envelope is too coarse. The superquadric field implementation is less standardized than the quadric Wyvill polynomial [S1].

## Background
Metaball-based Envelope Painting was introduced by L. Faynshteyn and T. McInerney in 2012 in the paper "Context-Preserving Volumetric Data Set Exploration Using a 3D Painting Metaphor" [S1]. It builds on the concept of Metaballs [2] (referenced in [S1]) and the MIDA visualization algorithm [19] (referenced in [S1]). The method was developed to address the need for intuitive, interactive exploration of unsegmented volumetric data, combining visualization and interaction in a single model. It was validated through a user study with 11 subjects [S1].

## Sources
[S1] Context-Preserving Volumetric Data Set Exploration Using a 3D Painting Metaphor, https://doi.org/10.1007/978-3-642-33179-4_33, fetched 2026-10-27
