# Hybrid Ray-Casting and Shear-Warp with RLE Compression

Also called: Hybrid algorithm, New combination of ray-casting and shear-warp

## Overview
This method renders large Run-Length Encoded (RLE) compressed volumes interactively by combining image-order shear-warp techniques with object-order ray-casting [S1]. It solves the problem of 3-fold data redundancy found in original shear-warp implementations, which required storing three separate classified RLE volumes for the three principal viewing directions [S1]. By removing this redundancy, the method reduces memory usage to approximately $O(N/3)$ while maintaining high image quality through true trilinear interpolation and Hermite polynomial extension [S1]. It sets itself apart from standard shear-warp by introducing new data structures for bijective mapping between voxel space and image space, enabling efficient Early Ray Termination (ERT) and space leaping without parallel volume and Intermediate Image (IIM) scanlines [S1].

## How it works
The algorithm decomposes the rendering process into data structure construction, view-dependent processing, and compositing with interpolation.

### Inputs and Outputs
*   **Input:** A single RLE-compressed volume dataset. The volume is conceptually permuted so that the principal viewing axis aligns with the Z-axis of the intermediate coordinate system, consistent with the parent Shear-warp principles .
*   **Output:** A rendered 2D image (IIM).

### Data Structures
Two new data structures are constructed to describe the bijective mapping between object space (voxels) and image space (IIM pixels) [S1].

1.  **Column Template:** Represents all information within a column of the volume, where a $2 \times 2$ arrangement of RLEs is denoted as a column. This structure stores:
    *   Sample point positions.
    *   Weights for trilinear interpolation.
    *   Ray segments.
    *   Number of sample points on ray segments.
    *   Affected IIM pixels [S1].
2.  **ERT Template:** Represents a projected scanline of the volume onto the IIM. It is used to implement efficient Early Ray Termination (ERT) by skipping opaque IIM pixels along oblique, discrete lines using Bresenham's algorithm [S1].

### Parameters
*   **Interpolation:** True trilinear interpolation is used for sampling [S1].
*   **Curve Extension:** Hermite polynomials are used to extend the discrete data set to continuous space for viewing along the two non-primary axes [S1].
*   **Typical Performance Impact:**
    *   Rendering speed is reduced by a factor of 2 to 4 compared to the original 3-RLE shear-warp algorithm [S1].
    *   Without the higher sampling rate and true trilinear interpolation, 85% of the original implementation performance would be achieved [S1].
    *   Using Hermite curves to suppress artifacts at 45-degree transitions results in a 30-40% reduced frame rate [S1].

### Steps

1.  **Preprocessing: Pointer Array Construction**
    Construct an array of pointers to every RLE scanline in the volume. This removes one layer of RLE data set redundancy, allowing rendering from a single main viewing direction without maintaining parallel volume and IIM scanlines [S1].

2.  **View Direction Handling**
    The processing logic depends on the rotation of the volume relative to the viewer:

    *   **Case A: Viewing along the 3rd viewing axis (X-axis in the permuted frame)**
        *   Process volume RLE scanlines along the projection direction.
        *   Since there is only one ray segment per column, consider the IIM pixel only once. No RLE encoding of the IIM is required [S1].

    *   **Case B: Rotation about only one axis (Y or Z)**
        *   IIM scanlines pass horizontally or vertically.
        *   Skip opaque ray segments using the dynamic RLE of the IIM instead of skipping opaque pixels [S1].

    *   **Case C: General case (Rotation about Y and Z simultaneously)**
        *   IIM scanlines pass obliquely. Encode IIM scanlines to handle oblique projection [S1].
        *   After compositing sample points and reaching a transparent voxel run, use space leaping [S1].

3.  **Ray-Casting and Compositing (General Case)**
    For the general case, the algorithm performs ray-casting within 4 RLE scanlines to preserve image quality and satisfy the Shannon-Nyquist theorem [S1].

    1.  **Space Leaping:** Skip transparent regions using the volume RLE data.
    2.  **Locate Start Point:** Use the **Column Template** to find the current sample point position and the affected IIM pixels after space leaping [S1].
    3.  **Early Ray Termination (ERT):** Use the **ERT Template** to skip $m$ opaque IIM pixels along oblique, discrete lines (Bresenham's algorithm). This identifies the corresponding voxel cuboid where compositing should start [S1].
    4.  **Sampling:** Sample the volume using true trilinear interpolation.
    5.  **Interpolation Extension (Non-Primary Axes):** If viewing along the two other main directions (not the primary Z-aligned axis), use Hermite polynomials to interpolate between sample points in adjacent volume slices to suppress artifacts at 45-degree transitions [S1].
    6.  **Compositing:** Composite the sampled values into the IIM using front-to-back accumulation .

### Formulas

**Hermite Polynomial Interpolation**
To suppress artifacts when viewing along the two non-primary axes, a Hermite polynomial $Q(t)$ is used to extend the discrete data set to continuous space [S1].

$$ Q(t) = (2t^3 - 3t^2 + 1)\vec{p}_1 + (-2t^3 + 3t^2)\vec{p}_2 + (t^3 - 2t^2 + t)\vec{n}_1 + (t^3 - t^2)\vec{n}_2 $$

Where:
*   $t \in [0, 1]$ specifies an intermediate point.
*   $\vec{p}_1, \vec{p}_2$ are sample points.
*   $\vec{n}_1, \vec{n}_2$ are tangent vectors at these points.
*   The tangent slope is given by $dQ(t)/dt$ [S1].

**Memory Complexity**
The memory usage is approximately $O(N/3)$ due to the removal of the 3-fold redundancy of RLE data sets [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_hybrid_shear_warp(volume_rle, view_matrix, image_width, image_height):
    """
    Renders a volume using the Hybrid Ray-Casting and Shear-Warp method.
    
    Args:
        volume_rle: The single RLE-compressed volume dataset.
        view_matrix: The 4x4 view matrix M_view.
        image_width: Width of the output image.
        image_height: Height of the output image.
        
    Returns:
        image: The rendered 2D image buffer.
    """
    
    # 1. Preprocessing: Construct pointer array to RLE scanlines
    # This removes one level of redundancy, allowing single-direction rendering
    scanline_pointers = build_scanline_pointer_array(volume_rle)
    
    # 2. Construct Data Structures
    # Column Template: Maps volume columns (2x2 RLE arrangement) to IIM pixels
    column_template = build_column_template(volume_rle, scanline_pointers)
    
    # ERT Template: Maps projected scanlines to IIM pixels for skipping opaque segments
    ert_template = build_ert_template(volume_rle, scanline_pointers)
    
    # Initialize Image Buffer
    image = zeros((image_height, image_width, 4)) # RGBA
    
    # 3. Determine View Axis and Processing Mode
    # Decompose view matrix to find principal axis (aligned with Z in intermediate space)
    principal_axis = get_principal_axis(view_matrix)
    
    if principal_axis == 'X':
        # Case A: View along 3rd axis (X)
        render_mode_x_axis(volume_rle, scanline_pointers, column_template, image)
    elif principal_axis in ['Y', 'Z']:
        # Case B: Rotation about one axis
        render_mode_one_axis(volume_rle, scanline_pointers, ert_template, image)
    else:
        # Case C: General case (Rotation about Y and Z)
        render_mode_general(volume_rle, scanline_pointers, column_template, ert_template, image)
        
    return image

def render_mode_general(volume_rle, scanline_pointers, column_template, ert_template, image):
    """
    Handles the general case with oblique IIM scanlines.
    """
    # Iterate through IIM pixels or scanlines
    for y in range(image_height):
        for x in range(image_width):
            
            # 1. Space Leaping
            # Skip transparent voxel runs using volume RLE
            start_voxel, end_voxel = perform_space_leap(volume_rle, scanline_pointers, x, y)
            
            if start_voxel is None:
                continue
                
            # 2. Locate Start Point using Column Template
            # Find current sample point and affected IIM pixels
            sample_info = column_template.get_sample_info(start_voxel)
            
            # 3. Early Ray Termination (ERT) using ERT Template
            # Skip m opaque IIM pixels along oblique discrete lines (Bresenham)
            # to find the corresponding voxel cuboid to start compositing
            m = ert_template.count_opaque_pixels(x, y)
            if m > 0:
                # Skip ahead in compositing or adjust ray start
                adjust_ray_start_for_ert(sample_info, m)
                
            # 4. Ray-Casting within 4 RLE scanlines
            # Perform ray-casting to preserve Shannon-Nyquist quality
            accumulated_color = (0.0, 0.0, 0.0, 0.0)
            accumulated_alpha = 0.0
            
            current_voxel = start_voxel
            while current_voxel is not None and accumulated_alpha < 1.0:
                
                # Sample volume using true trilinear interpolation
                # Note: For non-primary axes, use Hermite interpolation
                if is_non_primary_view():
                    value = sample_hermite(volume_rle, current_voxel, scanline_pointers)
                else:
                    value = sample_trilinear(volume_rle, current_voxel, scanline_pointers)
                    
                # Composite using Porter-Duff "over" operator
                color, alpha = value
                accumulated_color = composite_over(accumulated_color, (color, alpha))
                accumulated_alpha = accumulated_color[3]
                
                # Move to next voxel
                current_voxel = get_next_voxel(current_voxel)
                
            # Write to image
            image[y, x] = accumulated_color

def sample_hermite(volume_rle, voxel, scanline_pointers):
    """
    Samples the volume using Hermite polynomials to suppress artifacts at 45-degree transitions.
    """
    # Get sample points p1, p2 and tangent vectors n1, n2
    p1, n1 = get_sample_and_tangent(volume_rle, voxel, scanline_pointers, index=0)
    p2, n2 = get_sample_and_tangent(volume_rle, voxel, scanline_pointers, index=1)
    
    # Parameter t in [0, 1]
    t = 0.5 # Example intermediate point
    
    # Hermite polynomial formula
    # Q(t) = (2t^3 - 3t^2 + 1)p1 + (-2t^3 + 3t^2)p2 + (t^3 - 2t^2 + t)n1 + (t^3 - t^2)n2
    h1 = 2*t**3 - 3*t**2 + 1
    h2 = -2*t**3 + 3*t**2
    h3 = t**3 - 2*t**2 + t
    h4 = t**3 - t**2
    
    value = h1 * p1 + h2 * p2 + h3 * n1 + h4 * n2
    
    return value

def composite_over(accumulated, new):
    """
    Porter-Duff "over" operator.
    C_new = C_slice * alpha_slice + C_accum * (1 - alpha_slice)
    alpha_new = alpha_slice + alpha_accum * (1 - alpha_slice)
    """
    c_slice, a_slice = new
    c_accum, a_accum = accumulated
    
    new_alpha = a_slice + a_accum * (1 - a_slice)
    new_color = c_slice * a_slice + c_accum * (1 - a_slice)
    
    return new_color, new_alpha
```

### Edge Cases and Pitfalls
*   **45-Degree Transitions:** Viewing along the two non-primary axes can introduce artifacts. The method mitigates this using Hermite polynomials, but this incurs a 30-40% frame rate reduction [S1].
*   **Oblique Projections:** In the general case, IIM scanlines are oblique. The algorithm must encode IIM scanlines specifically to handle this, requiring Bresenham-based skipping for ERT [S1].
*   **Performance Overhead:** The removal of redundancy and increase in sampling rate ($\sqrt{3}$ along the major diagonal) and true trilinear interpolation result in a 2x to 4x speed reduction compared to the original 3-RLE shear-warp algorithm [S1].

## Operating characteristics
*   **Time Complexity:** The rendering speed is reduced by a factor of 2 to 4 compared to the original shear-warp algorithm [S1]. The overhead comes from the higher sampling rate (up to $\sqrt{3}$ if viewing along the major diagonal) and true trilinear interpolation (factor of 2) [S1].
*   **Memory Complexity:** Approximately $O(N/3)$ due to the removal of the 3-fold redundancy of RLE data sets [S1].
*   **Strengths:**
    *   Eliminates 3-fold data redundancy, saving significant memory.
    *   Preserves benefits of Lacroute's original implementation: RLE data, aligned scanlines, single object-order traversal, and efficient space leaping/ERT [S1].
    *   High image quality preserving the Shannon-Nyquist theorem [S1].
    *   No visible artifacts due to quantization after classification [S1].
*   **Limitations:**
    *   Significantly slower than the original 3-RLE shear-warp algorithm (2x-4x slower) [S1].
    *   Hermite interpolation for non-primary axes adds further computational cost (30-40% frame rate reduction) [S1].
*   **When to Use:** When memory is constrained and interactive rendering of large RLE-compressed volumes is required on a normal PC, and high image quality (suppression of quantization artifacts) is prioritized over maximum frame rate [S1].
*   **When Not to Use:** When maximum rendering speed is critical and memory is not a constraint, the original 3-RLE shear-warp algorithm may be preferable [S1].

## Background
The method was introduced by S. Gregor et al. in the paper "Volume rendering on one RLE compressed data set by a new combination of ray-casting and shear-warp" [S1]. It addresses limitations in the original shear-warp implementation by Lacroute, specifically the need for three classified volumes (RLE data sets) for the three principal viewing directions [S1]. The work builds on the Shear-warp algorithm [24-shear-warp.md] and incorporates techniques from Sweeney and Mueller regarding intermediate slices, replacing them with Hermite polynomials for better artifact suppression [S1].

## Sources
[S1] Volume rendering on one RLE compressed data set by a new combination of ray-casting and shear-warp, https://doi.org/10.1145/1186415.1186452, fetched 2026-10-31
