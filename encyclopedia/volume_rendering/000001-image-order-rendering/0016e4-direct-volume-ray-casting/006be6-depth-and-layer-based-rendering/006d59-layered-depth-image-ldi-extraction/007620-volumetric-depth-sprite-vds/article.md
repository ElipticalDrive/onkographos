# Volumetric Depth Sprite (VDS)

Also called: Layered Depth Image (LDI) representation

## Overview
Volumetric Depth Sprite (VDS) is an image-based representation of a solid shape's volume, designed to enable fast, real-time point-in-volume tests and ray-marching operations on arbitrary 3D solids [S1]. It extends the Layered Depth Image (LDI) concept by storing depth values along viewing rays aligned to the negative z-axis within a 3D texture, effectively capturing layers of unique depth complexity [S1]. Unlike standard LDI extraction which focuses on view-dependent rendering, VDS is optimized for volumetric parity testing, allowing a point to be classified as inside or outside a volume by counting depth layer crossings [S1].

## How it works
### Inputs and Outputs
**Input:** A static 3D mesh $S$ defined in world coordinates.
**Output:** A Volumetric Depth Sprite structure $VDS_S$ containing the sprite's position, transformation matrix, layered depth image data, and depth complexity.

### Data Structures
The VDS structure is defined as:
$$ VDS_S = (P_S, M_S, LDI_S, d_S) $$

*   $P_S \in \mathbb{R}^3$: The position of the sprite in world coordinates.
*   $M_S$: A $4 \times 4$ matrix representing linear transformations (rotation and scaling) of the VDS.
*   $LDI_S$: The layered depth image data, stored as a 3D texture.
*   $d_S \in \mathbb{N} \setminus \{0, 1\}$: The depth complexity of shape $S$, representing the maximum number of intersections any ray parallel to the z-axis makes with the mesh.

**3D Texture Layout:**
*   **Dimensions:** Width $w$, Height $h$, Depth $d_S$.
*   **Format:** Luminance texture format with a single 32-bit floating-point channel.
*   **Interpolation:** Nearest neighbor.
*   **Border:** Set to 1 for each color component.
*   **Wrap Mode:** Clamp to border.

### Pre-processing Steps
The VDS is generated via a pre-processing step using multipass Render-to-Texture (RTT) [S1].

1.  **Transformation:** Transform the shape $S$ into the unit volume $[0, 1]^3$. Set up an orthogonal projection covering this unit volume and adjust the near and far clipping planes accordingly.
2.  **Texture Setup:** Determine the depth complexity $d_S$ of the shape. Create a 3D texture with dimensions $w \times h \times d_S$. Configure nearest neighbor interpolation, set the texture border to 1, and set the wrap mode to clamp to border. Use the luminance format with a single 32-bit floating-point channel to bound texture size [S1].
3.  **Depth Peeling:** Perform depth peeling [Cas01] on the solid $S$ using RTT. For each slice $i$ of the 3D texture (where $0 \le i < d_S$), render the $i$-th depth layer. Linearize the depth values using an inverse W-buffer [LJ99] before storing them in the texture.

### Sampling and Ray Marching
To obtain a depth value $d_i \in [0, 1]$ for the $i$-th depth layer at a 2D point $(s, t)$, the 3D texture in $LDI_S$ is sampled at the coordinate $(s, t, \frac{i}{d_S})$ [S1].

The Volumetric Parity Test (VPT) classifies a point $P \in \mathbb{R}^3$ relative to the volume. In the ray-marching implementation, the parity of the coordinate swaps if the ray passes a depth layer [S1]. This allows for order-independent transparency and volume containment checks.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_vds(mesh, width=1024, height=1024):
    """
    Generates a Volumetric Depth Sprite from a static mesh.
    """
    # Step 1: Transform shape to unit volume [0, 1]^3
    # Apply transformation matrix T_unit to mesh vertices
    unit_mesh = transform_to_unit_volume(mesh)
    
    # Set orthogonal projection covering [0, 1]^3
    projection = OrthogonalProjection(near=0.0, far=1.0)
    
    # Step 2: Determine depth complexity d_S
    # This requires a preliminary pass or analytical estimation to find max intersections
    d_S = compute_depth_complexity(unit_mesh)
    
    # Create 3D texture
    # Format: Luminance, 32-bit float, single channel
    # Dimensions: width x height x d_S
    texture = create_3d_texture(width, height, d_S, format=FLOAT32_LUMINANCE)
    
    # Configure texture parameters
    texture.interpolation = NEAREST
    texture.border_color = (1.0, 1.0, 1.0, 1.0) # 1 for each component
    texture.wrap_mode = CLAMP_TO_BORDER
    
    # Step 3: Depth Peeling via Render-to-Texture
    for layer_index in range(d_S):
        # Render the layer_index-th depth layer
        # Use inverse W-buffer to linearize depth values
        depth_buffer = render_depth_peel(unit_mesh, layer_index, projection, inverse_w_buffer=True)
        
        # Store linearized depth values in the 3D texture slice
        # Coordinate mapping: (s, t) in texture space corresponds to (s, t, layer_index/d_S)
        texture.set_slice(layer_index, depth_buffer)
        
    # Construct VDS structure
    vds = {
        "position": (0.0, 0.0, 0.0), # World position, typically origin for unit volume
        "matrix": identity_matrix(), # Linear transformation matrix
        "ldi_data": texture,
        "depth_complexity": d_S
    }
    return vds

def volumetric_parity_test(vds, point_world):
    """
    Classifies a point as inside or outside the volume using VDS.
    """
    # Transform point to LDI texture space
    # Inverse of vds.matrix and translation applied
    point_local = transform_point(vds.matrix, point_world)
    
    # Map to texture coordinates (s, t)
    s = point_local.x
    t = point_local.y
    
    # Check if point is within texture bounds
    if not (0 <= s <= 1 and 0 <= t <= 1):
        return OUTSIDE
        
    # Ray march along z-axis (depth)
    # Sample the 3D texture at (s, t, z) for z in [0, 1]
    # Count crossings of depth layers
    parity = 0
    z = 0.0
    step = 1.0 / vds.depth_complexity
    
    while z <= 1.0:
        # Sample depth value at current z
        # Note: The texture stores depth values. 
        # A crossing occurs if the ray's z-coordinate matches a stored depth layer.
        # In practice, this is often implemented by checking if the point's z 
        # falls between depth layers or by counting layer intersections.
        
        # Simplified parity logic based on VPT description:
        # The parity swaps if the ray passes a depth layer.
        depth_val = vds.ldi_data.sample(s, t, z)
        
        # If the sampled depth is valid (not border value 1), we are at a surface
        if depth_val != 1.0:
            parity += 1
            
        z += step
        
    return INSIDE if parity % 2 == 1 else OUTSIDE
```

### Parameters and Edge Cases
*   **Texture Resolution:** $1024^2$ pixels is recommended for fairly high quality. Lower resolutions like $128^2$ may fail for complex non-convex solids [S1].
*   **Undersampling Artifacts:** Planes nearly parallel to the LDI direction (z-axis) produce fuzzy edges. Mitigation involves applying multisampling to increase the number of samples [S1].
*   **Static Mesh Limitation:** The approach is limited to static meshes. Animated shapes require re-computation of the VDS, which cannot be done in real-time for complex shapes with high texture resolution [S1].
*   **Precision:** The accuracy of the volume test depends on the texture resolution. Higher resolution improves precision but increases memory usage [S1].

## Operating characteristics
*   **Time Complexity:** VDS creation takes up to two seconds per shape, depending on depth complexity and texture dimension [S1]. This pre-processing time is significant and prevents real-time updates for animated meshes.
*   **Memory Complexity:** The space complexity is high, proportional to $w \times h \times d_S$. For $1024^2$ resolution and high depth complexity, this can be substantial [S1].
*   **Performance:** Once generated, the Volumetric Parity Test (VPT) can be implemented efficiently on modern GPU hardware (Shader Model 3.0 compliant) for real-time point-in-volume tests [S1].
*   **Strengths:** Fast real-time volumetric tests for arbitrary 3D solids; view-independent representation; efficient GPU implementation.
*   **Limitations:** High space complexity; limited to static meshes; undersampling artifacts for parallel planes; precision depends on texture resolution [S1].

## Background
The concept of Layered Depth Images (LDIs) was introduced by Seidel et al. [SGwHS98], representing a scene from a single camera view with multiple pixels along each line of sight. The size of the representation grows linearly with the observed depth complexity. Depth peeling for order-independent transparency was introduced by Cabral et al. [Cas01]. The Volumetric Depth Sprite (VDS) extends these concepts to perform real-time volumetric tests, as presented in [S1].

## Sources
[S1] Real-Time Volumetric Tests Using Layered Depth Images, https://doi.org/10.2312/egs.20081019, fetched 2026-10-26
