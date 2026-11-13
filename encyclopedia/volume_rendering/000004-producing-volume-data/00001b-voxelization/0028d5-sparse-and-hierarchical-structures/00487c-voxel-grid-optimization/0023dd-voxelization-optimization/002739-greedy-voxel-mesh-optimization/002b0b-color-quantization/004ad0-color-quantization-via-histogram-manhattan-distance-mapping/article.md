# Color Quantization via Histogram & Manhattan Distance Mapping

Also called: Frequency-based color mapping, Palette reduction

## Overview
This method reduces the number of unique colors in a voxel grid to a specified limit by selecting the most frequent colors as a palette and mapping all other colors to the nearest palette entry using Manhattan distance [S1, unofficial]. It differs from median-cut or k-means approaches by prioritizing frequency-based palette selection and using a computationally cheaper distance metric, making it suitable for real-time or resource-constrained voxelization pipelines [S1, unofficial].

## How it works
**Inputs:**
- A list of voxels, each containing an RGB color value.
- An integer `colorLimit` specifying the maximum number of colors in the output palette.

**Outputs:**
- A modified list of voxels where each voxel's color is snapped to one of the `colorLimit` most frequent original colors.

**Data Structures:**
- `Histogram`: A dictionary mapping `Color` (RGB tuple) to `int` (frequency count).
- `Palette`: A list of `Color` objects, size at most `colorLimit`.
- `ColorMap`: A dictionary mapping original `Color` to mapped `Color` (palette entry).

**Steps:**
1. **Handle Edge Cases:** If the voxel list is empty or `colorLimit` is effectively disabled (e.g., 0 or negative), return the original voxel list unchanged [S1, unofficial].
2. **Build Histogram:** Iterate through all voxels. For each voxel, increment the count for its color in the `Histogram` dictionary. This step is $O(N)$, where $N$ is the number of voxels [S1, unofficial].
3. **Sort and Select Palette:** Sort the entries in the `Histogram` by frequency in descending order. Select the top `colorLimit` colors from this sorted list to form the `Palette`. If the number of unique colors $C$ is less than `colorLimit`, the palette contains all unique colors [S1, unofficial].
4. **Enforce Hard Cap:** The implementation enforces a hard cap of 256 on `colorLimit`. Specifically, `colorLimit = Math.Min(colorLimit, 256)` [S1, unofficial].
5. **Build Color Map:** For each unique color $c_{orig}$ in the `Histogram` (iterating through the sorted list or all unique keys):
   - Compute the Manhattan distance to every color $c_{pal}$ in the `Palette`.
   - The Manhattan distance is defined as:
     $$d(c_1, c_2) = |R_1 - R_2| + |G_1 - G_2| + |B_1 - B_2|$$
     where $R, G, B$ are the red, green, and blue channel values of colors $c_1$ and $c_2$ [S1, unofficial].
   - Find the palette color $c_{best}$ that minimizes this distance.
   - Store the mapping $c_{orig} \rightarrow c_{best}$ in the `ColorMap` dictionary.
6. **Map Voxels:** Iterate through the original voxel list. For each voxel, replace its color with the mapped color retrieved from the `ColorMap` dictionary [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def color_quantize_histogram_manhattan(voxels, color_limit):
    # Edge case: empty list or disabled quantization
    if not voxels or color_limit <= 0:
        return voxels

    # Enforce hard cap of 256
    color_limit = min(color_limit, 256)

    # Step 1: Build frequency histogram
    histogram = {}
    for voxel in voxels:
        color = voxel.color
        if color in histogram:
            histogram[color] += 1
        else:
            histogram[color] = 1

    # Step 2: Sort by frequency descending
    # Sort keys by their frequency value
    sorted_colors = sorted(histogram.keys(), key=lambda c: histogram[c], reverse=True)

    # Step 3: Select top colors as palette
    palette = sorted_colors[:color_limit]

    # Step 4: Build color map
    color_map = {}
    for orig_color in histogram.keys():
        best_pal_color = None
        min_dist = float('inf')
        
        for pal_color in palette:
            # Manhattan distance
            dist = abs(orig_color.r - pal_color.r) + \
                   abs(orig_color.g - pal_color.g) + \
                   abs(orig_color.b - pal_color.b)
            
            if dist < min_dist:
                min_dist = dist
                best_pal_color = pal_color
        
        color_map[orig_color] = best_pal_color

    # Step 5: Map voxels
    quantized_voxels = []
    for voxel in voxels:
        new_voxel = voxel.copy()
        new_voxel.color = color_map[voxel.color]
        quantized_voxels.append(new_voxel)

    return quantized_voxels
```

**Parameters:**
- `colorLimit`: Maximum number of colors allowed in the output palette. Typical values depend on the desired visual fidelity vs. memory savings. The implementation enforces a hard cap of 256 [S1, unofficial].

**Pitfalls:**
- Empty voxel lists or disabled quantization bypass the algorithm entirely and return the original list unchanged [S1, unofficial].
- Boundary checks are implicit since the algorithm only processes existing voxels.
- The Manhattan distance metric may not perceptually match human vision as well as Euclidean distance in LAB space, potentially leading to suboptimal color choices for certain palettes [S1, unofficial].

## Operating characteristics
**Time Complexity:** $O(N + C \log C + N \cdot M)$, where $N$ is the number of voxels, $C$ is the number of unique colors, and $M$ is `colorLimit` [S1, unofficial].
- Histogram construction: $O(N)$ [S1, unofficial].
- Sorting unique colors: $O(C \log C)$ [S1, unofficial].
- Mapping: $O(N \cdot M)$ [S1, unofficial].

**Memory Complexity:** $O(C + N)$ for storing the histogram, palette, color map, and output voxels [S1, unofficial].

**Strengths:**
- Simple to implement.
- Fast for small palettes.
- Preserves the most common colors exactly.

**Limitations:**
- $O(N \cdot M)$ mapping can be slow for large $N$ and large $M$.
- Manhattan distance is not perceptually uniform.
- Does not consider spatial distribution of colors.

**When to use:**
- When computational resources are limited.
- When the most frequent colors are the most important.
- When a small palette size is required.

**When not to use:**
- When perceptual accuracy is critical.
- When the number of unique colors is very large and $M$ is also large.

## Background
This method is used in the FileToVox tool for converting 3D models to the .vox format [S1, unofficial]. It is an unofficial implementation detail of the FileToVox project [S1, unofficial].

## Sources
[S1] FileToVox, https://github.com/Zarbuz/FileToVox, fetched 2026-10-29 (unofficial)
