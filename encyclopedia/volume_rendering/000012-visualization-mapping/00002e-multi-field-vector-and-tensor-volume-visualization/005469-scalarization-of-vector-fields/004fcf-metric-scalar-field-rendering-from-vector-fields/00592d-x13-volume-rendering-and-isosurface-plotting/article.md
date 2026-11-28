# X13 Volume Rendering and Isosurface Plotting

Also called: 3D isosurface figures, GLMakie volume render

## Overview
X13 is a post-processing script that generates 3D volume renders and isosurface plots of scalar fields derived from vector field data (specifically velocity components) using the GLMakie library in Julia. It solves the problem of visualizing 3D turbulent flow structures (such as streamwise vortices) that are lost when reducing simulation data to 2D slices. It is distinguished from its sibling method X12, which uses orthogonal cutting planes via matplotlib, by rendering through the volume using GPU raymarching, providing a more immersive view of internal structures at the cost of requiring an OpenGL context (hardware or software rasterized).

## How it works
**Inputs and Outputs:**
- **Input:** NetCDF or HDF5 files containing simulation snapshots. Fields include velocity components ($u, v, w$) and optional derived fields.
- **Output:** PNG images of 3D figures containing volume renders or isosurfaces.

**Data Structures:**
- The script reads data into memory arrays. For HDF5 virtual datasets (from multi-GPU merges), the storage must be registered before reading to avoid `KeyError` [S1, unofficial].
- Grid data is assumed to be stored at cell centers for all velocity components, allowing direct gradient computation without staggered interpolation [S1, unofficial].

**Parameters:**
- `--field`: Comma-separated list of fields to render. Single field layouts panels by mode (volume/isosurface); multiple fields layout by field [S1, unofficial].
- `--mode`: 'volume' or 'isosurface' [S1, unofficial].
- `--overlay`: Name of a reference field (e.g., 'b') to draw as contours on vertical bounding walls [S1, unofficial].
- `--clim-percentile`: Upper percentile for color range, default 99 [S1, unofficial].
- `--zlim`: Z-cropping limit. Default 5 (wider than plotting scripts' 4) to allow centred differences [S1, unofficial].
- `--cols`: Number of columns in panel grid, default 3 [S1, unofficial].
- `--scale`: Appends suffix to budget terms [S1, unofficial].

**Steps:**

1. **Initialization and Virtual Storage Handling:**
   If the input file is an HDF5 virtual dataset, call `allow_virtual_storage!()` to register the storage code, preventing `KeyError` during read operations [S1, unofficial].

2. **Data Loading and Cropping:**
   Load the specified fields from the NetCDF/HDF5 file. Crop the z-dimension to the active layer defined by `--zlim`. Keep one cell of margin on each side of the cropped slab to maintain accuracy for centred differences [S1, unofficial].
   *Note:* Cropping *before* differentiation is critical to reduce memory usage. Cropping after differentiation causes memory peaks of 15–25 GB at $N_z=1024$ [S1, unofficial].

3. **Derivation of Scalar Fields:**
   If the requested fields are not present in the file, derive them from the velocity components $u, v, w$. Common derived fields include:
   - **Q-criterion (Q):** Second invariant of the velocity gradient tensor.
   - **Enstrophy:** Magnitude squared of the vorticity vector.
   - **Speed:** Magnitude of the velocity vector.
   *Note:* The file's own $\omega$ is often only the spanwise component, which is insufficient for 3D structure analysis. All three velocities are at cell centers, so the gradient tensor is computed using centred differences on the single grid [S1, unofficial].

4. **Gradient Computation:**
   Compute the velocity gradient tensor $\nabla \mathbf{u}$ using centred differences.
   - In x and y (periodic boundaries): Use centred differences.
   - In z (bounded boundaries): Use one-sided differences at the walls, matching the pipeline's treatment of the bounded axis [S1, unofficial].
   Formula for centred difference of a field $f$ at index $i$:
   $$ \frac{\partial f}{\partial x} \approx \frac{f_{i+1} - f_{i-1}}{2 \Delta x} $$
   This requires the one-cell margin established in Step 2 [S1, unofficial].

5. **Isosurface Level Calculation:**
   Set isosurface levels symmetrically around zero. For each desired magnitude level $L$, create a pair of levels $[-L, L]$. Log these levels for visibility (e.g., -0.0849, -0.0299, 0.0299, 0.0849) [S1, unofficial].
   - For positive-definite fields, clamp positive-field isosurface levels into the color range. The top level is set to the 0.995 quantile of the data, while the color range ends at the `--clim-percentile` (default 99) quantile to prevent saturation [S1, unofficial].

6. **Figure Layout and Rendering:**
   - **Panel Resolution:** Set panel resolution to 820x560 per panel to accommodate the cropped domain aspect ratio (roughly 14:4.7:8) [S1, unofficial].
   - **Camera Orientation:** Set camera azimuth to $1.30\pi$ to orient the streamwise direction (x) across the frame width while keeping the spanwise direction (y) readable [S1, unofficial].
   - **Color Scaling:** Use a fixed fraction of each panel's own range for isosurface levels: 0.18, 0.515, and 0.85. This ensures a surface means the same thing across panels. Use a symmetric `:balance` color scale for all panels, including positive-definite fields, leaving the lower half unused to indicate the quantity never changes sign [S1, unofficial].
   - **Overlay:** If `--overlay` is specified, draw contours of the reference field on the two vertical bounding walls. This shows the location of flow structures (e.g., billows) relative to the isosurfaces. Do not use isosurfaces for the overlay, as transparency cannot separate interpenetrating isosurfaces in GLMakie (alpha is ignored on volume contours) [S1, unofficial].

7. **Batch Rendering Setup:**
   For off-screen rendering on clusters without GPUs/displays, use Xvfb and Mesa software rasterizer (`LIBGL_ALWAYS_SOFTWARE=1`). Start Xvfb on a free display, smoke-test the OpenGL context, render, and kill the server [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
function render_x13(input_file, args):
    # 1. Handle HDF5 virtual datasets
    if is_hdf5_virtual(input_file):
        allow_virtual_storage!()

    # 2. Load and crop data
    # Load u, v, w and requested fields
    data = load_dataset(input_file, fields=args.field)
    
    # Crop z to active layer with margin for centred differences
    z_limit = args.zlim if args.zlim else 5
    margin = 1
    z_start = max(1, 1 + margin)
    z_end = min(Nz, Nz - margin)
    
    # Crop before differentiation to save memory
    data_cropped = crop_z(data, z_start, z_end)
    
    # 3. Derive fields if necessary
    if "Q" in args.field and "Q" not in data_cropped:
        # Compute gradient tensor using centred differences
        # Periodic in x, y; one-sided at z walls
        grad_u = compute_gradient(data_cropped.u, data_cropped.deltax, periodic_x=True, periodic_y=True, bounded_z=True)
        grad_v = compute_gradient(data_cropped.v, data_cropped.deltay, periodic_x=True, periodic_y=True, bounded_z=True)
        grad_w = compute_gradient(data_cropped.w, data_cropped.deltaz, periodic_x=True, periodic_y=True, bounded_z=True)
        
        # Compute Q-criterion (example derivation)
        # Q = 0.5 * (|omega|^2 - |S|^2) where omega is vorticity, S is strain rate
        vorticity = compute_vorticity(grad_u, grad_v, grad_w)
        strain = compute_strain(grad_u, grad_v, grad_w)
        data_cropped.Q = 0.5 * (norm(vorticity)^2 - norm(strain)^2)
        
    # 4. Calculate isosurface levels
    levels = []
    for magnitude in desired_magnitudes:
        levels.append(-magnitude)
        levels.append(magnitude)
        
    # Clamp positive fields
    if is_positive_definite(field):
        top_level_quantile = quantile(field_data, 0.995)
        clim_percentile = args.clim_percentile # default 99
        clim_limit = quantile(field_data, clim_percentile / 100.0)
        # Ensure levels are within clim range
        levels = [l for l in levels if abs(l) <= clim_limit]

    # 5. Render
    fig = Figure(resolution=(820 * cols, 560 * rows))
    
    for panel_index, field_name in enumerate(args.field):
        ax = Axis3(fig[panel_index])
        
        # Set camera azimuth
        ax.camera.azimuth = 1.30 * pi
        
        # Draw isosurfaces or volume
        if args.mode == "isosurface":
            # Use symmetric levels
            draw_isosurfaces(ax, data_cropped[field_name], levels=levels, 
                             color_scale=:balance, symmetric=true)
        else:
            draw_volume(ax, data_cropped[field_name])
            
        # Overlay reference field on walls if specified
        if args.overlay:
            overlay_field = data_cropped[args.overlay]
            draw_contours_on_walls(ax, overlay_field, z_boundaries=[z_start, z_end])
            
        # Set color limits
        clim = quantile(data_cropped[field_name], args.clim_percentile / 100.0)
        set_color_limits!(ax, clim=clim)
        
    # Save figure
    save("output.png", fig)
```

## Operating characteristics
- **Time Complexity:** Rendering time is approximately 90 seconds per six-panel figure at $N_z=1024$ on a Casper compute node. A full 101-record set takes ~2.5 hours [S1, unofficial].
- **Memory Complexity:** Memory usage is significantly reduced by cropping the z-dimension before differentiation. A subsetter script can reduce file size from 2.15 GiB to 0.006 GiB (336x compression) for testing [S1, unofficial]. Without cropping, memory peaks can reach 15–25 GB at $N_z=1024$ [S1, unofficial].
- **Hardware Requirements:** Requires an OpenGL context. Can run on CPU-only clusters using Xvfb and Mesa software rasterizer. Does not require a dedicated GPU [S1, unofficial].
- **Strengths:** Provides immersive 3D visualization of internal flow structures (e.g., vortices) that 2D slices miss. Efficient batch rendering on CPU clusters.
- **Limitations:** Transparency cannot separate interpenetrating isosurfaces. Requires careful tuning of isosurface levels and color scales for multi-field figures. Not suitable for batch jobs without any display/OpenGL support (use X12 instead) [S1, unofficial].

## Background
X13 was developed as part of the `tomchor/CoarseGrainedKHAPE` repository for rendering 3D isosurfaces of turbulent flow simulations. It was introduced in October 2026 as a companion to X12, which used orthogonal cutting planes. X13 leverages GLMakie for GPU raymarching volume rendering, addressing the need to visualize 3D structures like streamwise vortices in Kelvin-Helmholtz (KH) runs. It runs on the Casper compute cluster using off-screen rendering techniques [S1, unofficial].

## Sources
[S1] 3D isosurface figures: X13, a batch render job, and a subsetter, https://github.com/tomchor/CoarseGrainedKHAPE/pull/78, fetched 2026-10-28 (unofficial)
