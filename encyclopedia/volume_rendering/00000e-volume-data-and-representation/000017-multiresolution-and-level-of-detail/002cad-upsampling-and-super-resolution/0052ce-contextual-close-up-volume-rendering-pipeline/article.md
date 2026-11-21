# Contextual Close-up Volume Rendering Pipeline

Also called: Contextual Close-up Rendering, Focus of Attention, Multi-resolution B-spline volume rendering with gradient quantization, Real-Time Super Resolution Contextual Close-up, Volume Lens

## Overview
The Contextual Close-up Volume Rendering Pipeline is a visualization technique for real-time, high-quality rendering of clinical volumetric data that provides detailed views of Regions of Interest (ROI) while preserving the global context of the full volume [S1]. It distinguishes itself from hierarchical brick-based LOD methods [S1] and single-table gradient quantization [S1] by decimating the entire dataset for interactive navigation, using B-spline subdivision to super-sample a specific ROI, and employing a dual-access gradient quantization scheme to maintain feature fidelity in the enlarged view [S1].

## How it works
The method consists of a preprocessing phase to prepare data structures and a runtime phase to render the contextual close-up.

### Inputs and Outputs
*   **Input:** A 3D medical volume dataset (CT/MRI).
*   **Output:** A composite image showing the full volume with a transparent ROI cutout, an enlarged super-resolved ROI in a separate screen area, and a translucent connecting cone.

### Data Structures
1.  **Decimated Volume:** A lower-resolution copy of the original volume, generated via B-spline reverse subdivision, used for fast interaction [S1].
2.  **Primary Gradient Lookup Table:** A "static" table storing quantized gradient vectors for the main data volume [S1].
3.  **Secondary Gradient Table:** A table storing indices pointing to entries in the Primary Gradient Table, used to map arbitrary new gradients from the super-resolved ROI to existing quantized entries [S1].
4.  **Gradient Index Volume:** A 3D array with the same dimensions as the data volume, storing the index into the quantized gradient table for each voxel [S1].
5.  **Gradient 3D Texture:** The Gradient Index Volume uploaded to GPU memory. Each texel stores:
    *   **R, G components:** Index into the quantized gradient table (used to access the display intensity table) [S1].
    *   **B component:** Magnitude of the quantized gradient [S1].
6.  **Display Intensity Table:** A list of greyscale values, same length as the Primary Gradient Table. Elements are calculated via the inner product of the view/lighting vector and the quantized gradient vectors [S1].
7.  **ROI Definition:** Stores the 3D center position and box dimensions of the ROI [S1].

### Parameters
*   **ROI Sub-volume Dimensions:** $64^3$ voxels [S1].
*   **Super-resolution Sub-volume Dimensions:** $128^3$ voxels [S1].
*   **B-spline Order:** User-selectable (e.g., Order 3 or 4) to control smoothness [S1].
*   **Preprocessing Time:** Gradient quantization requires a few seconds [S1].
*   **Display Table Update Time:** Typically < 1 ms [S1].

### Steps

1.  **Load and Decimate:** Load the data volume. Apply fast B-spline multi-resolution reverse subdivision to generate a decimated copy for fast interaction [S1].
2.  **Calculate and Quantize Gradients:** Calculate gradients $\mathbf{V}^*$ for each voxel in the main data volume. Quantize these gradients using a highly optimized algorithm [S1].
3.  **Build Primary Table:** Store the unique quantized gradients in the Primary Gradient Lookup Table [S1].
4.  **Build Secondary Table:** Construct the Secondary Gradient Table. This table maps arbitrary gradient vectors (expected from the super-resolved ROI) to indices in the Primary Table [S1].
5.  **Build Gradient Index Volume:** For each voxel in the data volume, store the index pointing to its quantized gradient in the Primary Table. This forms the Gradient Index Volume [S1].
6.  **Upload to GPU:** Upload the Gradient Index Volume to the GPU as a 3D texture. Pack the index into the R and G channels and the gradient magnitude into the B channel [S1].
7.  **Initialize Display Intensity Table:** Create a table of greyscale values matching the length of the Primary Gradient Table. Initialize it by calculating the inner product between the initial view/lighting vector and each quantized gradient in the Primary Table [S1].
8.  **Define ROI:** Store the ROI center (3D position) and size (box dimensions) [S1].
9.  **Runtime Rendering Loop:**
    *   **Update ROI:** If the user moves the ROI, update the sub-volume definition based on the new center and size [S1].
    *   **Render Full Volume:** Render the original full-resolution volume. Use a stencil buffer to fully expose (make transparent/cull) the area corresponding to the ROI [S1].
    *   **Super-Resample ROI:** Extract the ROI sub-volume from the data. Apply B-spline subdivision to upsample it from $64^3$ to $128^3$ [S1].
    *   **Render ROI:** Render the super-resolved ROI texture enlarged in a circular or rectangular window in a separate screen area [S1]. Use the Secondary Gradient Table to map the finer gradients of the super-resolved voxels to the quantized gradients for shading [S1].
    *   **Render Connector:** Render a translucent cone connecting the physical location of the ROI in the original volume to the separated screen area. Use luminance for depth cueing [S1].
    *   **Update Shading:** If the view or lighting changes, update the Display Intensity Table by recalculating the inner products for all entries in the Primary Gradient Table [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def preprocess_volume(volume_data):
    """
    Step 1-6: Preprocessing phase.
    """
    # Step 1: Decimate volume for interaction
    decimated_volume = bspline_reverse_subdivision(volume_data, factor=2)
    
    # Step 2: Calculate and quantize gradients for main volume
    # V_star is the gradient vector for each voxel
    gradients = calculate_gradients(volume_data)
    quantized_gradients, primary_table = fast_gradient_quantization(gradients)
    
    # Step 3: Build secondary table for arbitrary gradients
    secondary_table = build_secondary_gradient_table(quantized_gradients)
    
    # Step 4: Create index volume
    # Map each voxel's gradient to an index in primary_table
    gradient_index_volume = create_index_volume(volume_data, primary_table)
    
    # Step 5: Pack into 3D texture
    # R, G = index; B = magnitude
    gradient_3d_texture = pack_gradient_texture(gradient_index_volume)
    
    # Step 6: Initialize display intensity table
    # Inner product of view vector and quantized gradients
    display_intensity_table = compute_display_intensities(primary_table, view_vector)
    
    return {
        'decimated_volume': decimated_volume,
        'primary_table': primary_table,
        'secondary_table': secondary_table,
        'gradient_3d_texture': gradient_3d_texture,
        'display_intensity_table': display_intensity_table
    }

def render_contextual_closeup(preprocessed_data, roi_center, roi_size, view_vector):
    """
    Step 7-9: Runtime rendering phase.
    """
    primary_table = preprocessed_data['primary_table']
    secondary_table = preprocessed_data['secondary_table']
    gradient_3d_texture = preprocessed_data['gradient_3d_texture']
    
    # Step 8: Define ROI
    roi_box = Box(center=roi_center, size=roi_size)
    
    # Step 9a: Render full volume with ROI exposed
    # Use stencil buffer to mask out ROI area
    render_full_volume(preprocessed_data['decimated_volume'], stencil_mask=roi_box)
    
    # Step 9b: Super-resample ROI
    roi_subvolume = extract_subvolume(preprocessed_data['decimated_volume'], roi_box)
    # Upsample 64^3 to 128^3
    super_resolved_roi = bspline_subdivision(roi_subvolume, order=3)
    
    # Step 9c: Render ROI in separate window
    # Use secondary table to map super-resolved gradients to primary table indices
    roi_shading = compute_roi_shading(super_resolved_roi, secondary_table, primary_table)
    render_window(roi_shading, window_shape='circular')
    
    # Step 9d: Render connecting cone
    render_cone(roi_center, window_position, luminance_depth_cueing=True)
    
    # Step 9e: Update display intensity table if view changed
    # Typically < 1 ms
    preprocessed_data['display_intensity_table'] = compute_display_intensities(
        primary_table, view_vector
    )
```

### Edge Cases and Pitfalls
*   **Gradient Mismatch:** The secondary table must efficiently handle arbitrary gradients from the super-resolved ROI that may not exist exactly in the primary table. The sources describe a "dual-access" mechanism but do not specify the exact hashing or nearest-neighbor algorithm used for this mapping [S1].
*   **Stencil Buffer Occlusion:** The ROI must be fully exposed in the full-volume render. Incorrect stencil setup may leave artifacts or occlude the ROI incorrectly [S1].
*   **Depth Cueing:** The translucent cone uses luminance for depth cueing. If the ROI is moved significantly, the cone must be updated to maintain correct spatial connection [S1].

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
