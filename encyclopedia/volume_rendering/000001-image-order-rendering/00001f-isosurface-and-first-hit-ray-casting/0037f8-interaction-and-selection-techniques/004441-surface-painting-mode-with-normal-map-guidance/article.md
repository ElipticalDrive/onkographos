# Surface Painting Mode with Normal Map Guidance

Also called: Auto-follow surface, MIDA-based Surface Painting, Surface Painting Mode

## Overview
Surface Painting Mode is an interaction technique that allows users to paint volumetric envelopes by automatically overriding the brush tip's depth and orientation to match the underlying isosurface of a volumetric dataset [S1, unofficial]. It solves the problem of manually aligning brush geometry with complex, curved internal structures by using a pre-computed normal map derived from ray casting [S1, unofficial]. It differs from standard painting by decoupling the brush position from the surface geometry, and from clipping techniques by modifying the painted object's attributes rather than the view volume [S1, unofficial].

## How it works
The method relies on the principles of [Isosurface and first-hit ray casting](20-isosurface-and-first-hit-ray-casting.md) to generate surface metadata, which is then used to guide the brush transformation [G].

### Inputs and Outputs
*   **Inputs:**
    *   Volumetric dataset $V$ with scalar values.
    *   Mouse cursor screen coordinates $(u, v)$.
    *   Brush parameters: type (spherical, cylindrical, superquadratic), size, height, and sign (positive/negative).
    *   Isosurface threshold $\tau$.
*   **Outputs:**
    *   Updated volumetric envelope (series of metaballs) aligned to the surface.
    *   Two 2D buffers: Position Buffer $B_P$ and Normal Buffer $B_N$.

### Data Structures
*   **Position Buffer $B_P$:** A 2D array of size $W \times H$ (screen resolution). Each element stores the 3D world position $\mathbf{p} = (x, y, z)$ of the first-hit isosurface point corresponding to the screen pixel.
*   **Normal Buffer $B_N$:** A 2D array of size $W \times H$. Each element stores the 3D surface normal $\mathbf{n} = (n_x, n_y, n_z)$ at the corresponding hit point.
*   **Brush Tip:** Defined as a metaball. For cylindrical brushes, the orientation is determined by the normal vector. For superquadratic metaballs, the orientation matrix is derived from the normal.

### Algorithm Steps

1.  **Isosurface Generation (Buffer Population):**
    For each pixel $(u, v)$ in the screen resolution $W \times H$:
    a.  Construct a ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ from the camera through $(u, v)$ into the volume space.
    b.  Perform MIDA ray casting: Evaluate scalar values $V(\mathbf{r}(t))$ along the ray.
    c.  Stop evaluation immediately upon encountering the first non-transparent voxel (where $V(\mathbf{r}(t)) \geq \tau$).
    d.  **Refinement:** Apply a 5-step isosurface refinement routine to the hit position $t_{hit}$:
        i.  Subdivide the current ray stepping size into increasingly smaller values.
        ii. Probe the volume space in both directions from the current position along the ray to find a finer matching isosurface intersection $t_{final}$.
    e.  Calculate the position $\mathbf{p} = \mathbf{r}(t_{final})$.
    f.  Calculate the normal $\mathbf{n}$ at $\mathbf{p}$ using finite differences of the scalar field $V$.
    g.  Store $\mathbf{p}$ in $B_P[u][v]$ and $\mathbf{n}$ in $B_N[u][v]$.

2.  **Normal Map Filtering:**
    Apply a convolution operation with a Gaussian blurring kernel to $B_N$ to reduce noise caused by the discrete nature of the volumetric dataset [S1, unofficial]. This prevents abrupt orientation changes in superquadratic metaballs.

3.  **Brush Interaction:**
    When the user moves the mouse to $(u, v)$ and initiates a paint stroke:
    a.  Retrieve $\mathbf{p}_{surf} = B_P[u][v]$ and $\mathbf{n}_{surf} = B_N[u][v]$.
    b.  **Depth Override:** Set the brush tip's depth (z-coordinate in local space or distance along ray) to match $\mathbf{p}_{surf}$. This ensures the brush follows the object's surface.
    c.  **Orientation Override:**
        *   If the brush is **cylindrical**: Orient the cylinder's axis perpendicular to $\mathbf{n}_{surf}$.
        *   If the brush is **superquadratic**: Calculate a rotation matrix $R$ using $\mathbf{n}_{surf}$ as the up vector or primary orientation axis, and apply it to the metaball.
    d.  **Penetration Depth:** Allow the user to adjust the depth of the object’s surface penetration by the brush tip, effectively controlling how deeply the brush intersects the surface [S1, unofficial].
    e.  Add the transformed metaball to the envelope series. The brush tip is always the first metaball, defining the attributes (size, field function) for subsequent merged metaballs [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_surface_buffers(volume, camera, width, height, threshold):
    """
    Generates Position Buffer B_P and Normal Buffer B_N.
    """
    B_P = empty_array(width, height, dtype=vec3)
    B_N = empty_array(width, height, dtype=vec3)
    
    for v in range(height):
        for u in range(width):
            # 1. Ray construction
            ray_origin, ray_direction = camera.screen_to_ray(u, v)
            
            # 2. Initial MIDA hit
            t_hit = find_first_hit_mida(volume, ray_origin, ray_direction, threshold)
            
            if t_hit is None:
                continue # No surface hit for this pixel
                
            # 3. Refinement
            t_final = refine_isosurface_hit(volume, ray_origin, ray_direction, t_hit, steps=5)
            
            # 4. Position and Normal calculation
            p = ray_origin + t_final * ray_direction
            n = calculate_normal(volume, p)
            
            # 5. Store in buffers
            B_P[u, v] = p
            B_N[u, v] = n
            
    # 6. Filtering
    B_N = gaussian_blur(B_N, kernel_size=3)
    
    return B_P, B_N

def paint_on_surface(mouse_u, mouse_v, brush, B_P, B_N):
    """
    Updates brush transformation based on surface buffers.
    """
    # Retrieve surface data
    p_surf = B_P[mouse_u, mouse_v]
    n_surf = B_N[mouse_u, mouse_v]
    
    # Override depth
    brush.depth = p_surf
    
    # Override orientation
    if brush.type == "cylindrical":
        # Orient cylinder perpendicular to normal
        brush.orientation = align_perpendicular(brush.axis, n_surf)
    elif brush.type == "superquadratic":
        # Calculate rotation matrix from normal
        R = rotation_matrix_from_normal(n_surf)
        brush.orientation = R
        
    # Apply penetration depth adjustment
    brush.position = p_surf + (n_surf * brush.penetration_depth)
    
    return brush
```

### Parameters
*   **Refinement Steps:** 5 steps are used in the refinement routine [S1, unofficial].
*   **Gaussian Kernel:** Applied to the normal map to reduce noise [S1, unofficial].
*   **Brush Shape:** Cylindrical or Spherical (Superquadratic metaballs are used for the envelope definition, but the brush tip shape influences the merge) [S1, unofficial].
*   **Penetration Depth:** Adjustable parameter controlling how deeply the brush intersects the surface [S1, unofficial].

### Edge Cases and Pitfalls
*   **Normal Noise:** Volumetric datasets produce noisy normal maps, leading to abrupt orientation changes in superquadratic metaballs [S1, unofficial]. Mitigated by 5-step refinement and Gaussian blurring [S1, unofficial].
*   **Transparent Voxels:** The algorithm stops at the first *non-transparent* voxel. If the volume is fully transparent, no surface is generated [S1, unofficial].
*   **Degenerate Normals:** At flat surfaces or singularities, normals may be unstable. The Gaussian blur helps smooth these transitions [S1, unofficial].

## Subtopics

### Interactive Concave Volume Clipping via GPU Boolean Operations
Uses boolean operations to clip volumes, whereas Surface Painting modifies the object's envelope via metaballs. Choose clipping for visibility control; choose painting for annotation/envelope creation [3592-interactive-concave-volume-clipping-via-gpu-boolean-operatio.md].

### Surface Peeling
Removes layers of the volume to reveal internal structures, whereas Surface Painting adds external envelopes. Choose peeling for internal exploration; choose painting for surface-aligned annotation [7679-surface-peeling.md].

### Direct Haptic Rendering of Isosurface by Intermediate Representation
Provides haptic feedback during interaction, whereas Surface Painting is primarily visual/interactive. Choose haptic rendering for tactile feedback; choose painting for visual envelope definition [8620-direct-haptic-rendering-of-isosurface-by-intermediate-repres.md].

### Selecting Occluders
Focuses on selecting objects that occlude others, whereas Surface Painting focuses on aligning brush geometry with the surface. Choose occluder selection for visibility management; choose painting for surface-aligned modification [14974-selecting-occluders.md].

## Operating characteristics
*   **Time Complexity:** $O(W \cdot H \cdot S)$ for buffer generation, where $W, H$ are screen dimensions and $S$ is the number of samples per ray (including refinement steps). Painting interaction is $O(1)$ per brush stroke after buffer generation.
*   **Memory Complexity:** $O(W \cdot H)$ for storing the two 2D buffers ($B_P$ and $B_N$).
*   **Performance:** The method requires a full screen-resolution ray cast to generate the buffers, which can be computationally expensive for high resolutions. However, the buffers can be reused for multiple painting operations without re-casting [S1, unofficial].
*   **Strengths:** Allows precise alignment of brush geometry with complex curved surfaces; supports negative brushes for erasing; flexible envelope definition via metaball merging [S1, unofficial].
*   **Limitations:** Noisy normals can cause visual artifacts without filtering; requires pre-computation of surface buffers; limited to isosurface-aligned painting [S1, unofficial].
*   **When to Use:** When the user needs to paint envelopes that closely follow the curvature of a volumetric object (e.g., painting an envelope around a finger's digit) [S1, unofficial].
*   **When Not to Use:** When painting on flat or simple surfaces where manual alignment is sufficient; when real-time interaction without pre-computation is required [S1, unofficial].

## Background
The method was introduced in the context of context-preserving volume image exploration using a 3D painting metaphor [S1, unofficial]. It builds upon the MIDA ray casting algorithm and the use of metaballs for envelope definition [S1, unofficial]. The technique addresses the need for intuitive interaction with volumetric data by providing a painting metaphor that adapts to the underlying geometry [S1, unofficial].

## Sources
[S1] Context-Preserving Volume Image Exploration Using A 3D Painting Metaphor, https://doi.org/10.32920/ryerson.14646888, fetched 2026-10-26 (unofficial)
