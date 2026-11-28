# Analytic Rendering of Curvilinear Volume Data

Also called: Analytic rendering, Curvilinear Volume Rendering Algorithm, Front-to-back compositing of curvilinear voxels, Tricubic voxel rendering

## Overview
Analytic rendering of curvilinear volume data is a direct volume rendering technique for data sampled on curvilinear grids (six-sided cells with non-planar faces) that represents each voxel as a cubic triparametric Bezier solid [S1, unofficial]. It solves the problem of rendering deformed grids without interpolating to a rectilinear grid or approximating voxels with triangles/tetrahedra, ensuring $C^1$ continuity of both geometry and data values across voxel boundaries [S1, unofficial]. Unlike footprint-based methods that require recomputing footprints for varying voxel shapes, this method renders each voxel analytically directly into the accumulating image using a 3D forward difference operator [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   A curvilinear grid defined by vertices $P_{i,j,k}$.
*   Scalar data values associated with the grid vertices.
*   A transfer function mapping data values to color and opacity.
*   Camera parameters (viewpoint, projection).

**Outputs:**
*   An accumulated image buffer (color and opacity).

### Data Structures
**Voxel List Entry:**
Each entry in the voxel list contains:
*   `i, j, k`: Three integers representing the indices of one corner of the voxel in the grid.
*   `min_z`: One float representing the minimum Z-coordinate of the voxel in screen space.

**Bezier Solid Control Points:**
A cubic triparametric Bezier solid is defined by 64 control points. These consist of:
*   8 vertices from the curvilinear grid (end points).
*   56 interior control points (boundary and non-boundary) computed from the grid vertices.

**Opacity Lookup Table:**
A precomputed table mapping total voxel opacity $\alpha_{total}$ and layer count $n$ to layer opacity $\alpha_{layer}$ to avoid repeated exponential calculations during rendering.

### Algorithm Steps

1.  **Construct Voxel List:** Iterate through the curvilinear grid to create a list of voxels. For each voxel, store the $(i,j,k)$ indices of a corner and compute its minimum Z value in screen space.
2.  **Sort Voxels:** Sort the voxel list by `min_z` in ascending order.
    *   *Pitfall:* Sorting by minimum Z fails when voxels overlap in depth such that the voxel with the smaller Z is "mostly behind" the voxel with larger Z. In such cases, front-to-back compositing is incorrect, and voxel subdivision is required to achieve a valid ordering [S1, unofficial].
3.  **Orient Voxels:** Orient the parametric dimensions ($u, v$) of each voxel so that "most" forward differencing occurs parallel to the image plane. This maximizes the likelihood of detecting pixel collisions and minimizes forward difference calculations in depth [S1, unofficial].
4.  **Compute Control Points:** For each voxel, compute the 56 interior control points from the 8 grid vertices using the Catmull-Rom formulation to ensure $C^1$ continuity [S1, unofficial].
    *   Compute the derivative vector $D_r$ at point $P_i$:
        $$D_r = \frac{1}{2\beta}(P_{i+1} - P_{i-1})$$
        where $\beta$ controls the tension of the curve [S1, unofficial].
    *   Transform Catmull-Rom control points to Bezier control points ($B_0, B_1, B_2, B_3$):
        $$B_0 = P_{i-1}$$
        $$B_3 = P_{i+1}$$
        $$B_1 = B_0 + \frac{1}{3}D_r$$
        $$B_2 = B_3 - \frac{1}{3}D_{r+1}$$
    *   Compute boundary interior control points along the $u$-axis:
        $$\frac{\partial f}{\partial u} = \frac{1}{2\beta}(P_{i+1,j} - P_{i-1,j})$$
    *   Compute boundary interior control points along the $v$-axis:
        $$\frac{\partial f}{\partial v} = \frac{1}{2\beta}(P_{i,j+1} - P_{i,j-1})$$
    *   Compute non-boundary interior patch points using a tensor product of the derivative:
        $$\frac{\partial^2 f}{\partial u \partial v} = \frac{1}{36\beta^2}(P_{i+1,j+1} - P_{i-1,j+1} - P_{i+1,j-1} + P_{i-1,j-1})$$
    *   Compute the Bezier representation of a non-boundary control point $B_{1,1}$:
        $$B_{1,1} = P_{i,j} + \frac{\partial f}{\partial u} + \frac{\partial f}{\partial v} + \frac{\partial^2 f}{\partial u \partial v}$$
    *   Non-boundary interior points in the third dimension are computed analogously using the tensor product $\frac{\partial^3 f}{\partial u \partial v \partial w}$ [S1, unofficial].
5.  **Convert Bezier Form:** Convert the Bezier solid from "solid eight" form to point form suitable for the forward difference operator [S1, unofficial].
6.  **Render Voxel:** Evaluate the tricubic solid using a three-dimensional forward difference operator.
    *   The evaluation uses the matrix formulation:
        $$\sum_{k=0}^3 \sum_{j=0}^3 \sum_{i=0}^3 A_{i,j,k} = B^T G M G^T B$$
        where $G$ is the array of 64 control points, $M$ is the Bezier basis matrix, and $E$ is a function of step sizes [S1, unofficial].
7.  **Handle Opacity (Sub-voxel):** To handle opacity correctly at the sub-voxel level, decompose the voxel into $n$ uniform-thickness layers (sheets) and composite them front-to-back.
    *   The differential equation for front-to-back compositing is:
        $$\alpha_n = (1 - \alpha_{n-1}) \alpha + \alpha_{n-1}$$
    *   The layer opacity $\alpha_{layer}$ required to achieve a total voxel opacity $\alpha_{total}$ with $n$ layers is:
        $$\alpha_{layer} = 1 - \exp\left(-\frac{1}{n} \xi(\alpha_{total})\right)$$
        where $\xi(\alpha) = -\ln(1 - \alpha)$ [S1, unofficial].
    *   Precompute $\alpha_{layer}$ for a range of $\alpha_{total}$ and $n$ values and store in a lookup table [S1, unofficial].
    *   *Pitfall:* Using a linear transfer function for opacity accumulation treats volumetric data as unit-thickness material, causing the volume to appear too opaque and losing thickness perception [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def analytic_render_curvilinear_volume(grid_vertices, grid_data, camera, transfer_function, num_layers=10):
    """
    Renders a curvilinear volume using analytic tricubic Bezier solids.
    
    Args:
        grid_vertices: 3D array of shape (Nx, Ny, Nz) containing vertex positions.
        grid_data: 3D array of shape (Nx, Ny, Nz) containing scalar data values.
        camera: Object containing projection and view matrices.
        transfer_function: Function mapping data value to (color, opacity).
        num_layers: Number of sub-voxel layers for opacity compositing.
        
    Returns:
        image_buffer: Accumulated color and opacity buffer.
    """
    
    # 1. Construct Voxel List
    voxel_list = []
    for i in range(grid_vertices.shape[0] - 1):
        for j in range(grid_vertices.shape[1] - 1):
            for k in range(grid_vertices.shape[2] - 1):
                # Get 8 vertices of the voxel
                v = get_voxel_vertices(grid_vertices, i, j, k)
                # Compute min_z in screen space
                min_z = compute_min_z(v, camera)
                voxel_list.append({
                    'i': i, 'j': j, 'k': k,
                    'min_z': min_z,
                    'vertices': v,
                    'data': get_voxel_data(grid_data, i, j, k)
                })
    
    # 2. Sort Voxels by min_z
    voxel_list.sort(key=lambda x: x['min_z'])
    
    # 3. Precompute Opacity Lookup Table
    opacity_table = precompute_opacity_table(num_layers)
    
    # Initialize Image Buffer
    image_buffer = initialize_buffer()
    
    # 4. Render Each Voxel
    for voxel in voxel_list:
        # 4a. Compute Bezier Control Points
        control_points = compute_bezier_control_points(voxel['vertices'])
        
        # 4b. Orient Voxel (Simplified: assume optimal orientation for now)
        # In practice, determine u, v, w axes to maximize forward diff parallel to image plane
        u_axis, v_axis, w_axis = determine_optimal_axes(voxel['vertices'], camera)
        
        # 4c. Render Voxel using Forward Differencing
        render_bezier_solid(
            control_points=control_points,
            voxel_data=voxel['data'],
            camera=camera,
            transfer_function=transfer_function,
            opacity_table=opacity_table,
            num_layers=num_layers,
            u_axis=u_axis,
            v_axis=v_axis,
            w_axis=w_axis,
            image_buffer=image_buffer
        )
        
    return image_buffer

def compute_bezier_control_points(vertices):
    """
    Computes 64 Bezier control points from 8 curvilinear grid vertices.
    Uses Catmull-Rom formulation for C1 continuity.
    """
    # Extract indices for the 8 vertices of the voxel
    # P_000, P_100, P_010, P_110, P_001, P_101, P_011, P_111
    # For simplicity, assume vertices are ordered such that neighbors are accessible
    # In a full implementation, global grid indices are needed to access P_{i+1}, P_{i-1}, etc.
    
    # Placeholder for global grid access logic
    # P_i is the vertex at (i,j,k)
    
    # Compute derivatives for Catmull-Rom
    # D_r = 1/(2*beta) * (P_{i+1} - P_{i-1})
    # Beta is a tension parameter. Not given in sources, assume 1.0 or user-defined.
    beta = 1.0
    
    # Compute boundary derivatives
    # du = 1/(2*beta) * (P_{i+1,j} - P_{i-1,j})
    # dv = 1/(2*beta) * (P_{i,j+1} - P_{i,j-1})
    # dw = 1/(2*beta) * (P_{i,j,k+1} - P_{i,j,k-1})
    
    # Compute non-boundary derivatives (tensor products)
    # d2uv = 1/(36*beta^2) * (P_{i+1,j+1} - P_{i-1,j+1} - P_{i+1,j-1} + P_{i-1,j-1})
    # d2uw, d2vw analogously
    # d3uvw analogously
    
    # Transform to Bezier control points
    # B0 = P_{i-1}
    # B3 = P_{i+1}
    # B1 = B0 + 1/3 * Dr
    # B2 = B3 - 1/3 * D_{r+1}
    
    # Compute all 64 control points
    # B_{1,1,1} = P_{i,j,k} + du + dv + dw + d2uv + d2uw + d2vw + d3uvw
    # ... (all combinations)
    
    return control_points # Array of 64 points

def render_bezier_solid(control_points, voxel_data, camera, transfer_function, opacity_table, num_layers, u_axis, v_axis, w_axis, image_buffer):
    """
    Renders a single Bezier solid using 3D forward differencing.
    """
    # Convert Bezier from solid eight form to point form
    point_form = convert_bezier_to_point_form(control_points)
    
    # Get Bezier basis matrix M
    M = get_bezier_basis_matrix()
    
    # Determine step sizes based on camera and voxel size
    step_u, step_v, step_w = compute_step_sizes(camera, u_axis, v_axis, w_axis)
    
    # Initialize forward difference arrays
    # A is a 3D array of size 4x4x4
    A = initialize_forward_difference_arrays(step_u, step_v, step_w)
    
    # Evaluate the solid using the matrix formulation
    # Sum_{k,j,i} A_{i,j,k} = B^T G M G^T B
    # G is the control point array (64 points)
    # B is derived from E (step sizes) and M
    
    # For each pixel covered by the voxel's bounding box
    for pixel in voxel_screen_projection:
        # Compute initial forward differences for this pixel
        init_diffs = compute_initial_forward_diffs(pixel, camera, u_axis, v_axis, w_axis)
        
        # Iterate through the solid using forward differencing
        # This generates a series of points/surfaces within the voxel
        
        # Decompose into num_layers for opacity
        layer_opacity = opacity_table.lookup(voxel_data, num_layers)
        
        # Composite layers front-to-back
        accumulated_alpha = 0.0
        accumulated_color = (0, 0, 0)
        
        for layer in range(num_layers):
            # Evaluate color and opacity for this layer
            color, alpha = evaluate_layer(layer, init_diffs, point_form, M, step_u, step_v, step_w, transfer_function)
            
            # Apply layer opacity
            alpha = alpha * layer_opacity
            
            # Front-to-back compositing
            # alpha_n = (1 - alpha_{n-1}) * alpha + alpha_{n-1}
            # Note: The formula in the source is for the accumulated alpha, 
            # but typically compositing is: C_out = C_src * alpha_src + C_dst * (1 - alpha_src)
            # The source equation (6) alpha_n = (1 - alpha_{n-1}) * alpha + alpha_{n-1} 
            # seems to describe the accumulation of opacity itself if alpha is the layer opacity.
            # Standard compositing:
            # alpha_new = alpha_old + alpha_layer * (1 - alpha_old)
            # color_new = color_old * (1 - alpha_layer) + color_layer * alpha_layer
            
            # Using the source's differential equation form for opacity accumulation:
            alpha_new = (1 - accumulated_alpha) * alpha + accumulated_alpha
            
            # Color compositing (standard Porter-Duff over)
            color_new = color * alpha + accumulated_color * (1 - alpha)
            
            accumulated_alpha = alpha_new
            accumulated_color = color_new
            
            if accumulated_alpha > 1.0:
                break
                
        # Write to image buffer
        write_to_buffer(pixel, accumulated_color, accumulated_alpha, image_buffer)

def precompute_opacity_table(num_layers):
    """
    Precomputes layer opacity for a range of total opacities.
    alpha_layer = 1 - exp(-1/n * xi(alpha_total))
    xi(alpha) = -ln(1 - alpha)
    """
    table = {}
    alpha_values = linspace(0.0, 1.0, 100)
    for alpha_total in alpha_values:
        xi = -log(1 - alpha_total)
        alpha_layer = 1 - exp(-xi / num_layers)
        table[alpha_total] = alpha_layer
    return table
```

## Operating characteristics
**Time and Memory Complexity:**
Memory consumption is linear with respect to the size of the input dataset. The voxel list requires three integers and one float per voxel. Rendering stage memory is negligible, consisting of four 64-element double-precision arrays and a few matrices [S1, unofficial].

**Measured Performance:**
The method is significantly more expensive in CPU time compared to trilinear interpolation or hardware splatting technologies. The trade-off is image quality, with tricubic rendering providing significantly better detail, particularly for small vortical structures in CFD datasets [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Provides $C^1$ continuous shape and data, resulting in superior detail compared to trilinear interpolation. Handles different projected voxel shapes without footprint recomputation.
*   **Limitations:** High computational cost. Sorting by minimum Z can fail for overlapping voxels, requiring subdivision. Linear transfer functions lead to incorrect thickness perception [S1, unofficial].

**When to Use:**
Use when high-quality, continuous rendering of curvilinear grid data is required and computational resources are sufficient. Avoid when real-time performance is critical or when voxel overlap is extensive and subdivision is prohibitive [S1, unofficial].

## Background
**Origin:**
The method was presented by Wes Bethel from the Graphics Group at the Computing Resources Department, Lawrence Berkeley Laboratory, Berkeley, California [S1, unofficial].

**History and Development:**
The technique builds on previous work in curvilinear volume rendering, such as [Wil90] and [Shi90], which approximated voxels with triangles or tetrahedra. It also references [Kau87] for the forward difference operator and [Mor85] for the tensor product derivative formulation. The method addresses issues with opacity accumulation identified in [Lau91] and [Wil91] [S1, unofficial].

**Applications:**
Primarily used in scientific visualization for computational fluid dynamics (CFD) data, such as turbulence boundary layer analysis [S1, unofficial].

## Sources
[S1] Analytic rendering of curvilinear volume data, https://doi.org/10.2172/10112506, fetched 2026-10-26 (unofficial)
