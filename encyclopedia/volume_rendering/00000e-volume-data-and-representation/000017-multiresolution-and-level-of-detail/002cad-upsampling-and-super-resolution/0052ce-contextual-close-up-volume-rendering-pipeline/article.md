# Contextual Close-up Volume Rendering Pipeline

Also called: Contextual Close-up Rendering, Focus of Attention, Multi-resolution B-spline volume rendering with gradient quantization, Real-Time Super Resolution Contextual Close-up, Volume Lens

## Overview
The Contextual Close-up Volume Rendering Pipeline is a visualization technique for real-time, high-quality rendering of clinical volumetric data that provides detailed views of Regions of Interest (ROI) while preserving the global context of the full volume [S1]. It distinguishes itself from hierarchical brick-based LOD methods [S1] and single-table gradient quantization [S1] by decimating the entire dataset for interactive navigation, using B-spline subdivision to super-sample a specific ROI, and employing a dual-access gradient quantization scheme to maintain feature fidelity in the enlarged view [S1].

## How it works
The method consists of a preprocessing phase to prepare data structures and a runtime phase to render the contextual close-up [S1].

### Inputs and Outputs
*   **Input:** A 3D medical volume dataset (CT/MRI) [S1].
*   **Output:** A composite image showing the full volume with a transparent ROI cutout, an enlarged super-resolved ROI in a separate screen area, and a translucent connecting cone [S1].

### Data Structures
1.  **Decimated Volume:** A lower-resolution copy of the original volume, generated via B-spline reverse subdivision, used for fast interaction [S1].
2.  **Primary Gradient Lookup Table:** A "static" table storing quantized gradient vectors for the main data volume [S1].
3.  **Secondary Gradient Table:** A table storing indices pointing to entries in the Primary Gradient Table, used to map arbitrary new gradients from the super-resolved ROI to existing quantized entries [S1].
4.  **Gradient Index Volume:** A 3D array with the same dimensions as the data volume, storing the index into the quantized gradient table for each voxel [S1].
5.  **Gradient 3D Texture:** The Gradient Index Volume uploaded to GPU memory. Each texel stores: R and G components holding the index into the quantized gradient table (used to access the display intensity table), and the B component containing the magnitude of the quantized gradient [S1].
6.  **Display Intensity Table:** A list of greyscale values, same length as the Primary Gradient Table. Elements are calculated via the inner product of the view/lighting vector and the quantized gradient vectors [S1].
7.  **ROI Definition:** Stores the 3D center position and box dimensions of the ROI [S1].

### Parameters
*   **ROI Sub-volume Dimensions:** $64^3$ voxels [S1].
*   **Super-resolution Sub-volume Dimensions:** $128^3$ voxels [S1].
*   **B-spline Order:** Configurable (e.g., 3 or 4) to control smoothness [S1].
*   **Preprocessing Time:** Gradient quantization requires a few seconds [S1].
*   **Display Table Update Time:** Typically less than 1 ms [S1].

### Steps
1.  Load the data volume and apply a fast B-spline multi-resolution reverse subdivision technique to generate a decimated copy for fast interaction [S1].
2.  Calculate gradients $\mathbf{V}^*$ for each voxel in the main data volume and quantize them using a highly optimized algorithm [S1].
3.  Store the unique quantized gradients in a short "static" primary lookup table [S1].
4.  Construct a secondary gradient table with indices pointing to entries in the primary table, enabling access to existing quantized gradients for arbitrary new gradient vectors from the super-resolution ROI [S1].
5.  For each voxel in the data volume, store the index pointing to its quantized gradient in the primary table, forming a 3D array matching the data volume dimensions [S1].
6.  Upload this 3D array to the GPU as a 3D texture, packing the index into the R and G channels and the gradient magnitude into the B channel [S1].
7.  Create a display intensity table of greyscale values matching the primary table length. Initialize it by calculating the inner product between the current view (or lighting) vector and each quantized gradient vector [S1].
8.  Define the ROI by storing its 3D center position and box dimensions [S1].
9.  During runtime, update the ROI definition as the user moves it via mouse controls [S1]. Render the original full-resolution volume with the ROI area fully exposed (transparent/culled) using a stencil buffer shape [S1]. Extract the ROI sub-volume, apply B-spline subdivision to yield a $128^3$ super-resolution sub-volume from the $64^3$ input, and render it enlarged in a circular or rectangular window in a separate screen area [S1]. Use the secondary gradient indexing method to allow real-time gradient-dependent rendering of the ROI [S1]. Render a translucent cone connecting the physical location of the ROI to the separated screen area, using luminance for depth cueing [S1]. Update the display intensity table whenever the view changes [S1].

### Formulas
*   **Gradient Quantization:** $\mathbf{V}^* = \text{Quantize}(\nabla f)$, where $\nabla f$ is the voxel gradient and $\mathbf{V}^*$ is the quantized vector [S1].
*   **Display Intensity:** $I = \mathbf{v} \cdot \mathbf{V}^*$, where $\mathbf{v}$ is the view/lighting vector and $I$ is the greyscale value stored in the display intensity table [S1].
*   **Super-resolution Scaling:** Input ROI is $64^3$ voxels, output is $128^3$ voxels via B-spline subdivision [S1].

### Edge Cases and Pitfalls
*   **Gradient Mismatch:** The secondary table must efficiently handle arbitrary gradients from the super-resolved ROI that may not exist exactly in the primary table [S1].
*   **Stencil Buffer Setup:** The ROI must be fully exposed in the full-volume render to maintain context [S1].
*   **Depth Cueing:** The translucent cone uses luminance for depth cueing to maintain spatial connection as the ROI moves [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def preprocess_volume(volume_data, view_vector):
    # Step 1: Decimate volume for interaction
    decimated_volume = bspline_reverse_subdivision(volume_data, factor=2)
    # Step 2: Calculate and quantize gradients
    gradients = calculate_gradients(volume_data)
    quantized_gradients, primary_table = fast_gradient_quantization(gradients)
    # Step 3: Build secondary table
    secondary_table = build_secondary_gradient_table(quantized_gradients)
    # Step 4: Create index volume
    gradient_index_volume = create_index_volume(volume_data, primary_table)
    # Step 5: Pack into 3D texture
    gradient_3d_texture = pack_gradient_texture(gradient_index_volume)
    # Step 6: Initialize display intensity table
    display_intensity_table = compute_display_intensities(primary_table, view_vector)
    return {
        'decimated_volume': decimated_volume,
        'primary_table': primary_table,
        'secondary_table': secondary_table,
        'gradient_3d_texture': gradient_3d_texture,
        'display_intensity_table': display_intensity_table
    }

def render_contextual_closeup(preprocessed_data, roi_center, roi_size, view_vector):
    primary_table = preprocessed_data['primary_table']
    secondary_table = preprocessed_data['secondary_table']
    gradient_3d_texture = preprocessed_data['gradient_3d_texture']
    # Step 8: Define ROI
    roi_box = Box(center=roi_center, size=roi_size)
    # Step 9a: Render full volume with ROI exposed
    render_full_volume(preprocessed_data['decimated_volume'], stencil_mask=roi_box)
    # Step 9b: Super-resample ROI
    roi_subvolume = extract_subvolume(preprocessed_data['decimated_volume'], roi_box)
    super_resolved_roi = bspline_subdivision(roi_subvolume, order=3)
    # Step 9c: Render ROI in separate window
    roi_shading = compute_roi_shading(super_resolved_roi, secondary_table, primary_table)
    render_window(roi_shading, window_shape='circular')
    # Step 9d: Render connecting cone
    render_cone(roi_center, window_position, luminance_depth_cueing=True)
    # Step 9e: Update display intensity table if view changed
    preprocessed_data['display_intensity_table'] = compute_display_intensities(
        primary_table, view_vector
    )

# Standard operation definitions
def bspline_reverse_subdivision(data, factor): return downsample_b_spline(data, factor)
def calculate_gradients(data): return central_difference_gradient(data)
def fast_gradient_quantization(gradients): return optimized_quantize(gradients)
def build_secondary_gradient_table(quantized_gradients): return map_arbitrary_to_primary(quantized_gradients)
def create_index_volume(data, primary_table): return voxel_to_index_map(data, primary_table)
def pack_gradient_texture(index_volume): return pack_rgb_texture(index_volume)
def compute_display_intensities(primary_table, view_vector): return [dot(view_vector, grad) for grad in primary_table]
def Box(center, size): return {'center': center, 'size': size}
def render_full_volume(data, stencil_mask): draw_volume_with_stencil(data, stencil_mask)
def extract_subvolume(data, roi_box): return crop_volume(data, roi_box)
def bspline_subdivision(data, order): return upsample_b_spline(data, order)
def compute_roi_shading(super_resolved_roi, secondary_table, primary_table): return apply_shading(super_resolved_roi, secondary_table, primary_table)
def render_window(shading, window_shape): draw_window(shading, window_shape)
def render_cone(roi_center, window_position, luminance_depth_cueing): draw_cone(roi_center, window_position, luminance_depth_cueing)
```

## Operating characteristics
*   **Time Complexity:** Preprocessing (gradient quantization) takes a few seconds [S1]. Display intensity table updates take less than 1 ms [S1].
*   **Memory Complexity:** Requires storage for the decimated volume, the primary and secondary gradient tables, and the gradient index volume (3D texture) [S1].
*   **Performance:** Designed for real-time interaction on inexpensive GPUs [S1].
*   **Strengths:** Provides intuitive contextual close-up views similar to medical illustration techniques; preserves global context; fast shading updates [S1].
*   **Limitations:** Uses smaller datasets compared to brick-based LOD methods [S1]. The ROI size is fixed at $64^3$ in the described implementation [S1].
*   **When to use:** When interactive exploration of internal features in clinical volumetric data is required, and maintaining global context is critical [S1].
*   **When not to use:** For very large datasets requiring hierarchical LOD based on distance (use [Cubic Upsampling via Tiling](2342-cubic-upsampling-via-tiling.md) or similar brick-based methods) [S1].

## Background
The technique was presented by S. Chan, J. R. Mitchell, et al. at EuroVis 2006 [S1]. It is inspired by medical illustration techniques for depicting contextual close-up views of selected regions of interest [S1]. It differs from Weiler et al. [WWH*00] and Wang and Shen [WS05] who use brick data structures with hierarchical LOD based on distance [S1]. It also differs from Csebfalvi et al. [CLMAK01] who use a single lookup table of quantized gradient directions [S1].

## Sources
[S1] Real-Time Super Resolution Contextual Close-up of Clinical Volumetric Data, https://doi.org/10.2312/vissym/eurovis06/347-354, fetched 2026-10-26
