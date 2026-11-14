# Image-Centric Volume Rendering with SN-SVM Classification

Also called: Image-centric approach

## Overview
This method enables direct volume rendering by allowing users to interact with raw 2D volume slices to define training data for an SN-SVM classifier, which then automatically classifies all voxels and assigns colors via color harmonization [S1]. It distinguishes itself from siblings like [Classified Volume Rendering](3590-classified-volume-rendering.md) by avoiding pre-processing steps such as branch decomposition or contour tree computation, thereby preserving raw data granularity and preventing information loss during initial semantic realization [S1].

## How it works
**Inputs:** Raw 3D volume data (intensity values per voxel) [S1].
**Outputs:** Classified volume (voxel labels) and a color/opacity map for each class [S1].

**Data Structures:**
*   **Volume Data:** 3D array of intensity values [G].
*   **Slice Entropy List:** List of slice indices sorted by descending entropy [S1].
*   **Training Set:** List of voxels selected by the user, each tagged with a class label [S1].
*   **Feature Vector:** 5-dimensional vector per voxel: $[I, G, A_X, A_Y, A_Z]$ [S1].
    *   $I$: Intensity value [S1].
    *   $G$: 3D gradient magnitude [S1].
    *   $A_X, A_Y, A_Z$: Average intensities of neighbors in planes perpendicular to X, Y, and Z axes respectively [S1].

**Steps:**

1.  **Slice Selection:** Calculate the image entropy for every slice in the X, Y, and Z directions [S1]. Sort these slices in descending order of entropy [S1]. Select the top three slices (one from each direction) to present to the user [S1].
2.  **User Interaction:** Present the three selected slices in a GUI [S1]. The user draws free-form boundaries around groups of voxels on these slices to select training samples [S1]. Upon clicking "define class," all voxels inside the boundary are assigned to the selected class [S1].
3.  **Feature Extraction:** For every voxel in the volume, extract the 5D feature vector: [S1].
    *   **Intensity ($I$):** The raw value of the voxel [S1].
    *   **Gradient Magnitude ($G$):** Calculate gradient components $G_x, G_y, G_z$ and compute: [S1]
        $$ G = \sqrt{G_x^2 + G_y^2 + G_z^2} $$
    *   **Average Neighboring Intensities ($A_X, A_Y, A_Z$):** [S1]
        *   $A_X$: Fix the X index. Average the intensities of the 8 neighbors in the Y-Z plane [S1].
        *   $A_Y$: Fix the Y index. Average the intensities of the 8 neighbors in the X-Z plane [S1].
        *   $A_Z$: Fix the Z index. Average the intensities of the 8 neighbors in the X-Y plane [S1].
4.  **Classification:** Train an SN-SVM classifier using the user-selected training voxels and their extracted 5D features [S1]. Use a one-vs-one scheme for multi-class problems [S1]. Classify all voxels in the volume based on their features [S1].
5.  **Color Assignment:** Map each class to a color and opacity value using color harmonization techniques to ensure distinguishable and aesthetically pleasing results [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
from collections import Counter

def calculate_entropy(slice_data):
    """Calculate Shannon entropy of a 2D slice."""
    flat = [val for row in slice_data for val in row]
    counts = Counter(flat)
    total = len(flat)
    entropy = 0.0
    for count in counts.values():
        p = count / total
        if p > 0:
            entropy -= p * math.log2(p)
    return entropy

def calculate_gradient_magnitude(volume, x, y, z):
    """Calculate G = sqrt(Gx^2 + Gy^2 + Gz^2) using central differences [G]."""
    def get_val(ix, iy, iz):
        ix = max(0, min(ix, volume.shape[0] - 1))
        iy = max(0, min(iy, volume.shape[1] - 1))
        iz = max(0, min(iz, volume.shape[2] - 1))
        return volume[ix, iy, iz]
    Gx = get_val(x + 1, y, z) - get_val(x - 1, y, z)
    Gy = get_val(x, y + 1, z) - get_val(x, y - 1, z)
    Gz = get_val(x, y, z + 1) - get_val(x, y, z - 1)
    return math.sqrt(Gx**2 + Gy**2 + Gz**2)

def calculate_average_neighbor_intensity(volume, x, y, z, axis):
    """Calculate average intensity of 8 neighbors in the plane perpendicular to 'axis'."""
    sum_intensity = 0.0
    count = 0
    def get_val(ix, iy, iz):
        ix = max(0, min(ix, volume.shape[0] - 1))
        iy = max(0, min(iy, volume.shape[1] - 1))
        iz = max(0, min(iz, volume.shape[2] - 1))
        return volume[ix, iy, iz]
    if axis == 'X':
        for dy in (-1, 0, 1):
            for dz in (-1, 0, 1):
                if dy == 0 and dz == 0: continue
                sum_intensity += get_val(x, y + dy, z + dz)
                count += 1
    elif axis == 'Y':
        for dx in (-1, 0, 1):
            for dz in (-1, 0, 1):
                if dx == 0 and dz == 0: continue
                sum_intensity += get_val(x + dx, y, z + dz)
                count += 1
    elif axis == 'Z':
        for dx in (-1, 0, 1):
            for dy in (-1, 0, 1):
                if dx == 0 and dy == 0: continue
                sum_intensity += get_val(x + dx, y + dy, z)
                count += 1
    return sum_intensity / count if count > 0 else 0.0

def extract_features(volume):
    features = {}
    for x in range(volume.shape[0]):
        for y in range(volume.shape[1]):
            for z in range(volume.shape[2]):
                intensity = volume[x, y, z]
                grad_mag = calculate_gradient_magnitude(volume, x, y, z)
                avg_x = calculate_average_neighbor_intensity(volume, x, y, z, 'X')
                avg_y = calculate_average_neighbor_intensity(volume, x, y, z, 'Y')
                avg_z = calculate_average_neighbor_intensity(volume, x, y, z, 'Z')
                features[(x, y, z)] = [intensity, grad_mag, avg_x, avg_y, avg_z]
    return features

def image_centric_volume_rendering(volume):
    # 1. Slice Selection
    slice_entropies = {
        'X': [(i, calculate_entropy(volume[i, :, :])) for i in range(volume.shape[0])],
        'Y': [(j, calculate_entropy(volume[:, j, :])) for j in range(volume.shape[1])],
        'Z': [(k, calculate_entropy(volume[:, :, k])) for k in range(volume.shape[2])]
    }
    selected_slices = []
    for axis in ['X', 'Y', 'Z']:
        sorted_slices = sorted(slice_entropies[axis], key=lambda item: item[1], reverse=True)
        selected_slices.append((axis, sorted_slices[0][0]))

    # 2. User Interaction: User draws boundaries on selected slices and assigns classes [G]
    training_data = {} # Populated by GUI interaction

    # 3. Feature Extraction
    all_features = extract_features(volume)

    # 4. Classification: Train SN-SVM on training_data features and classify all voxels [G]
    voxel_classes = {} # Populated by SVM inference

    # 5. Color Assignment: Map classes to colors using color harmonization [G]
    color_map = {} # Populated by color mapping algorithm

    return voxel_classes, color_map
```

**Parameters:**
*   **Number of slices:** 3 (one per axis) [S1].
*   **Feature dimension:** 5 [S1].
*   **Classifier:** SN-SVM with one-vs-one scheme for multi-class [S1].

**Edge Cases and Pitfalls:**
*   **Boundary Voxels:** Neighbor averaging and gradient calculation require handling volume boundaries [G].
*   **Small Training Sets:** The method relies on the robustness of SN-SVM to work with small numbers of training samples [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1].
*   **Memory Complexity:** Not given in the sources [S1].
*   **Strengths:** Provides granular control by interacting with raw data [S1]. Avoids information loss from pre-processing [S1]. Automated color assignment improves aesthetics [S1].
*   **Limitations:** Requires user interaction to define training data [S1]. Performance depends on the quality of the selected training samples [S1].
*   **When to use:** When users need detailed control over classification without complex pre-processing and want aesthetically pleasing automatic coloring [S1].

## Background
Introduced by the authors of [S1], the method contrasts with [Guo and Yuan 2013] which use branch decomposition and contour trees for pre-processing [S1]. The authors conducted a quantitative user survey comparing their system with other DVR methods, noting that most evaluations are qualitative [S1].

## Sources
[S1] A Novel Image-Centric Approach Toward Direct Volume Rendering, https://doi.org/10.1145/3152875, fetched 2026-10-29
