# Linear Look-up Table Classification

Also called: Histogram-based classification, Nearest neighbor classification, VolPack Lookup Table Classification, vpRamp, vpSetClassifierTable

## Overview
Linear look-up table classification is a volume rendering method that maps voxel intensity values to material composition percentages using a pre-calculated linear table [S1, unofficial]. It solves the problem of assigning physical properties to discrete voxel data by treating classification as a nearest-neighbor decision based on density peaks [S1, unofficial]. Unlike cubic spline or piecewise linear interpolation methods which allow arbitrary opacity curves, this method enforces a strict linear ordering of materials, making it computationally simple but restricted to scenarios where materials are distinct and adjacent in density space [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A volume dataset where each voxel has one or more scalar fields (e.g., density, T1/T2 MRI values).
*   **Output:** For each voxel, a set of material percentage values (typically 0.0 to 1.0) and a final opacity value derived from the product of these percentages [S1, unofficial][S2].

**Data Structures:**
The core data structure is a linear look-up table (LUT) [S1, unofficial].
*   **VolPack Implementation:** The table is a contiguous array of floats [S2].
*   **Indexing:** The voxel field value (e.g., intensity) serves as the index into the table [S2].
*   **Table Size:** Defined in bytes, not number of entries [S2].
*   **Association:** Tables are associated with specific voxel fields via a parameter number and field index [S2].

**Prerequisites (Conditions for Correctness):**
1.  Each material corresponds to a known, distinct density value [S1, unofficial].
2.  Each voxel contains no more than two materials [S1, unofficial].
3.  If a voxel contains two materials, their densities must follow each other in the specific order defined by the linear table [S1, unofficial].

**Algorithm Steps:**

1.  **Histogram Analysis:** Analyze the image histogram to identify peaks. These peaks represent the centers of the density regions for the materials [S1, unofficial].
2.  **Table Initialization:** Create a piecewise linear ramp to define the table values.
    *   Define `ramp_x` coordinates (table indices) corresponding to the density boundaries or centers.
    *   Define `ramp_y` coordinates (values) representing the desired material percentage or opacity at those indices.
    *   Linearly interpolate values between the `ramp_x` points [S1, unofficial][S2].
3.  **Table Association:** Associate the initialized table with the specific voxel field used for classification [S2].
4.  **Classification (Runtime):**
    *   For each voxel, retrieve the value from the designated field.
    *   Use this value to index into the look-up table.
    *   Retrieve the material percentage value from the table.
    *   If multiple fields are used, multiply the retrieved values from their respective tables to determine the final opacity [S2].
5.  **Thresholding:** Apply a minimum opacity threshold to discard transparent voxels [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Step 1: Define ramp parameters based on histogram peaks
# ramp_x: indices in the table (e.g., intensity values)
# ramp_y: target material percentages or opacities
ramp_x = [0, 24, 255]
ramp_y = [0.0, 1.0, 1.0]
num_points = len(ramp_x)
table_size_bytes = (max(ramp_x) + 1) * 4 # Assuming float32, size covers max index

# Step 2: Initialize the lookup table with linear interpolation
# This creates a piecewise linear function between the defined points
lookup_table = [0.0] * (max(ramp_x) + 1)

for i in range(num_points - 1):
    x0 = ramp_x[i]
    x1 = ramp_x[i+1]
    y0 = ramp_y[i]
    y1 = ramp_y[i+1]
    
    # Linear interpolation for all indices between x0 and x1
    for idx in range(x0, x1 + 1):
        if x1 == x0:
            lookup_table[idx] = y0
        else:
            # Standard linear interpolation formula
            t = (idx - x0) / (x1 - x0)
            lookup_table[idx] = y0 + t * (y1 - y0)

# Step 3: Associate table with voxel field (VolPack specific API call)
# param_num: 0 (first parameter)
# param_field: SCALAR_FIELD (index of the voxel field)
# table: pointer to lookup_table
# table_size: table_size_bytes
# vpSetClassifierTable(vpc, 0, SCALAR_FIELD, lookup_table, table_size_bytes)

# Step 4: Set minimum opacity threshold
# vpSeti(vpc, VP_MIN_VOXEL_OPACITY, threshold)

# Step 5: Classification for a single voxel
def classify_voxel(voxel_intensity, lookup_table, min_opacity=0.0):
    # Clamp intensity to valid table range
    idx = int(voxel_intensity)
    if idx < 0:
        idx = 0
    elif idx >= len(lookup_table):
        idx = len(lookup_table) - 1
        
    # Retrieve material percentage
    material_percentage = lookup_table[idx]
    
    # Apply threshold
    if material_percentage < min_opacity:
        return 0.0
    else:
        return material_percentage
```

**Edge Cases and Pitfalls:**
*   **Ordering Violation:** The method fails if materials do not follow the linear ordering in density. For example, at an air-bone interface, soft tissue may appear between air and bone in density, violating the adjacency criteria required for the linear table [S1, unofficial].
*   **Multi-Material Voxels:** The method assumes voxels contain no more than two materials. Voxels with three or more materials will be misclassified [S1, unofficial].
*   **Memory Management:** In VolPack, the lookup table memory is never deallocated by the library, even if the rendering context is destroyed. The caller is responsible for managing this memory leak if not using the library's internal management [S2].
*   **Precision:** The table maps discrete indices to continuous values. If the input data has high precision but the table is sparse, interpolation errors may occur. However, the linear interpolation within the table mitigates this for the defined ramps [S2].

## Subtopics

### Multi-variable Scatter-plot Classification
This variant extends linear look-up table classification to multiple image parameters (e.g., T1 and T2 weighted MRI densities) [S1, unofficial]. Instead of a 1D histogram, it uses a 2D scatter-plot (histogram of two inputs) to locate clusters representing materials in a 2D space [S1, unofficial]. Classification is performed using a nearest-neighbor decision function where the distance is the length of the chord joining the center of each 2D region with the 2D point corresponding to the voxel's parameters [S1, unofficial]. This allows for better separation of materials that overlap in a single density dimension but are distinct in multi-parametric space [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   **Time:** $O(1)$ per voxel for classification, as it involves a single array lookup and optional multiplication [S1, unofficial][S2].
*   **Memory:** $O(N)$ where $N$ is the size of the lookup table in bytes. The table size is determined by the range of the input data and the desired resolution of the classification [S2].

**Measured Performance:**
*   The method is extremely fast due to the simplicity of the lookup operation [S1, unofficial].
*   Performance is hardware-independent as it relies on basic memory access and arithmetic [S2].

**Scaling:**
*   Scales linearly with the number of voxels [S1, unofficial].
*   Does not scale well to complex scenes with many materials or high-dimensional parameter spaces without significant increases in table size or complexity [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Simple to implement, computationally efficient, effective for data with distinct, non-overlapping density peaks (e.g., neurological CT) [S1, unofficial].
*   **Limitations:** Cannot handle materials with overlapping density distributions or complex interfaces (e.g., air-bone-soft tissue) without manual intervention or carving [S1, unofficial]. Requires strict ordering of materials [S1, unofficial].

**When to Use:**
*   Use for datasets where materials have distinct, known density values and voxels contain at most two materials [S1, unofficial].
*   Suitable for neurological CT data where bone, air, and soft tissue are well-separated in density [S1, unofficial].

**When Not to Use:**
*   Do not use for datasets with significant partial volume effects involving three or more materials [S1, unofficial].
*   Do not use when materials have overlapping density ranges that cannot be resolved by a simple linear ordering [S1, unofficial].

## Background
**Origin:**
The method was described in the context of volumetric rendering of medical data, specifically for stereotactic neurosurgery planning [S1, unofficial]. It is also implemented in the VolPack library, a standard tool for volume rendering [S2].

**History and Development:**
The technique is based on the principle of nearest-neighbor classification using histogram analysis [S1, unofficial]. It was developed to provide a simple, automated way to segment structures of interest in medical imaging without resorting to complex surface rendering methods [S1, unofficial]. The VolPack implementation provides utility functions like `vpRamp` to facilitate the creation of these tables [S2].

**Where it is Used:**
*   Neurological CT data visualization [S1, unofficial].
*   Applications requiring fast, simple classification of scalar fields [S2].

## Sources
[S1] Volumetric rendering of medical data : applications to stereotactic neurosurgery planning, https://doi.org/10.82308/10070, fetched 2026-10-26 (unofficial)
[S2] VolPack User's Guide, http://www-graphics.stanford.edu/software/volpack/vp_userguide.html, fetched 2026-10-28
