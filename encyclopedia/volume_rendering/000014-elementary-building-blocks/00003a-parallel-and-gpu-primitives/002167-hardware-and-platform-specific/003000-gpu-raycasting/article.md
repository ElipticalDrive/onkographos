# GPU Raycasting

Also called: GPU Fragment Program Ray Casting, Hardware-accelerated Volume Raycasting, Ray-casting algorithm base on GPU, Ray-casting algorithm based GPU, Stegmaier's approach, rectilinear volume ray caster on GPU

## Overview
GPU Raycasting is a volume rendering technique that generates 2D images from 3D data fields by casting rays through the volume, where the calculation of resampling and image synthesis is processed by the GPU fragment shader for parallel computation [S2]. It approximates the volume rendering integral by sampling at a finite number of positions along each ray [S1, unofficial]. Unlike path tracing or methods requiring explicit visibility ordering, it sidesteps these limitations by relying on the GPU's parallel fragment processing, though it introduces accuracy limitations related to ray-triangle intersections and requires all data to reside in GPU memory [S4, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** 3D volume data (loaded as a 3D texture), transfer functions (loaded as 1D textures for color and opacity), ray entry point ($p_{in}$), ray exit point ($p_{out}$), and ray direction parameters.
*   **Outputs:** A 2D color buffer with associated alpha channel representing the composited volume image.

**Data Structures:**
*   **3D Texture:** Stores the scalar volume data.
*   **1D Textures:** Store the transfer functions mapping scalar values to color and opacity.
*   **Fragment Shader State:** Holds the current accumulated color (`destColor`), remaining opacity (`remainOpacity`), current sampling position (`pos`), and loop counters.

**Parameters:**
*   $p_{in}$: Position of the ray entering the data volume [S2].
*   $p_{out}$: Position of the ray running out of the data volume [S2].
*   $rayDir$: The distance/step size between resampling points [S2].
*   $t_{Max}$: The total number of sampling points (maximum traversal distance in steps) [S2].
*   Threshold: `0.0039` (equivalent to $1/255$), used as the minimum remaining opacity to continue sampling [S2].

**Algorithm Steps:**
1.  **Initialization:** Load the 3D volume data and 1D transfer functions (color and opacity) into GPU memory [S2]. Initialize the accumulated color `destColor` to an initial value (e.g., background color) and `remainOpacity` to $1.0 - \text{destColor}.a$ [S2].
2.  **Ray Setup:** Calculate the normalized ray direction vector $\vec{d} = \text{normalize}(p_{out} - p_{in})$ [S2].
3.  **Sampling Loop:** While the sampling point is inside the volume (`inside == TRUE`):
    a.  **Position Calculation:** Calculate the current sampling position $pos$ using the index $i \in \mathbb{N}$:
        $$pos = p_{in} + i \cdot rayDir \cdot \vec{d}$$
        [S2]
    b.  **Boundary Check:** If $pos - p_{in} > p_{out} - p_{in}$, the point is out of volume data; terminate loop [S2].
    c.  **Texture Sampling:** Sample the volume texture at `pos` to get the scalar `value`:
        $$value \leftarrow \text{texture3D}(\text{dataSetTexture}, pos)$$
        [S2]
    d.  **Transfer Function Lookup:** Map the scalar value to a property `scalar` and lookup the corresponding `opacity` from the opacity 1D texture:
        $$scalar \leftarrow \text{scalarFromValue}(value)$$
        $$opacity \leftarrow \text{texture1D}(\text{opacityTexture}, scalar)$$
        [S2]
    e.  **Shading and Compositing:** If `opacity > 0.0`:
        i.  Calculate the `color` contribution (shading):
            $$color \leftarrow \text{shade}(value)$$
            [S2]
        ii. Apply opacity to color:
            $$color \leftarrow color \cdot opacity$$
            [S2]
        iii. Accumulate color and update remaining opacity using front-to-back blending:
            $$destColor \leftarrow destColor + color \cdot remainOpacity$$
            $$remainOpacity \leftarrow remainOpacity \cdot (1.0 - opacity)$$
            [S2]
    f.  **Advance Ray:** Increment the position and step counter:
        $$pos \leftarrow pos + rayDir$$
        $$t \leftarrow t + 1.0$$
        [S2]
    g.  **Termination Check:** Update `inside` flag:
        $$inside \leftarrow (t < t_{Max}) \land (remainOpacity \ge 0.0039)$$
        [S2]
4.  **Output:** Write the final `destColor` and `remainOpacity` (converted to alpha) to the framebuffer:
    $$gl\_FragColor \leftarrow destColor$$
    $$gl\_FragColor.a \leftarrow 1.0 - remainOpacity$$
    [S2]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Fragment Shader Entry Point
def fragment_shader(p_in, p_out, ray_dir, t_max, volume_texture, color_tex, opacity_tex):
    # 1. Initialization
    dest_color = initial_color() # e.g., background color
    remain_opacity = 1.0 - dest_color.a
    pos = p_in
    t = 0.0
    
    # Normalized direction vector
    dir_vec = normalize(p_out - p_in)
    
    # 2. Sampling Loop
    while True:
        # Boundary check: if distance from pin exceeds total length
        if distance(pos - p_in) > distance(p_out - p_in):
            break
            
        # Sample volume data
        value = texture3D(volume_texture, pos)
        
        # Get scalar and opacity
        scalar = scalar_from_value(value)
        opacity = texture1D(opacity_tex, scalar)
        
        if opacity > 0.0:
            # Shade the point
            color = shade(value)
            color = color * opacity
            
            # Front-to-back compositing
            dest_color = dest_color + color * remain_opacity
            remain_opacity = remain_opacity * (1.0 - opacity)
            
        # Advance
        pos = pos + ray_dir
        t = t + 1.0
        
        # Termination conditions
        if t >= t_max:
            break
        if remain_opacity < 0.0039: # 1/255
            break
            
    # 3. Output
    return dest_color, 1.0 - remain_opacity
```

## Subtopics

### [GPU Ray Casting with Fast Intersection](7032-gpu-ray-casting-with-fast-intersection.md)
Implements a single-pass technique testing each ray for intersection exactly once per pass, supporting one object per pass. It avoids complex hierarchical empty-space skipping structures, relying on a simplified fast intersection test effective for single-intersection scenarios. It differs from general GPU ray casting by optimizing for single intersection and using specific normalized camera projection geometry.

### [GPU-based Ray Casting](7131-gpu-based-ray-casting.md)
A rendering technique for unstructured grids that sidesteps visibility order limitations and path tracing (PT) accuracy issues. It differs from PT by requiring all data to reside in GPU memory, which limits dataset size, whereas PT streams data from CPU memory.

### [GPU Ray Casting with Depth Peeling and Tile-Based Subdivision](7813-gpu-ray-casting-with-depth-peeling-and-tile-based-subdivisio.md)
Improves GPU-based ray casting for unstructured grids by combining tile-based screen subdivision with depth peeling. It handles non-convex meshes efficiently, solving the "imaginary cells" problem caused by mesh non-convexities without requiring convexification, while reducing fragment processing overhead.

### [Recursive Ray-casting for Global Illumination](8544-recursive-ray-casting-for-global-illumination.md)
An extension that computes specular reflection and refraction effects at interactive frame rates by processing a ray-caster multiple times with different entry and exit points. It supports both Direct Volume Rendering (DVR) and isosurface rendering, aiming to combine specular effects within a single image.

### [Single Scattering Illumination](9628-single-scattering-illumination.md)
A volumetric lighting model computing radiance from light undergoing exactly one scattering event. It distinguishes between Direct Single Scattering (DSS) and Indirect Single Scattering (ISS), providing analytic formulations for both, including occlusion handling via shadow volumes for real-time hardware rendering.

### [Distributed Volume Raycasting with Bricks](9869-distributed-volume-raycasting-with-bricks.md)
A parallel technique subdividing volumetric data into convex subvolumes (bricks) assigned to separate cluster nodes for independent GPU raycasting. It uses software-based compositing to avoid hardware blending limitations and `glReadPixels` bottlenecks.

### FPGA-based Raycasting
An FPGA-based system introduced in 1997 by the University of Tübingen for perspective view raycasting. It requires long pre-processing but achieves 10 Hz performance for a $256^3$ volume [S5, unofficial].

### Stochastic Ray Shifting
A technique applied during ray casting to reduce banding artifacts by shifting ray origins stochastically. It is controlled by a boolean UI parameter "Apply Noise" that toggles the application of this shifting [S6].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of samples per ray and the number of pixels (fragments). Parallelism is exploited across pixels via the fragment shader [S1, unofficial].
*   **Memory Complexity:** Requires the entire 3D volume dataset and transfer functions to reside in GPU memory [S4, unofficial]. This limits the maximum dataset size compared to methods that stream data from CPU memory [S4, unofficial].
*   **Performance:** Feasible from a computational standpoint due to advances in graphics hardware and algorithms [S3, unofficial]. Performance scales with consumer graphics boards and SIMD instruction sets, which can avoid costly conditionals through masking operations [S1, unofficial].
*   **Strengths:** Sidesteps the explicit need for computing visibility order and many accuracy issues associated with Path Tracing (PT) [S4, unofficial]. Simple and robust implementation [S4, unofficial].
*   **Limitations:** Accuracy limitations of ray-triangle intersections can lower image quality [S4, unofficial]. The requirement for all data in GPU memory restricts dataset size [S4, unofficial].
*   **When to Use:** Suitable for interactive visualization of volumetric data where hardware acceleration is available and dataset sizes fit within GPU memory [S3, unofficial].
*   **When Not to Use:** When datasets exceed available GPU memory or when high-accuracy path tracing is required despite the complexity [S4, unofficial].

## Background
GPU Raycasting was introduced by Kruger in 2003, making it feasible from a computational standpoint due to advances in graphics hardware and algorithms [S3, unofficial]. It became a milestone for medical visualization, particularly for studying the evolution of brain white matter lesions and representing diffusion tensor data [S3, unofficial]. The technique leverages the parallel processing power of the GPU fragment shader to perform resampling and image synthesis, moving away from CPU-based ray casting limitations [S2].

## Sources
[S1] Interactive Volume Rendering Techniques, https://www.scribd.com/document/479714183/Interactive-Volume-Rendering-with-Ray-Tracing, fetched 2026-10-25 (unofficial)
[S2] Volume Rendering of 3D Borehole Data Based on GPU, https://doi.org/10.2991/icsem.2013.181, fetched 2026-10-26
[S3] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
[S4] A Survey of GPU-Based Volume Rendering of Unstructured Grids., https://www.researchgate.net/publication/220162223_A_Survey_of_GPU-Based_Volume_Rendering_of_Unstructured_Grids, fetched 2026-10-28 (unofficial)
[S5] Volume Ray Casting Techniques, https://www.scribd.com/document/102700078/03-raycasting, fetched 2026-10-25 (unofficial)
[S6] VStudio - A Framework for Interactive Volume Visualization, https://www.cg.tuwien.ac.at/courses/Visualisierung/2008-2009/Beispiel1/Hausmair/index.html, fetched 2026-10-27
