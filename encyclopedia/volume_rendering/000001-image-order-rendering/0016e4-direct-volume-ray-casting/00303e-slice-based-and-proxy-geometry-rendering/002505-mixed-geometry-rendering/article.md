# Mixed Geometry Rendering

Also called: Combining geometric objects with volumetric data, Direct Volume Rendering with Z-Buffer Geometry, Intermixing Volumes and Geometry, Wireframe and Surface Combination

## Overview
Mixed geometry rendering integrates explicit geometric primitives (isosurfaces, bounding boxes, wireframes) with volumetric data in a single image to provide spatial context and structural clarity [S2, book][S1]. It solves the problem of isolating volumetric features from their surrounding environment by allowing users to see both the internal scalar field distribution and the discrete boundaries or grid structures that define the data domain [S2, book][S1]. Unlike pure volume rendering, which treats the dataset as a continuous medium, this method explicitly renders surface geometry, often using the hardware depth buffer to occlude volume data behind opaque surfaces [S2, book].

## How it works
The method adapts the rendering pipeline based on whether the volume rendering is performed via hardware texture mapping or software ray casting/splatting. The core challenge is maintaining correct depth ordering between opaque geometry, semi-transparent geometry, and the volume samples.

### Inputs and Outputs
*   **Inputs**: Volumetric scalar field data (3D texture or cell-organized grid), geometric primitives (triangles, wireframes), camera parameters, transfer functions (scalar-to-color/opacity mapping), and lighting parameters.
*   **Outputs**: A 2D image where geometry and volume are composited correctly according to depth.

### Data Structures
1.  **Volumetric Data**: Stored as a 3D texture (for hardware) or in CPU memory (for software ray casting) [S2, book][S1].
2.  **Transfer Function**: A 1D texture mapping scalar values to RGBA [S2, book].
3.  **Geometry**: Standard triangle meshes or line segments [S2, book][S1].
4.  **Depth Buffer**: Hardware Z-buffer storing the depth of the closest opaque geometry [S2, book].
5.  **A-Buffer (Variant)**: A linked list of fragments per pixel, sorted by depth, used in scanline hybrid rendering [S3, unofficial].

### Steps

#### 1. Gradient Computation (For Lighting)
To model light interaction with the volume (e.g., for wireframes or implicit surfaces within the volume), the gradient at each voxel is computed [S1].
*   Let $A$ represent the differences in coordinates and values between three neighboring points.
*   The X component of the gradient is calculated as the ratio of two determinants [S1]:
    $$
    \nabla_x = \frac{\det \begin{pmatrix} A_{f0} & A_{y0} & A_{z0} \\ A_{f1} & A_{y1} & A_{z1} \\ A_{f2} & A_{y2} & A_{z2} \end{pmatrix}}{\det \begin{pmatrix} A_{x0} & A_{y0} & A_{z0} \\ A_{x1} & A_{y1} & A_{z1} \\ A_{x2} & A_{y2} & A_{z2} \end{pmatrix}}
    $$
    Where $A_{xi}, A_{yi}, A_{zi}$ are coordinate differences and $A_{fi}$ are value differences for three pairs of points [S1].
*   The Y and Z components are computed similarly by substituting the value differences into the second and third columns of the numerator determinant, respectively [S1].

#### 2. Geometry and Volume Preparation
*   **Hardware Volume Rendering**:
    1.  Render all **opaque** geometric primitives first [S2, book].
    2.  Sort semi-transparent geometric primitives and texture-mapped volume polygons in back-to-front order. This may require splitting polygons to ensure correct sorting [S2, book].
*   **Software Volume Rendering (Ray Casting/Splatting)**:
    1.  Render opaque geometry and capture the results into the hardware depth buffer [S2, book].
    2.  For ray casting: Convert the depth value $Z$ for each pixel into a distance $t$ along the view ray. This $t$ bounds the segment of the ray considered during volume rendering [S2, book].
    3.  For splatting: Compare the depth of every volume sample to the value stored in the depth buffer. Accumulate the sample's contribution only if the sample is in front of the geometry for that pixel [S2, book].

#### 3. Volume Rendering and Compositing
*   **Hardware Path**:
    1.  Enable alpha blending with Source = `GL_ONE`, Destination = `GL_ONE_MINUS_SRC_ALPHA` [S2, book].
    2.  Draw the sorted semi-transparent geometry and volume polygons.
    3.  The fragment shader samples the volume, looks up the transfer function, and outputs color/opacity for hardware compositing [S2, book].
*   **Software Path**:
    1.  Perform ray marching or splatting for the volume, respecting the depth bounds from the geometry [S2, book].
    2.  Convert scalar values to opacity and color using mapping functions [S1].
    3.  Integrate opacity and color along the ray [S1].
    4.  Blend the final volume color with the geometric color using the over operator [S2, book].
    5.  Specifically, multiply the stored geometric color in the pixel by $(1 - \text{integral of opacity along the ray})$ and add this product to the volume ray color [S1].

#### 4. Wireframe Specifics
*   Wireframe geometry is rendered using a standard Z-buffer technique [S1].
*   **Pitfall**: Wireframe geometry completely occludes volume data behind it. However, volume data in front of the wireframe correctly transmits colors from the geometry [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_mixed_geometry(volume_data, geometry, camera, transfer_function):
    # 1. Setup Depth Buffer for Geometry
    # Clear depth buffer
    clear_depth_buffer()
    
    # 2. Render Opaque Geometry
    # Set blend mode to None (opaque)
    set_blend_mode(OPAQUE)
    for prim in geometry.opaque_primitives:
        rasterize(prim)
        
    # 3. Prepare Volume Rendering Context
    # For software ray casting, read depth buffer to get bounds
    depth_buffer = read_depth_buffer()
    
    # 4. Render Semi-Transparent Geometry and Volume
    # Sort semi-transparent geometry and volume polygons back-to-front
    sorted_primitives = sort_back_to_front(geometry.transparent_primitives, volume_polygons)
    
    set_blend_mode(SRC_ONE, GL_ONE_MINUS_SRC_ALPHA)
    
    for prim in sorted_primitives:
        if prim.is_geometry:
            rasterize(prim)
        elif prim.is_volume:
            # Hardware texture mapping path
            render_volume_polygon(prim, volume_data, transfer_function)
            
    # 5. Software Ray Casting Path (Alternative to Hardware)
    # If using software ray casting instead of hardware polygons:
    for y in range(height):
        for x in range(width):
            # Get depth bound from opaque geometry
            z_depth = depth_buffer[y, x]
            t_bound = convert_z_to_t(z_depth, camera)
            
            # Ray march volume, stopping at t_bound
            color_vol, opacity_vol = ray_march_volume(
                volume_data, 
                camera.ray(x, y), 
                max_t=t_bound,
                transfer_function=transfer_function
            )
            
            # Get geometric color from depth buffer (already rendered)
            color_geo = color_buffer[y, x]
            
            # Composite: Color_geo * (1 - opacity_vol) + Color_vol
            final_color = color_geo * (1.0 - opacity_vol) + color_vol
            color_buffer[y, x] = final_color

    return color_buffer
```

## Subtopics

### Hybrid Scanline A-Buffer Volume Rendering
A hybrid rendering system that uses a fast scanline a-buffer algorithm for surface-defined objects and volume rendering for procedural gases, combining them via Z-sorted a-buffer fragments [S3, unofficial]. It creates an a-buffer for a scanline containing a list for each pixel of all fragments that partially or fully cover the pixel. For each volume covering the scanline, volume rendering is performed, creating a-buffer fragments for separate sections of the volumes. Volume rendering ceases once full coverage of the pixel by volume or surface-defined elements is achieved. These volume a-buffer fragments are sorted into the a-buffer fragment list based on their average Z-depth values, and the list is rendered to produce the final color [S3, unofficial]. It uses a physically-based low-albedo illumination and atmospheric attenuation model for gases, with volumetric shadows combined using three-dimensional shadow tables [S3, unofficial].

## Operating characteristics
*   **Time Complexity**: The complexity of volume rendering is $O(n^3)$ if every voxel is visited [S2, book]. Performance is heavily dependent on the resolution and the number of geometry primitives.
*   **Memory Usage**: High for cell-organized data. For example, a model with 381,548 tetrahedral cells required over 50 Megabytes of memory [S1].
*   **Performance**: On a Stardent ST1000 (5 Mips processors), rendering a 512x512 window for a bearing housing (16,896 tetrahedral cells) took 149.2 seconds, and a CFD analysis (381,548 cells) took 591.6 seconds [S1].
*   **Strengths**: Provides clear structural context (e.g., bounding boxes, cut planes) alongside volumetric data [S2, book]. Efficient for hardware-accelerated opaque geometry integration [S2, book].
*   **Limitations**: Wireframe geometry can incorrectly occlude volume data behind it [S1]. High memory consumption for large cell-organized datasets [S1]. Sorting semi-transparent geometry and volume polygons can be computationally expensive and may require polygon splitting [S2, book].

## Background
The technique originated in the late 1980s as a way to combine the structural clarity of geometric modeling with the continuous data representation of volume rendering. Early implementations focused on finite element models and computational fluid dynamics (CFD) visualizations [S1]. The method was detailed in works such as "Raytracing irregular volume data" (1989) which described combining wireframe grids with ray-traced volume data [S1], and later formalized in VTK documentation as a standard approach for intermixing volumes and geometry [S2, book]. Hybrid approaches using a-buffers for gases and surfaces were also explored in the context of procedural modeling and animation [S3, unofficial].

## Sources
[S1] Raytracing irregular volume data, https://doi.org/10.1145/99307.99316, fetched 2026-10-25
[S2] Chapter 7 - Advanced Computer Graphics - VTK, https://examples.vtk.org/site/VTKBook/07Chapter7/, fetched 2026-10-29 (book)
[S3] Procedural Modeling, Animation, and Rendering of Gases, Fluids, and Textures, https://www.researchgate.net/profile/David-Ebert-4/publication/2786953_Procedural_Modeling_Animation_and_Rendering_of_Gases_Fluids_and_Textures/links/00b7d51a73e42a3c4e000000/Procedural-Modeling-Animation-and-Rendering-of-Gases-Fluids-and-Textures.pdf, fetched 2026-10-31 (unofficial)
