# Spherical Harmonics Data Extraction

Also called: PLY Harmonics Data Extraction, f_rest extraction

## Overview
Spherical Harmonics (SH) Data Extraction parses SH coefficients from a 3D Gaussian Splatting model file (typically a PLY format) into a structured data layout suitable for GPU rendering [S1, unofficial]. It solves the problem of mapping flat, property-based storage in the source file to the hierarchical texture arrays required by the rendering engine [S1, unofficial]. It is distinct from [Spherical Harmonics Evaluation](3848-spherical-harmonics-evaluation.md), which computes color from these coefficients, and [Spherical Harmonics Rotation](3985-spherical-harmonics-rotation.md), which transforms them for view-dependent effects [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A PLY file containing vertex data for $N$ Gaussians. The data includes standard properties (position, covariance, opacity) and optional SH properties named `f_rest_y` where $y$ is an integer index [S1, unofficial].
- **Output:** A set of texture data arrays per emitter (Gaussian), organized by SH degree level (L1, L2, L31, L32) [S1, unofficial].

**Data Structures:**
The extraction organizes coefficients into four arrays per emitter [S1, unofficial]. Each coefficient is a 3-component vector (RGB) [S1, unofficial].
1. `harmonicsL1TextureData`: Stores Level 1 harmonics. Contains 9 coefficients (3 degrees $\times$ 3 channels). Layout: 3 pixels per Gaussian, where each pixel holds 3 coefficients [S1, unofficial].
2. `harmonicsL2TextureData`: Stores Level 2 harmonics. Contains 15 coefficients. Layout: 5 pixels per Gaussian [S1, unofficial].
3. `harmonicsL31TextureData`: Stores the first part of Level 3 harmonics. Contains 12 coefficients. Layout: 4 pixels per Gaussian [S1, unofficial].
4. `harmonicsL32TextureData`: Stores the second part of Level 3 harmonics. Contains 9 coefficients. Layout: 3 pixels per Gaussian [S1, unofficial].

Total coefficients per Gaussian: $9 + 15 + 12 + 9 = 45$ (15 SH coefficients $\times$ 3 RGB channels) [S1, unofficial].

**Numbered Steps:**
1. **Detect Higher Order Harmonics:** Iterate $y$ from 0 to 44 with a step of 3 [S1, unofficial]. For each $y$, check if the property key `"f_rest_" + y` exists in the vertex data [S1, unofficial]. If any key is missing, set `higherOrderHarmonicsExists` to `false` and stop [S1, unofficial]. If all are present, set it to `true` [S1, unofficial].
2. **Extract Level 1 (L1):** If `higherOrderHarmonicsExists` is true, iterate $y$ from 0 to 8 with a step of 3 [S1, unofficial]. For each $y$, read the values for indices `"f_rest_y"`, `"f_rest_y+1"`, and `"f_rest_y+2"` for the current Gaussian $i$ [S1, unofficial]. Construct an RGB color vector from these three values [S1, unofficial]. Append this vector to `harmonicsL1TextureData` for the current emitter [S1, unofficial].
3. **Extract Level 2 (L2):** If `higherOrderHarmonicsExists` is true, iterate $y$ from 9 to 23 with a step of 3 [S1, unofficial]. For each $y$, read indices `"f_rest_y"`, `"f_rest_y+1"`, and `"f_rest_y+2"` [S1, unofficial]. Construct the RGB vector and append to `harmonicsL2TextureData` [S1, unofficial].
4. **Extract Level 3 Part 1 (L31):** If `higherOrderHarmonicsExists` is true, iterate $y$ from 24 to 35 with a step of 3 [S1, unofficial]. For each $y$, read indices `"f_rest_y"`, `"f_rest_y+1"`, and `"f_rest_y+2"` [S1, unofficial]. Construct the RGB vector and append to `harmonicsL31TextureData` [S1, unofficial].
5. **Extract Level 3 Part 2 (L32):** If `higherOrderHarmonicsExists` is true, iterate $y$ from 36 to 44 with a step of 3 [S1, unofficial]. For each $y$, read indices `"f_rest_y"`, `"f_rest_y+1"`, and `"f_rest_y+2"` [S1, unofficial]. Construct the RGB vector and append to `harmonicsL32TextureData` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_sh_data(vertex_data: List[Dict[str, float]], num_gaussians: int) -> Dict[str, List[List[float]]]:
    """
    Extracts SH coefficients from PLY vertex data.
    vertex_data: List of dicts, where each dict represents a Gaussian's properties.
                 Keys are strings like 'f_rest_0', 'f_rest_1', etc.
    Returns: Dictionary with keys 'L1', 'L2', 'L31', 'L32', each containing a list of lists of RGB vectors.
    """
    # Step 1: Check for existence of higher order harmonics
    higher_order_exists = True
    for y in range(0, 45, 3):
        key = f"f_rest_{y}"
        if key not in vertex_data[0]:
            higher_order_exists = False
            break
    
    if not higher_order_exists:
        return {'L1': [], 'L2': [], 'L31': [], 'L32': []}

    # Initialize storage for one emitter (Gaussian)
    l1_data = []
    l2_data = []
    l31_data = []
    l32_data = []

    # Process each Gaussian
    for i in range(num_gaussians):
        gauss = vertex_data[i]
        
        # Step 2: Extract L1 (y=0 to 8, step 3)
        for y in range(0, 9, 3):
            r = gauss[f"f_rest_{y}"]
            g = gauss[f"f_rest_{y+1}"]
            b = gauss[f"f_rest_{y+2}"]
            l1_data.append([r, g, b])
            
        # Step 3: Extract L2 (y=9 to 23, step 3)
        for y in range(9, 24, 3):
            r = gauss[f"f_rest_{y}"]
            g = gauss[f"f_rest_{y+1}"]
            b = gauss[f"f_rest_{y+2}"]
            l2_data.append([r, g, b])
            
        # Step 4: Extract L31 (y=24 to 35, step 3)
        for y in range(24, 36, 3):
            r = gauss[f"f_rest_{y}"]
            g = gauss[f"f_rest_{y+1}"]
            b = gauss[f"f_rest_{y+2}"]
            l31_data.append([r, g, b])
            
        # Step 5: Extract L32 (y=36 to 44, step 3)
        for y in range(36, 45, 3):
            r = gauss[f"f_rest_{y}"]
            g = gauss[f"f_rest_{y+1}"]
            b = gauss[f"f_rest_{y+2}"]
            l32_data.append([r, g, b])

    return {
        'L1': l1_data,
        'L2': l2_data,
        'L31': l31_data,
        'L32': l32_data
    }
```

**Parameters:**
- `higherOrderHarmonicsExists`: Boolean flag. Default `true`. Set to `false` if any `f_rest_y` key (for $y=0, 3, \dots, 42$) is missing from the vertex data [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Missing Properties:** If the PLY file does not contain higher-order SH coefficients, the extraction must handle the absence gracefully by skipping the respective levels or returning empty arrays [S1, unofficial].
- **Index Bounds:** Ensure that reading `f_rest_y+1` and `f_rest_y+2` does not exceed the defined range of SH coefficients (up to 44). The loops are structured to prevent out-of-bounds access [S1, unofficial].
- **Precision:** Not given in the sources.

## Operating characteristics
- **Time Complexity:** $O(N \cdot D)$, where $N$ is the number of Gaussians and $D$ is the number of SH coefficients (45 for full SH3). The extraction is a linear scan of the vertex data [S1, unofficial].
- **Memory Complexity:** $O(N \cdot C)$, where $C$ is the number of coefficients per Gaussian (45 floats). The data is stored in four separate arrays per emitter [S1, unofficial].
- **Strengths:** Direct mapping from PLY property names to texture data simplifies the pipeline. Separation by level allows for flexible LOD management [S1, unofficial].
- **Limitations:** Requires the PLY file to have specific property naming conventions (`f_rest_y`). If the file format changes, the extraction logic must be updated [S1, unofficial].
- **When to Use:** When loading a 3D Gaussian Splatting model for rendering in an engine that uses texture-based SH storage [S1, unofficial].
- **When Not to Use:** If the model uses a different SH parameterization or if real-time extraction is not required (e.g., offline processing) [S1, unofficial].

## Background
The method originates from the implementation of 3D Gaussian Splatting in Unreal Engine, specifically the `unreal-splat` project [S1, unofficial]. It adapts the standard PLY format used by 3DGS to the texture-based SH representation required by the engine's renderer [S1, unofficial]. The separation into L1, L2, L31, and L32 levels reflects the engine's internal organization of SH coefficients for efficient sampling and rotation [S1, unofficial].

## Sources
[S1] JI20/unreal-splat, https://github.com/JI20/unreal-splat, fetched 2026-10-27 (unofficial)
