# Box Smoothing for Level-Set Extraction
Also called: Density Field Smoothing, Field Blur
## Overview
Box Smoothing for Level-Set Extraction applies a box blur filter to a binary density field to convert sharp 0/1 steps into smooth transitions, allowing Marching Cubes to generate rounded geometry instead of sharp stair-stepped edges [S1, unofficial]. It differs from its parent, Gaussian Smoothing for Level-Set Extraction, by using a simpler averaging kernel that is computationally cheaper but may preserve less of the original field's high-frequency detail [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: A 3D density array `density` of size $W \times H \times D$, containing binary values (0 or 1) [S1, unofficial].
- Output: The same `density` array, modified in-place (via buffer swapping) to contain smoothed values [S1, unofficial].

**Data Structures:**
- `density`: A 1D array of floats of size $W \times H \times D$, with coordinate mapping $i = x + y \cdot W + z \cdot W \cdot H$ [S1, unofficial].
- `buffer`: A temporary array of the same size and type as `density` [S1, unofficial].

**Parameters:**
- `iterations`: The number of times the blur pass is repeated. Not given in the sources.

**Steps:**
1. Allocate a temporary buffer `buffer` of the same size as `density` [S1, unofficial].
2. Repeat the following for `iterations` times:
    1. For each voxel $(x, y, z)$ in the volume:
        1. If $(x, y, z)$ is a border voxel (i.e., $x=0$, $x=W-1$, $y=0$, $y=H-1$, $z=0$, or $z=D-1$):
            1. Set `buffer[x, y, z] = 1.0` [S1, unofficial].
            2. Continue to the next voxel.
        2. If $(x, y, z)$ is a non-border voxel:
            1. Calculate the sum of the current voxel's density and its 6 immediate face-neighbors:
               $$ d_{new}(x,y,z) = \frac{1}{7} \left( d(x,y,z) + d(x+1,y,z) + d(x-1,y,z) + d(x,y+1,z) + d(x,y-1,z) + d(x,y,z+1) + d(x,y,z-1) \right) $$
            2. Set `buffer[x, y, z] = d_{new}(x,y,z)` [S1, unofficial].
    2. Swap the `density` and `buffer` arrays for the next iteration [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def box_smooth(density, width, height, depth, iterations):
    buffer = [0.0] * (width * height * depth)
    for _ in range(iterations):
        for z in range(depth):
            for y in range(height):
                for x in range(width):
                    if x == 0 or x == width - 1 or y == 0 or y == height - 1 or z == 0 or z == depth - 1:
                        buffer[z * width * height + y * width + x] = 1.0
                    else:
                        idx = z * width * height + y * width + x
                        neighbor_sum = (density[idx] + density[idx + 1] + density[idx - 1] +
                                        density[idx + width] + density[idx - width] +
                                        density[idx + width * height] + density[idx - width * height])
                        buffer[idx] = neighbor_sum / 7.0
        density, buffer = buffer, density
```

**Edge Cases and Pitfalls:**
- Border voxels are hardcoded to 1.0, which may cause boundary artifacts if the field does not naturally reach 1.0 at the edges [S1, unofficial].
- The method assumes a binary input (0 or 1) [S1, unofficial].
- Precision issues are not explicitly discussed, but floating-point division by 7.0 is used [S1, unofficial].

## Subtopics

### Field Smoothing Pass
An optional post-generation blur pass applied to the density field to round off sharp relief edges [S1, unofficial]. Unlike mesh-based smoothing, this method does not directly average or displace vertices of an already extracted mesh; it filters the scalar field first, allowing Marching Cubes to generate rounded geometry instead of sharp stair-stepped edges [S1, unofficial]. It builds on Box Smoothing for Level-Set Extraction [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- Time: $O(iterations \cdot W \cdot H \cdot D)$, where $W, H, D$ are the dimensions of the volume. Each iteration processes every voxel once [S1, unofficial].
- Memory: $O(W \cdot H \cdot D)$ for the temporary buffer [S1, unofficial].

**Measured Performance:**
- Not given in the sources.

**Scaling:**
- Not given in the sources.

**Strengths and Limitations:**
- Strengths: Simple to implement, computationally efficient compared to Gaussian smoothing, effective at removing stair-stepping artifacts from binary volumes [S1, unofficial].
- Limitations: May destroy fine details, border handling is simplistic (hardcoded to 1.0), only considers face-neighbors (not edge or corner neighbors) [S1, unofficial].

**When to Use:**
- Use when a fast, simple smoothing method is needed for binary density fields before Marching Cubes extraction [S1, unofficial].

**When Not to Use:**
- Do not use when preserving fine details is critical [S1, unofficial].

## Background
Origin: The method is described in the context of a Unity voxel terrain implementation [S1, unofficial].
History and Development: The specific implementation details are found in the `TerrainChunk.cs` script of the `UnityVoxelTerrain` repository [S1, unofficial].
Where it is used: It is used in voxel-based terrain generation to smooth the density field before surface extraction [S1, unofficial].

## Sources
[S1] Rabidus4k/UnityVoxelTerrain, https://github.com/Rabidus4k/UnityVoxelTerrain, fetched 2026-10-28 (unofficial)
