# GPU Implementation of Style Transfer Function Lookup

Also called: Three-Texture Style Lookup

## Overview
This method implements the Style Transfer Function (1D) on GPU hardware by decomposing the mapping from data values and surface normals to color and opacity into three separate textures [S1]. It solves the memory cost of using a single 3D texture for style interpolation by exploiting native trilinear interpolation without the high storage requirements [S1]. It differs from [Mip-Mapping for Style Functions](782-mip-mapping-for-style-functions.md) by decoupling the index mapping from style storage to avoid mipmapping artifacts, and from [Multi-dimensional Style Transfer Function (2D)](813-multi-dimensional-style-transfer-function-2d.md) by restricting the domain to 1D data values [G].

## How it works
**Inputs:**
- Sample data value $s$ [S1].
- Eye-space surface normal components $n_x, n_y$ [S1].

**Outputs:**
- Final color $(r, g, b)$ and derived opacity $\alpha_d$ [S1].
- Uniform opacity $\alpha_u$ [S1].

**Data Structures:**
1. **Transfer Function Texture (tft):** A 1D texture storing $(i, \alpha_u)$ tuples for each data value [S1].
   - $i$: Index value ranging from $0$ to $N-1$, where $N$ is the number of styles specified in the style transfer function [S1]. Fractional values indicate interpolation between two discrete styles [S1].
   - $\alpha_u$: Uniform opacity value [S1].
2. **Index Function Texture (ift):** A 1D texture mapping index values $i$ (range $0$ to $N-1$) to style locations $j$ (range $0$ to $M-1$) in the style function texture [S1].
   - $M$: Number of distinct styles in the style transfer function [S1].
   - This mapping is discrete; no interpolation is performed during lookup [S1].
3. **Style Function Texture (sft):** Stores $M$ distinct styles, each being a 2D image (lit sphere map) [S1].
   - Stored as a 3D texture or an alternative format to avoid mipmapping problems [S1].
   - Lookup coordinates are $(n_x, n_y, j)$, where $j$ is the style index [S1].

**Parameters:**
- $N$: Number of styles specified in the style transfer function (range of index values in `tft`) [S1].
- $M$: Number of distinct styles in the style transfer function (range of locations in `sft`) [S1].

**Algorithm Steps:**
1. **Retrieve Index and Opacity:** Lookup $(i, \alpha_u) = \text{tft}(s)$ using the sample value $s$ [S1].
2. **Compute Interpolation Weights:** Calculate integer indices $i_0 = \lfloor i \rfloor$, $i_1 = i_0 + 1$, and the interpolation weight $w = i - i_0$ [S1].
3. **Retrieve Style Indices:** Lookup style indices $j_0 = \text{ift}(i_0)$ and $j_1 = \text{ift}(i_1)$ [S1]. If no style is used multiple times in the style transfer function, these lookups can be skipped [S1].
4. **Interpolate Style:** Perform lookups into the style function texture using the normal components $n_x, n_y$ and the style indices $j_0, j_1$, then linearly interpolate between them: $(r, g, b, \alpha_d) = \text{sft}(n_x, n_y, j_0) \cdot (1 - w) + \text{sft}(n_x, n_y, j_1) \cdot w$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def style_transfer_lookup(s, nx, ny, tft, ift, sft, N, M):
    # Step 1: Retrieve index i and uniform opacity alpha_u from tft
    i, alpha_u = tft[s]
    
    # Step 2: Compute interpolation weights
    i0 = math.floor(i)
    i1 = i0 + 1
    w = i - i0
    
    # Step 3: Retrieve style indices j0 and j1 from ift
    j0 = ift[i0]
    j1 = ift[i1]
    
    # Step 4: Interpolate style colors
    c0 = sft[nx, ny, j0]
    c1 = sft[nx, ny, j1]
    
    r = c0[0] * (1.0 - w) + c1[0] * w
    g = c0[1] * (1.0 - w) + c1[1] * w
    b = c0[2] * (1.0 - w) + c1[2] * w
    alpha_d = c0[3] * (1.0 - w) + c1[3] * w
    
    return r, g, b, alpha_d, alpha_u
```

**Edge Cases and Pitfalls:**
- If no style is used multiple times, the `ift` lookups are redundant and can be skipped [S1].
- Using a 3D texture for `sft` can cause mipmapping artifacts; an alternative storage format may be required [S1].

## Operating characteristics
- **Memory Complexity:** The method uses three separate textures instead of a single 3D texture, significantly reducing storage requirements [S1].
- **Performance:** Requires one lookup into `tft`, up to two into `ift`, and two into `sft` per sample [S1].
- **Strengths:** Avoids high storage requirements of 3D textures and allows efficient interpolation between discrete styles [S1].
- **Limitations:** Requires careful handling of `sft` storage to avoid mipmapping issues [S1].

## Background
This method was introduced for illustrative volume rendering to provide artists with control over visual appearance while maintaining memory efficiency [S1]. It extends 1D transfer functions by incorporating surface normals to drive style variations, addressing the impracticality of using a single 3D texture for style interpolation [S1].

## Sources
[S1] Style Transfer Functions for Illustrative Volume Rendering, https://doi.org/10.1111/j.1467-8659.2007.01095.x, fetched 2026-10-26
