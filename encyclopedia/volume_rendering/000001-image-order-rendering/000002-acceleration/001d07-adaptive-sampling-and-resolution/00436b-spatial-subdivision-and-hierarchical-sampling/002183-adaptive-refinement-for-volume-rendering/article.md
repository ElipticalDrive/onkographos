# Adaptive Refinement for Volume Rendering

Also called: Adaptive Refinement Volume Rendering, Image-order adaptive refinement, Interactive Adaptive Refinement, Feedback-based Adaptive Refinement

## Overview
Adaptive Refinement for Volume Rendering is an image-order technique that progressively improves visual fidelity by casting rays non-uniformly based on local image complexity, balancing rendering quality and performance in interactive settings [S3]. It sets itself apart from its siblings by either using a feedback loop to maintain a target frame rate while bounding error per texel [S1] or by employing a rigorous Finite Element Method (FEM)-based error estimator to guide h-adaptive subdivision of the image plane [S2, unofficial], whereas [Adaptive Screen Sampling](5838-adaptive-screen-sampling.md) typically relies on variance-based metrics and [Adaptive Ray Shooting & Quadtree Subdivision](8158-adaptive-ray-shooting-quadtree-subdivision.md) often focuses on object-space or hybrid approaches.

## How it works
The method operates as a reactive control loop or an iterative approximation sequence that adjusts sampling density to maintain a target frame rate or minimize $L^2$ error [S1][S2, unofficial]. It builds on the shared principles of [Adaptive Sampling and Resolution](1412-adaptive-sampling-and-resolution.md).

### Inputs and Outputs
*   **Inputs:** 3D scalar field data (voxels), transfer function (mapping scalar to color/opacity), camera parameters, and user-defined parameters (target frame rate, error thresholds, initial resolution bounds) [S1][S2, unofficial][S3].
*   **Outputs:** A rendered image (framebuffer) and an error map or flag array indicating computed vs. interpolated regions [S2, unofficial][S3].

### Data Structures
1.  **Quadtree (CPU):** Stores image regions as quadratic cells. Each leaf node represents a cell that can be refined into four children [S2, unofficial].
2.  **Error Map / Flag Array $F$ (CPU/GPU):** A 2D array (e.g., $R \times R$) where pixels indicate whether a ray has been traced ($F=1$) or if the region is computed via interpolation ($F=0$) [S3]. In FEM-based implementations, a 1D single-channel floating-point texture stores error values for quadtree leaves [S2, unofficial].
3.  **Interpolation Flag Array $G$ (CPU):** Tracks pixels that have been interpolated to avoid redundant computation [S3].
4.  **Color Buffer $C$:** Stores the final color values for each pixel [S3].

### Formulas
The volume rendering integral along a ray parametrized by arc length $w^x$ of length $L^x$ is [S2, unofficial]:
$$ I(x) = \int_0^{L^x} E(w^x) \exp\left(-\int_0^t A(w^x) ds\right) dt $$
where $E$ is emission and $A$ is absorption.

The approximation error is measured using the $L^2$ error norm [S2, unofficial]:
$$ \boldsymbol{\eta} = \|u - u_n\|_S := \left( \int_S (u - u_n)^2 dx \right)^{\frac{1}{2}} $$
where $u$ is the exact solution on the viewport $S$ and $u_n$ is the approximative solution.

The local error contribution on each cell $T$ is estimated by comparing two subsequent approximations [S2, unofficial]:
$$ \eta_{V_n}^T \approx \|u_{n-1} - u_n\|_T = \left( \int_T (u_{n-1} - u_n)^2 dx \right)^{\frac{1}{2}} $$
This integral is solved numerically using a Gaussian quadrature rule with $N=3$ points [S2, unofficial].

Relative $L^2$ error is defined as [S2, unofficial]:
$$ \eta_{rel} = \|u - u_n\|_S / \|u\|_S $$

### Parameters
*   **$m$:** Fixed number of cells to mark for refinement per iteration. Typical value: 1000 [S2, unofficial].
*   **$p$:** Polynomial degree for interpolation during reconstruction. Fixed value: 2 (bilinear/quadratic) [S2, unofficial].
*   **romax:** Maximum ray density/region size. Initial value: 16 [S3].
*   **romin:** Minimum ray density/region size. Initial value: 2 [S3].
*   **$\epsilon$ (e):** Error threshold for color difference. Initial value: 16–32 [S3].
*   **target_frame_rate:** Desired frame rate budget. Typical value: 10 fps [S1].
*   **error_threshold:** Maximum allowable visual error. Typical value: 1 pixel per texel [S1].
*   **W:** Number of evenly spaced locations along a ray for resampling [S3].

### Steps
The implementation varies slightly between the FEM-based approach [S2, unofficial] and the interactive level-of-detail approach [S3], but both follow a refinement loop.

#### FEM-Based Iterative Refinement [S2, unofficial]
1.  **Initialize:** Start with a coarse uniform grid (quadtree leaves).
2.  **Estimate Error:** For each new quadtree leaf, estimate the local error $\eta_T$ using the hierarchical a-posteriori estimator (Equation 9) [S2, unofficial].
3.  **Mark Cells:** Use a heap data structure to select the $m$ cells with the biggest impact on total error $\eta$. Sorting cost is $O(m \log n)$ [S2, unofficial].
4.  **Refine:** Subdivide marked cells into four new quadtree leaves. Generate new evaluation points: one at the cell center and four at the edge midpoints (max 5 points per leaf). Samples are never computed twice [S2, unofficial].
5.  **Ray Casting:** For each new evaluation point, perform standard ray-casting. Use **Step Doubling** for error control: compute color with full step size $h$ and two half steps $h/2$. If squared difference < error bound, accept and increase $h$; otherwise, decrease $h$ and repeat [S2, unofficial].
6.  **Reconstruct:** Render approximation $u_n$ to the screen using polynomial degree $p=2$ interpolation on a uniform grid [S2, unofficial].
7.  **Loop:** Repeat until convergence to pixel accuracy or user-specified error bound. Interrupt if camera or transfer function changes [S2, unofficial].

#### Interactive Level-of-Detail Refinement [S3]
1.  **Initialize:** Set `romax`, `romin`, and `epsilon`. Clear flags $F$ and $G$, and color buffer $C$ [S3].
2.  **Loop** while `romax` > 1:
    a.  **RayTraceData:** Divide image into square regions of size `romax`. For each region, call `RayTraceRegion` [S3].
        *   `RayTraceRegion(u,v,w)`: Cast rays from the four corner pixels of the region (if flag $F$ is clear). Set $F=1$ after tracing. If region size $w > w_{min}$ AND `Difference` (color range at corners > $\epsilon$) is true, divide into four subregions of size $w/2$ and recurse [S3].
        *   `RayTrace`: Resample data at $W$ evenly spaced locations, tri-linearly interpolate from 8 surrounding voxels, and composite with a fully opaque background [S3].
    b.  **InterpolateImage:** Divide image into square regions of size `romax`. For each region, call `InterpolateRegion` [S3].
        *   `InterpolateRegion(u,v,w)`: Interpolate colors at midpoints of sides (linear) and center (bilinear) of the region (if flags $F$ and $G$ are clear). Set $G=1$. If $w/2 > 1$, divide into four subregions and recurse [S3].
    c.  **DisplayImage:** Show the current image [S3].
    d.  **Clear:** For all pixels where $F$ is clear, reset $C$ and $G$ to 0 [S3].
    e.  **Increment:** Update `romax`, `romin`, and `epsilon` to next values in the sequence [S3].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# FEM-Based Implementation Structure [S2, unofficial]

def adaptive_refinement_fem(image_width, image_height, max_iterations, m):
    # Initialize quadtree with coarse grid
    quadtree = Quadtree(image_width, image_height)
    
    for iteration in range(max_iterations):
        # 1. Estimate error for new leaves
        error_values = []
        for leaf in quadtree.get_new_leaves():
            # Estimate local error using Gaussian quadrature on u_{n-1} - u_n
            local_error = estimate_local_error(leaf)
            error_values.append((leaf, local_error))
            
        # 2. Mark top m cells using a heap
        marked_cells = get_top_m_cells(error_values, m)
        
        # 3. Refine marked cells
        new_evaluation_points = []
        for cell in marked_cells:
            children = cell.refine() # Creates 4 new leaves
            # Generate evaluation points: center + 4 edge midpoints
            points = get_evaluation_points(cell)
            new_evaluation_points.extend(points)
            
        # 4. Compute ray casting for new points
        for point in new_evaluation_points:
            # Step Doubling for ray integration error control
            color_full = cast_ray_step(point, step_size=h)
            color_half = cast_ray_step_half(point, step_size=h/2.0)
            diff = squared_difference(color_full, color_half)
            
            if diff < error_bound:
                h = increase_step_size(h)
                color = color_full
            else:
                h = decrease_step_size(h)
                color = cast_ray_step(point, step_size=h) # Re-evaluate
            
            point.color = color
            
        # 5. Reconstruct and render
        render_image(quadtree, polynomial_degree=2)
        
        # Check convergence or interrupt on camera move
        if check_convergence() or camera_moved():
            break

# Interactive Level-of-Detail Implementation Structure [S3]

def adaptive_refinement_interactive(R, romax_init, romin_init, epsilon_init):
    # R is image side length
    flags_F = zeros(R, R) # Ray traced flag
    flags_G = zeros(R, R) # Interpolated flag
    colors = zeros(R, R, 3)
    
    romax = romax_init
    romin = romin_init
    epsilon = epsilon_init
    
    while romax > 1:
        # Ray Tracing Phase
        for v in range(0, R, romax):
            for u in range(0, R, romax):
                ray_trace_region(u, v, romax, romin, epsilon, flags_F)
                
        # Interpolation Phase
        for v in range(0, R, romax):
            for u in range(0, R, romax):
                interpolate_region(u, v, romax, flags_F, flags_G, colors)
                
        display(colors)
        
        # Clear interpolated colors for next iteration
        for v in range(R):
            for u in range(R):
                if flags_F[v][u] == 0:
                    colors[v][u] = 0
                    flags_G[v][u] = 0
                    
        # Increment LOD parameters
        romax = next_romax(romax)
        romin = next_romin(romin)
        epsilon = next_epsilon(epsilon)

def ray_trace_region(u, v, w, wmin, epsilon, flags_F):
    # Cast rays from four corners
    for j in [0, w]:
        for i in [0, w]:
            if u+i < R and v+j < R and flags_F[u+i][v+j] == 0:
                colors[u+i][v+j] = cast_ray(u+i, v+j)
                flags_F[u+i][v+j] = 1
                
    # Subdivision check
    if w > wmin and difference(u, v, w, epsilon) > 0:
        for j in [0, w/2]:
            for i in [0, w/2]:
                ray_trace_region(u+i, v+j, w/2, wmin, epsilon, flags_F)

def difference(u, v, w, epsilon):
    # Check if range of intensity values for any color component
    # at the four corners exceeds epsilon
    corners = get_corners(u, v, w)
    max_diff = 0
    for c in [R, G, B]:
        vals = [corner[c] for corner in corners]
        diff = max(vals) - min(vals)
        if diff > max_diff:
            max_diff = diff
    return max_diff

def interpolate_region(u, v, w, flags_F, flags_G, colors):
    # Interpolate at midpoints and center
    for j in [0, w/2, w]:
        for i in [0, w/2, w]:
            if u+i < R and v+j < R and flags_F[u+i][v+j] == 0 and flags_G[u+i][v+j] == 0:
                colors[u+i][v+j] = interpolate(u, v, w, u+i, v+j)
                flags_G[u+i][v+j] = 1
                
    # Recursion
    if w/2 > 1:
        for j in [0, w/2]:
            for i in [0, w/2]:
                interpolate_region(u+i, v+j, w/2, flags_F, flags_G, colors)
```

### Edge Cases and Pitfalls
*   **Latency:** Reactive feedback introduces scheduling latency; proactive scheduling using system timing parameters like fill rates is recommended [S1].
*   **Interactivity:** The loop is interrupted whenever the camera moves or the transfer function changes to guarantee interactivity [S2, unofficial].
*   **Parameter Sensitivity:** Manual selection of level-of-detail parameters (romax, romin, epsilon) can lead to sub-optimal presentation. Inappropriate values cause features to be missed, ignored, or poorly resolved [S3].
*   **Levoy's Estimator Limitation:** Unlike Levoy's variance-based estimator which refines linear regions with strong slopes unnecessarily, the hierarchical estimator notices good approximation in linear regions and focuses on interior edges [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Marking complexity is $O(m \log n)$ for sorting quadtree leaves using a heap, where $m$ is the number of marked cells and $n$ is the number of leaves [S2, unofficial]. Rendering time grows linearly with dataset size for full ray tracing, but adaptive refinement reduces effective cost by casting fewer rays in smooth regions [S3].
*   **Memory Complexity:** Quadtree data structures are stored on the CPU. Error estimation output is a 1D single-channel floating-point texture with $m$ entries [S2, unofficial].
*   **Measured Performance:**
    *   On a GeForce3 GPU (640x480 resolution), the system maintained a 10 fps budget, adding cells to reduce error below 1 pixel/texel [S1].
    *   On an Intel Core 2 Duo 3.0 GHz with NVIDIA GeForce 8800GT GPU, 7 iterations were needed for approximation error $10^{-2}$ on a 256x256 initial grid with $m=1000$. A single iteration took 20–40 ms for screen resolutions 512x512 and 2048x2048 respectively [S2, unofficial].
    *   Cytochrome dataset (113 slices of 256x256): Frame 1 took 4.6 seconds with 3,870 rays (20% coverage). Frame 4 took 25.7 seconds total with 19,289 rays (100% coverage) [S3].
    *   Head dataset (CT scan, 113 slices of 256x256): Frame 1 took 13 seconds with 5,179 rays (16% coverage). Frame 4 took 104 seconds total with 30,603 rays (100% coverage) [S3].
*   **Strengths:** Faster convergence (fewer rays for desired quality) compared to uniform discretization [S2, unofficial]. Allows interactive frame rates by focusing computational effort only where necessary [S3].
*   **Limitations:** Reactive feedback latency [S1]. Manual parameter tuning required for optimal results [S3].

## Background
*   **Origin:** The concept of adaptive image-space subdivision for volume rendering was introduced by Marc Levoy in 1990 [Lev90], who used a simple error measure based on corner pixel variance [S2, unofficial].
*   **Development:** The FEM-based approach was developed by Hadwiger et al. (2011), leveraging finite element theory for explicit error control and convergence analysis [S2, unofficial]. The interactive level-of-detail approach was presented by Upson et al. (1988), focusing on incremental refinement for molecular graphics and medical imaging [S3].
*   **Usage:** Demonstrated in interactive exploration of unstructured grids (Blunt Fin model) [S1], CT scans (Head, Engine) [S2, unofficial], and molecular graphics (Cytochrome B5) [S3]. Implemented in C++ and OpenGL/GLSL requiring Shader Model 4.0 GPU [S2, unofficial].

## Sources
[S1] Interactive Visualization of Unstructured Grids Using Hierarchical 3D ..., https://web.eecs.umich.edu/~jjcorso/pubs/leven02unstructgrid.pdf, fetched 2026-10-27
[S2] Adaptive Screen-Space Sampling for Volume Ray-Casting, https://webdoc.sub.gwdg.de/ebook/serien/ah/ZIB/ZR_11_04.pdf, fetched 2026-10-27 (unofficial)
[S3] Volume Rendering by Adaptive Refinement, https://www.cs.unc.edu/techreports/88-030.pdf, fetched 2026-10-29
