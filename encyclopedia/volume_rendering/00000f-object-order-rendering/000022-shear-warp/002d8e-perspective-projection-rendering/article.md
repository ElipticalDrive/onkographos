# Perspective Projection Rendering

Also called: Perspective Projection Rendering 2, Perspective Shear-Warp

## Overview
Perspective Projection Rendering is a volume rendering method that extends the shear-warp factorization to support perspective viewing transformations [S2]. It solves the problem of rendering volumetric data with correct perspective foreshortening, which parallel projection methods cannot handle natively [S2]. It sets itself apart from its siblings by scaling each volume slice during the resampling phase to account for distance from the camera, whereas [Shear-Warp Algorithm](4304-shear-warp-algorithm.md) and [Affine-Transformations Method](2038-affine-transformations-method.md) typically assume parallel projection or handle perspective differently [S2].

## How it works
The method builds on the [Shear-warp](24-shear-warp.md) factorization, decomposing the viewing transformation matrix $M_{view}$ into a permutation $P$, a perspective shear $S_{\text{persp}}$, and a 2D warp $M_{\text{warp}}$ [S2]:
$$ M_{view} = M_{\text{warp}} \cdot S_{\text{persp}} \cdot P $$
The permutation $P$ aligns the principal viewing axis with a coordinate axis (usually Z) [S2]. The perspective shear $S_{\text{persp}}$ transforms the volume such that viewing rays become parallel to the Z-axis, but unlike parallel shear, it includes a scaling component to handle perspective [S2]. The warp $M_{\text{warp}}$ corrects the distortion to produce the final image [S2].

**Inputs and Outputs:**
*   **Input:** Volume data $V$, viewing transformation matrix $M_{view}$, transfer function (if not pre-classified), RLE structures for the volume.
*   **Output:** Final 2D image.

**Data Structures:**
*   **RLE Volume:** Each copy contains four bytes per non-transparent voxel and one byte per run [S2]. Three copies (one for each principal axis) are required to avoid performance degradation when the major viewing axis changes [S2].
*   **Intermediate Image:** Buffer for compositing sheared slices.

**Steps:**
1.  **Factorization:** Compute $S_{\text{persp}}$ and $M_{\text{warp}}$ from $M_{view}$ [S2].
    $$ S_{\text{persp}} = \begin{pmatrix} 1 & 0 & 0 & 0 \\ 0 & 1 & 1 & 0 \\ s'_x & s'_y & 1 & s'_w \\ 0 & 0 & 0 & 1 \end{pmatrix} $$
    The elements $s'_x, s'_y, s'_w$ are derived from $M_{view}$ [S2].
2.  **Shear and Resampling:** For each slice $z_0$ of the volume data:
    *   Translate the slice by $(z_0 s'_x, z_0 s'_y)$ [S2].
    *   Scale the slice uniformly by $1/(1 + z_0 s'_w)$ [S2].
    *   Resample the transformed slice into the intermediate image space using a box reconstruction filter and a box low-pass filter [S1, S1, unofficial].
    *   **Note:** Because of the scaling, more than two voxel scanlines may need to be traversed simultaneously to contribute to an intermediate image scanline, and voxel scanlines may not be traversed at the same rate as image scanlines [S1, unofficial].
    *   **Parameter:** Choose scaling factors such that the closest slice has unit scaling; all other slices will have scaling $< 1$, ensuring no slice is enlarged [S1, unofficial].
3.  **Compositing:** Composite the resampled slices in front-to-back order into the intermediate image using the Porter-Duff "over" operator [S2]:
    $$ C_{new} = C_{slice} \cdot \alpha_{slice} + C_{accum} \cdot (1 - \alpha_{slice}) $$
    $$ \alpha_{new} = \alpha_{slice} + \alpha_{accum} \cdot (1 - \alpha_{slice}) $$
    Early ray termination is supported by skipping further slices for pixels where accumulated opacity exceeds a threshold (e.g., 0.95) [S2].
4.  **Warp:** Transform the intermediate image to the final image space using $M_{\text{warp}}$ [S2]. This step corrects the distortion introduced by the shear and applies the correct perspective projection [S2]. Bilinear interpolation is typically used [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def perspective_shear_warp_render(volume, M_view, rle_structures):
    # 1. Factorization
    P, S_persp, M_warp = factorize_view_matrix(M_view)
    s_prime_x, s_prime_y, s_prime_w = get_shear_params(S_persp)
    
    # Determine slicing direction from P
    slice_axis = get_slice_axis(P)
    
    # 2. Shear and Resampling
    intermediate_image = create_empty_image()
    
    # Iterate over slices along the principal axis
    for z0 in range(volume.depth):
        # Get slice data
        slice_data = get_slice(volume, z0, slice_axis)
        
        # Calculate scaling factor
        # Ensure closest slice (z0=0 or min z) has unit scaling
        # The formula 1/(1 + z0 * s'_w) handles the perspective foreshortening
        scale_factor = 1.0 / (1.0 + z0 * s_prime_w)
        
        # Translate and Scale
        # Translation vector depends on s'_x, s'_y and z0
        tx = z0 * s_prime_x
        ty = z0 * s_prime_y
        
        # Resample slice into intermediate image
        # Use box reconstruction and low-pass filters [S1, unofficial]
        # Note: More than 2 voxel scanlines may contribute to one image scanline [S1, unofficial]
        resampled_slice = resample_slice(
            slice_data, 
            translation=(tx, ty), 
            scale=scale_factor, 
            filter_type="box"
        )
        
        # Composite into intermediate image
        composite_front_to_back(intermediate_image, resampled_slice)
        
        # Early ray termination check [S2]
        if early_ray_termination(intermediate_image, threshold=0.95):
            break

    # 3. Warp
    final_image = warp_image(intermediate_image, M_warp)
    
    return final_image
```

## Subtopics

### Perspective Projection Algorithm
A specific implementation of the perspective shear-warp method that utilizes run-length encoded volume data [S2]. It transforms each slice $z_0$ by translating it by $(z_0 s'_x, z_0 s'_y)$ and scaling it by $1/(1 + z_0 s'_w)$ [S2]. It is chosen when perspective accuracy is required and the volume fits in memory with three RLE copies [S2]. It does not suffer from performance degradation when the major viewing axis changes, provided the three RLE copies fit in memory [S2].

## Operating characteristics
*   **Time Complexity:** Rendering time is generally higher than parallel projection due to the additional scaling and resampling operations [S2].
*   **Memory Complexity:** Requires three RLE copies of the volume in memory to maintain performance across all viewing angles [S2]. For a $256 \times 256 \times 226$ dataset, this totals 9.8 Mbytes [S2].
*   **Measured Performance:** On an SGI Indigo R4000, the algorithm renders the $128 \times 128 \times 109$ 'brainsmall' dataset in an average of 1.0 seconds and the $128 \times 128 \times 113$ 'headsmall' dataset in 1.1 seconds [S2]. Color renderings take 1.3x longer than grayscale renderings [S2].
*   **Strengths:** Supports perspective viewing; performance is stable regardless of the major viewing axis if memory constraints are met [S2].
*   **Limitations:** Requires more memory than parallel methods; scaling factors mean voxel scanlines are not traversed at the same rate as image scanlines, complicating hardware acceleration [S1, unofficial].

## Background
The method was introduced as an extension to the shear-warp factorization to support perspective viewing transformations [S2]. It was developed by the authors of the "Fast Volume Rendering Using a Shear-warp Factorization" paper [S2]. It is used in medical imaging and scientific visualization where perspective accuracy is critical [S2].

## Sources
[S1] Shear-Warp Volume Rendering, https://www.cs.unh.edu/~cs880/lectureNotes/ShearWarp.color.pdf, fetched 2026-10-27 (unofficial)
[S2] Fast Volume Rendering Using a Shear-warp Factorization of The Viewing Transformation, http://i.stanford.edu/pub/cstr/reports/csl/tr/95/678/CSL-TR-95-678.pdf, fetched 2026-11-04
