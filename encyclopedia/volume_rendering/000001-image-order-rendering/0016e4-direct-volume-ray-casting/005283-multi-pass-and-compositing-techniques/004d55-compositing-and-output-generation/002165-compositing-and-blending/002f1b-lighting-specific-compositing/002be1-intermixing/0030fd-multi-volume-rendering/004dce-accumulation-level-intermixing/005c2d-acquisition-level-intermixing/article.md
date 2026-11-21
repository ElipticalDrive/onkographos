# Acquisition Level Intermixing

Also called: Data intermixing at acquisition level, Statistical Vectorial Volume Classification, Vectorial Volume Classification

## Overview
Acquisition Level Intermixing is a multi-volume rendering framework that fuses multiple spatially aligned scalar volumes into a single vectorial volume before the standard rendering pipeline begins [S1, S2]. Unlike [Accumulation-level intermixing](13575-accumulation-level-intermixing.md), which combines ray samples during traversal, this method performs statistical classification on the combined data to extract material properties, thereby preserving depth ordering and enabling joint analysis of dependent acquisitions [S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** $N$ scalar volumes $\{V_1, V_2, \dots, V_N\}$ representing different acquisitions of the same subject, assumed to be spatially aligned on a common grid.
*   **Outputs:** A final rendered image based on classified material properties.

**Data Structures:**
*   **Vectorial Volume ($V_{vec}$):** A 3D grid where each voxel at position $\mathbf{x}$ contains a vector $\mathbf{v}(\mathbf{x})$ of dimension $N$.
    *   Layout: $\mathbf{v}(\mathbf{x}) = [v_1(\mathbf{x}), v_2(\mathbf{x}), \dots, v_N(\mathbf{x})]$, where $v_i(\mathbf{x})$ is the scalar value from volume $V_i$ at $\mathbf{x}$.
    *   Size: $W \times H \times D \times N$ elements, where $W, H, D$ are the grid dimensions.

**Parameters:**
*   $N$: Number of acquisitions (dimensions of the vectorial volume).
*   $K$: Total number of tissues or materials to classify [S2].
*   $C_k$: Assigned color for material $k$, where $1 \le k \le K$ [S2].
*   $\alpha_k$: Assigned global opacity for material $k$, where $1 \le k \le K$ [S2].

**Steps:**

1.  **3D/3D Image Registration:** Align the $N$ input volumes into the same spatial space. This ensures that corresponding physical locations in different acquisitions map to the same voxel coordinates [S1, S2].
2.  **Form Vectorial Volume:** Construct $V_{vec}$ by stacking the aligned scalar volumes. For every voxel $\mathbf{x}$, create the vector $\mathbf{v}(\mathbf{x})$ containing the values from all $N$ acquisitions [S1, S2].
3.  **Statistical Classification:** Apply a statistical multi-dimensional classification method to $V_{vec}$ to estimate material probability distributions. The sources specify a **neighborhood weighted Gaussian mixture model** [S1, S2]. This step yields the probability that a voxel belongs to each of the $K$ materials [S2].
    *   *Note:* The sources cite a previous paper [9] in [S1] and [14] in [S2] for the detailed algorithm of this classification, noting it is less affected by noise and avoids mis-classification on boundaries [S2]. The specific mathematical formulation of the neighborhood weighted Gaussian mixture model is not provided in the current text [S1, S2].
4.  **Assign Material Properties:** Based on the classification results, assign the material color $C_k$ and global opacity $\alpha_k$ for each of the $K$ tissues [S2].
5.  **Rendering:** Render the multi-volume using the classified material properties. The method supports both surface rendering (extracting geometric primitives) and direct volume rendering techniques [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def acquisition_level_intermixing(volumes, num_tissues):
    """
    volumes: List of N scalar 3D arrays (volumes), all same shape (W, H, D).
    num_tissues: Integer K, number of materials to classify.
    Returns: Rendered image (or material property maps).
    """
    # Step 1: Registration
    # The input volumes must already be registered to a common grid.
    # If not, perform 3D/3D image registration here.
    # [G] Implementation of registration depends on external tools (e.g., ITK, Elastix).
    # We assume 'volumes' are already aligned.
    
    N = len(volumes)
    shape = volumes[0].shape
    
    # Step 2: Form Vectorial Volume
    # Stack volumes along a new axis to create a vector per voxel.
    # Resulting shape: (W, H, D, N)
    vectorial_volume = np.stack(volumes, axis=-1)
    
    # Step 3: Statistical Classification
    # Apply neighborhood weighted Gaussian mixture model.
    # This returns material probability distributions for each voxel.
    # [G] The specific algorithm for 'classify_vectorial_volume' is detailed in 
    # external references [S1, Ref 9] and [S2, Ref 14].
    material_probabilities = classify_vectorial_volume(vectorial_volume, num_tissues)
    
    # Step 4: Assign Material Properties
    # C_k: Color for tissue k
    # alpha_k: Opacity for tissue k
    # These are user-defined or derived from the classification confidence.
    colors = assign_colors(num_tissues)
    opacities = assign_opacities(num_tissues)
    
    # Step 5: Rendering
    # Render using the classified properties.
    # This can be surface rendering or direct volume rendering.
    image = render_material_properties(material_probabilities, colors, opacities)
    
    return image
```

**Edge Cases and Pitfalls:**
*   **Registration Error:** If volumes are not perfectly aligned, the vectorial volume will contain misaligned data, leading to incorrect classification and rendering artifacts [S1, S2].
*   **Noise:** The classification method is chosen specifically to be less affected by noise, but poor signal-to-noise ratio in input acquisitions may still degrade material probability estimates [S2].
*   **Boundary Mis-classification:** Standard classification may fail at boundaries; the neighborhood weighted approach is used to mitigate this [S2].

## Operating characteristics
*   **Time Complexity:** Dominated by the 3D/3D registration step and the multi-dimensional statistical classification (Gaussian mixture model) on the vectorial volume. The classification operates on $W \times H \times D$ voxels with $N$ dimensions.
*   **Memory Complexity:** Requires storing the vectorial volume, which is $N$ times the size of a single scalar volume.
*   **Strengths:**
    *   Preserves depth ordering by mixing data at the earliest stage (acquisition level) rather than during ray casting [S2].
    *   Enables joint statistical analysis of dependent acquisitions (e.g., from the same patient) [S1, S2].
    *   Reduces mis-classification on boundaries compared to scalar-only methods [S2].
*   **Limitations:**
    *   Requires high-quality 3D/3D registration.
    *   Computationally intensive due to multi-dimensional classification.
    *   Not suitable for independent volumes where spatial correspondence is not meaningful [S1].
*   **When to use:** Multi-modal medical imaging visualization where multiple acquisitions of the same subject need to be combined (e.g., preoperative kidney planning) [S2].
*   **When not to use:** When volumes are not spatially aligned or represent independent subjects [S1].

## Background
Acquisition Level Intermixing was proposed to address the limitations of existing multi-volume rendering methods that mix data at specific steps of the rendering pipeline (image, composition, or illumination levels). The authors argue that volumes from the same patient are not independent and should be analyzed together [S1, S2]. The method was demonstrated in the context of medical imaging, specifically for preoperative kidney planning [S2]. The statistical classification component relies on a neighborhood weighted Gaussian mixture model, detailed in prior work by the same authors [S1, Ref 9; S2, Ref 14].

## Sources
[S1] A Multi-Volume Visualization Framework for Spatial Aligned Volumes after 3D/3D Image Registration, https://doi.org/10.1109/ICISE.2009.77, fetched 2026-10-26
[S2] A multivolume visualization method for spatially aligned volumes after 3D/3D image registration, https://doi.org/10.1016/j.irbm.2011.01.040, fetched 2026-10-26
