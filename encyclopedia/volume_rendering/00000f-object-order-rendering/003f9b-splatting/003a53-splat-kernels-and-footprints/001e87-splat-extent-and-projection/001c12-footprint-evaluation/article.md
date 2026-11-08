# Footprint Evaluation

Also called: Footprint Mapping, Splatting (Westover), Westover 1990, Westover Splatting, Westover's Footprint Evaluation, Westover's Splatting

## Overview
Footprint Evaluation is a forward-mapping (object-order) volume rendering algorithm that projects each voxel in object space onto the 2D image plane, where its contribution is distributed as a "splat" or footprint [S1, unofficial][S3, unofficial][S9, unofficial]. It solves the problem of efficiently rendering volumetric data by convolving every voxel with a reconstruction kernel and accumulating these contributions on the image plane, contrasting with image-order methods like ray casting that sample along rays [S3, unofficial][S9, unofficial]. It sets itself apart from its siblings by pre-integrating the 3D kernel into a 2D footprint table to avoid on-the-fly integration, enabling fast per-voxel projection and compositing [S10, unofficial][S11, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume grid of voxels $V(i,j,k)$ with scalar values $\rho(s)$, a viewing transform (orthographic or perspective), and a 3D spherically symmetric reconstruction kernel $w(x,y,z)$ [S2, unofficial][S6, unofficial].
*   **Output:** A 2D image buffer where each pixel contains the accumulated color and opacity from all projected voxels [S3, unofficial][S4, unofficial].

**Data Structures:**
1.  **Volume Grid:** A 3D array of voxels. Each voxel $s$ has a position $(x_s, y_s, z_s)$ in object space and a value $\rho(s)$ [S9, unofficial].
2.  **Footprint Table:** A pre-computed 2D lookup table representing the integration of the 3D kernel along the ray axis. For orthographic projection with uniform grid spacing and symmetric kernels, a single generic table is used for all voxels [S4, unofficial][S9, unofficial]. For perspective projection, view-dependent elliptical footprints may require separate tables or transformations [S6, unofficial][S9, unofficial].
3.  **Buffers:**
    *   **Compositing Buffer:** The final image accumulator [S6, unofficial].
    *   **Sheet Buffer (Optional):** An intermediate buffer used in the sheet-buffer method to cache splats from a slice of the volume before compositing into the main buffer [S1, unofficial][S6, unofficial].

**Formulas:**
The reconstructed continuous volume function $f_r(v)$ is the sum of sample densities weighted by the kernel:
$$ f_r(v) = \sum_k w(v - v_k) f(v_k) $$
where $f(v_k)$ is the density at sample $k$ and $w$ is the reconstruction kernel [S2, unofficial][S6, unofficial].

The intensity $I(p)$ for a pixel $p$ along ray $r$ is the integral of the reconstructed function:
$$ I(p) = \int f_r(p+r) dr = \int \sum_k w(p+r-v_k) f(v_k) dr $$
By switching summation and integral, the intensity becomes the sum of voxel densities weighted by the splatting kernel (footprint):
$$ I(p) = \sum_k f(v_k) \cdot \int w(p+r-v_k) dr $$
The term $\int w(p+r-v_k) dr$ is the **footprint function** $F(x,y)$, defined as the integral of the 3D kernel along the ray axis (z-axis):
$$ F(x,y) = \int w(x,y,z) dz $$
For a Gaussian kernel $w(x,y,z) = d \exp(-x^2/a^2 - y^2/b^2 - z^2/c^2)$, the footprint is also a Gaussian [S2, unofficial][S6, unofficial].

The contribution of a sample $s$ to an image plane pixel $p(x,y)$ is:
$$ I_s(p) = \rho(s) F(x-x_s, y-y_s) $$
where $(x_s, y_s)$ is the projected center of the voxel in screen space [S9, unofficial].

**Numbered Steps:**
1.  **Preprocessing:**
    *   Select a 3D spherically symmetric kernel $w(x,y,z)$, such as a Gaussian [S6, unofficial].
    *   Compute the 2D footprint function $F(x,y) = \int w(x,y,z) dz$ [S2, unofficial][S6, unofficial].
    *   Store $F(x,y)$ in a finely sampled 2D lookup table (Footprint Table) [S4, unofficial][S10, unofficial].
    *   If using perspective projection, determine the transformation to map the generic spherical footprint to the view-dependent elliptical footprint [S6, unofficial][S9, unofficial].

2.  **Volume Slicing (Sheet Selection):**
    *   Determine the stack of volume slices most parallel to the image plane based on the viewing angle [S2, unofficial][S6, unofficial].
    *   If the viewing angle is ~30 degrees, use the stack aligned with the x-axis; if ~70 degrees, use the stack aligned with the y-axis [S2, unofficial][S6, unofficial].

3.  **Projection and Splatting (Per Voxel):**
    *   Visit voxels in the selected stack in either back-to-front or front-to-back order [S3, unofficial][S4, unofficial].
    *   For each voxel $s$:
        a.  Transform the voxel center $(x_s, y_s, z_s)$ from object space to screen space $(x_s', y_s')$ [S9, unofficial].
        b.  Compute the screen-space extent of the footprint (the region of pixels affected) [S9, unofficial].
        c.  For each pixel $(x,y)$ within the extent:
            i.  Calculate the displacement $(dx, dy)$ from the voxel's projected center.
            ii. Look up the weight $w_{footprint} = F(dx, dy)$ from the footprint table [S10, unofficial].
            iii. If using perspective projection, apply the view-dependent transformation to the footprint coordinates or weight [S6, unofficial][S9, unofficial].
            iv. Accumulate the voxel's contribution to the pixel:
                $$ I_{pixel} \leftarrow I_{pixel} + \rho(s) \cdot w_{footprint} $$
                (Note: Standard splatting often uses simple addition or front-to-back compositing with opacity [S3, unofficial][S6, unofficial]).

4.  **Compositing (Sheet-Buffer Variant):**
    *   If using the sheet-buffer method [S1, unofficial][S7, unofficial]:
        a.  Add voxel kernels within the current sheet to the sheet buffer.
        b.  Transfer the sheet buffer to the compositing buffer.
        c.  Composite the sheet buffer with the compositing buffer using a compositing operator (e.g., front-to-back) [S6, unofficial].
        d.  Repeat for subsequent sheets.

5.  **Final Output:**
    *   The compositing buffer contains the final image [S9, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Preprocessing
def compute_footprint_table(kernel_func, table_size, resolution):
    """
    Computes the 2D footprint table by integrating the 3D kernel along Z.
    kernel_func: function w(x, y, z)
    table_size: int, dimension of the square table
    resolution: float, pixel size in object space units
    """
    table = [[0.0 for _ in range(table_size)] for _ in range(table_size)]
    center = table_size // 2
    for i in range(table_size):
        for j in range(table_size):
            x = (i - center) * resolution
            y = (j - center) * resolution
            # Integrate kernel along z from -infinity to infinity
            # Approximated by numerical integration or analytic solution for Gaussian
            footprint_val = integrate_kernel_along_z(kernel_func, x, y)
            table[i][j] = footprint_val
    return table

def integrate_kernel_along_z(kernel_func, x, y):
    # Placeholder for actual integration logic
    # For Gaussian w(x,y,z) = exp(-x^2/a^2 - y^2/b^2 - z^2/c^2)
    # Integral over z is sqrt(pi)*c * exp(-x^2/a^2 - y^2/b^2)
    return 0.0 # Not given in sources

# Rendering
def footprint_evaluation(volume, view_transform, footprint_table, kernel_value_func):
    """
    volume: 3D array of voxel values
    view_transform: function to map (x,y,z) -> (sx, sy)
    footprint_table: 2D array of precomputed footprint weights
    kernel_value_func: function to get voxel value rho(s)
    """
    image_width, image_height = get_image_dimensions()
    image = [[0.0 for _ in range(image_height)] for _ in range(image_width)]
    
    # Determine slice order based on viewing angle
    slices = get_slices_parallel_to_view(volume, view_transform)
    
    for slice in slices:
        # Optional: Use sheet buffer for this slice
        sheet_buffer = [[0.0 for _ in range(image_height)] for _ in range(image_width)]
        
        for voxel in slice:
            # 1. Project voxel center
            sx, sy = view_transform(voxel.x, voxel.y, voxel.z)
            
            # 2. Determine footprint extent
            # Assume footprint has radius R in screen pixels
            extent = get_footprint_extent(sx, sy, image_width, image_height)
            
            # 3. Splat onto sheet buffer
            for dx, dy in extent:
                px = int(sx + dx)
                py = int(sy + dy)
                
                if 0 <= px < image_width and 0 <= py < image_height:
                    # Look up footprint weight
                    # Map screen offset to table index
                    table_idx_x = int(dx) + len(footprint_table) // 2
                    table_idx_y = int(dy) + len(footprint_table) // 2
                    
                    weight = footprint_table[table_idx_x][table_idx_y]
                    
                    # Get voxel value
                    rho = kernel_value_func(voxel)
                    
                    # Accumulate
                    # Note: Simple addition. For correct opacity compositing, 
                    # front-to-back compositing with alpha is needed [S6, unofficial]
                    sheet_buffer[px][py] += rho * weight
        
        # Composite sheet buffer to image
        for px in range(image_width):
            for py in range(image_height):
                image[px][py] += sheet_buffer[px][py]
                
    return image
```

**Parameters:**
*   **Kernel Shape/Size:** Critical for quality. Larger footprints increase blurring (used for high pixel-to-voxel ratios); smaller sizes generate gaps [S2, unofficial][S6, unofficial]. A 3D Gaussian is a common choice [S6, unofficial].
*   **Footprint Table Size:** Larger tables allow nearest neighbor interpolation; smaller tables require interpolation from nearest samples [S11, unofficial].
*   **Traversal Order:** Back-to-front guarantees correct visibility; front-to-back is faster as it can stop when pixels are fully opaque [S3, unofficial][S6, unofficial].

**Edge Cases and Pitfalls:**
*   **Overlapping Splats:** When splats overlap, the algorithm performs reconstruction and integration on a per-splat basis rather than globally, resulting in incorrect compositing and aliasing artifacts [S2, unofficial][S4, unofficial][S6, unofficial].
*   **High Resolution/Aliasing:** Standard splatting fails to correctly render cases where volume sampling rate is higher than image sampling rate (e.g., orthographic projections of high-resolution volumes), causing severe spatial and temporal aliasing [S4, unofficial].
*   **View-Dependent Footprints:** In perspective projection, footprints become elliptical. If not handled correctly (e.g., by transforming the generic table), artifacts occur [S6, unofficial][S9, unofficial].
*   **Sheet Buffer Popping:** In the sheet-buffer method, sudden changes in slice orientation can cause popping artifacts [S1, unofficial][S7, unofficial].
*   **Non-linear Classification:** If classification or shading is non-linear, performing it prior to projection (as is efficient in splatting) produces incorrect results [S4, unofficial].

## Subtopics

### [Antialiased Splatting with Footprint Alignment](1288-antialiased-splatting-with-footprint-alignment.md)
Antialiased Splatting with Footprint Alignment is a forward-mapping volume rendering technique that prevents sample-rate aliasing in perspective views by dynamically scaling splat sizes and attenuating splat energy for voxels beyond a critical distance. It sets itself apart from standard splatting by varying the kernel size and energy as a function of depth to match the diverging integration grid rays, ensuring the integration grid sampling frequency remains above twice the maximum volume raster frequency.

### [Footprint Function Mechanism](4731-footprint-function-mechanism.md)
The Footprint Function Mechanism is a method for implementing 3D spherically symmetric splatting kernels in volume rendering by reducing the computational dimensionality from 3D to 2D. It solves the problem of efficiently projecting volumetric data onto a 2D image plane by pre-computing the integration of the 3D kernel along one axis, storing the result as a 2D lookup table. This approach sets itself apart from naive 3D splatting by avoiding per-pixel 3D kernel evaluations, instead relying on 2D table lookups and interpolation, as detailed in [Footprint Evaluation](1180-footprint-evaluation.md).

### Separable Footprint (SF) Projector
The Separable Footprint (SF) Projector is a projector model that accounts for the finite width of detector pixels and volume voxels, providing higher accuracy than methods that do not model finite widths (like Siddon or Joseph), though at higher computational cost [S13, unofficial]. It models finite detector/voxel widths, whereas Siddon and Joseph do not. SF is more accurate and avoids artifacts produced by Siddon/Joseph in some cases, but is more computationally expensive [S13, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   Splatting has a complexity of $O(k^2 n^3)$, where $k$ is the kernel size and $n^3$ is the voxel count, because every voxel is splatted exactly once [S3, unofficial][S4, unofficial].
*   This is faster than ray casting, which has a complexity of at least $O(k^3 n^3)$ because reconstruction is performed for each sample point along the ray [S3, unofficial][S4, unofficial].
*   Memory usage depends on the footprint table size and the number of buffers (image, sheet) [S6, unofficial].

**Measured Performance:**
*   Splatting is inherently faster than ray casting for large datasets due to pre-computed convolution [S3, unofficial][S4, unofficial].
*   However, achieving high image quality with splatting requires as much CPU computing time as ray casting, due to the need for large kernels and handling overlaps [S2, unofficial][S6, unofficial].
*   Footprint mapping is more costly than trilinear interpolation at moderate screen sizes but can be faster at large screen sizes for specific datasets due to screen-size-independent setup costs [S12].

**Strengths and Limitations:**
*   **Strengths:** Fast object-order rendering; simple static parallel decomposition; efficient for sparse data (air voxels can be disregarded); pre-integrated kernels reduce per-voxel cost [S3, unofficial][S6, unofficial].
*   **Limitations:** Poor quality in original form (blurry images for zoomed views due to Gaussian cutting high frequencies); aliasing artifacts when splats overlap or sampling rate is high; incorrect compositing for non-linear transfer functions; high fill-rate for zoomed splats [S2, unofficial][S4, unofficial][S6, unofficial].

**When to Use:**
*   Use when speed is critical and high-quality anti-aliasing is not the primary concern, or when using variants like Antialiased Splatting.
*   Suitable for real-time visualization of large volumetric datasets on CPU or GPU [S3, unofficial][S6, unofficial].

**When Not to Use:**
*   Avoid when high-frequency details must be preserved without blurring.
*   Avoid when strict physical accuracy (correct compositing of overlapping volumes) is required without additional corrections.
*   Avoid for non-linear classification/shading if performed pre-projection.

## Background
Footprint Evaluation, commonly known as Splatting, was introduced by L. Westover in 1990 in the paper "Footprint evaluation for volume rendering" [S9, unofficial][S11, unofficial]. It was first described for rendering regular grids on a CPU [S1, unofficial]. The method was developed to provide an efficient object-order alternative to image-order ray casting, leveraging the linearity of integration to pre-compute kernel contributions [S3, unofficial][S4, unofficial]. It has since been extended to handle perspective projection, scattered data (e.g., SPH), and improved quality through techniques like sheet buffering and anti-aliasing [S1, unofficial][S6, unofficial].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] Direct Volume Rendering Techniques, https://www.scribd.com/document/853597780/VIS-Modules-06-Direct-Volume-Rendering, fetched 2026-10-25 (unofficial)
[S3] DTIC ADA363073: Perspective Splatting, https://archive.org/details/DTIC_ADA363073, fetched 2026-10-26 (unofficial)
[S4] DTIC ADA606342: An Anti-Aliasing Technique for Splatting, https://archive.org/details/DTIC_ADA606342, fetched 2026-10-26 (unofficial)
[S5] Volume rendering techniques for general purpose graphics hardware, https://opus4.kobv.de/opus4-fau/files/9/Dissertation.pdf, fetched 2026-10-26 (unofficial)
[S6] [PDF] Direct Volume Rendering | CGL @ ETHZ, https://cgl.ethz.ch/teaching/former/scivis_07/Notes/stuff/StuttgartCourse/VIS-Modules-06-Direct_Volume_Rendering.pdf, fetched 2026-10-26 (unofficial)
[S7] Hardware-Accelerated Volume Rendering, https://vcg.seas.harvard.edu/publications/20040101-hardware-accelerated-volume-rendering/paper, fetched 2026-10-27 (unofficial)
[S8] (PDF) EWA volume splatting - ResearchGate, https://www.researchgate.net/publication/48546966_EWA_volume_splatting, fetched 2026-10-28 (unofficial)
[S9] PDF Volume Rendering - LSU, http://www.csc.lsu.edu/~karki/courses/SV/12-Splatting.pdf, fetched 2026-10-28 (unofficial)
[S10] Footprint Evaluation for Volume Rendering - SlideServe, https://www.slideserve.com/christopherivy/footprint-evaluation-for-volume-rendering-powerpoint-ppt-presentation, fetched 2026-10-28 (unofficial)
[S11] Footprint Evaluation for Volume Rendering A Feedforward Approach, https://slidetodoc.com/footprint-evaluation-for-volume-rendering-a-feedforward-approach/, fetched 2026-10-28 (unofficial)
[S12] A Practical Evaluation of Popular Volume Rendering ..., https://courses.cs.duke.edu/spring03/cps296.8/papers/PracticalEvaluationofVolumeRenderingAlgorithms.pdf, fetched 2026-10-29
[S13] PDF Differentiable Forward Projector for X-ray Computed Tomography, https://differentiable.xyz/papers-2023/paper_48.pdf, fetched 2026-10-29 (unofficial)
