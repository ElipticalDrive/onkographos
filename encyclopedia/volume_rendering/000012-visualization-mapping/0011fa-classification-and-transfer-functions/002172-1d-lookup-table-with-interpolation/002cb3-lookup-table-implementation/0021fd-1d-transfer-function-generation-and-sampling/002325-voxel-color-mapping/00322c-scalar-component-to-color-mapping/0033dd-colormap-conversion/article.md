# Colormap Conversion

Also called: Ci Calculation, Data to Color Conversion, Linear Color Scaling, Matplotlib Colormap Indexing

## Overview
Colormap Conversion maps scalar data values to RGB colors by linearly scaling the input to a normalized index and indexing into a pre-defined color table [S1, S2, unofficial]. It solves the problem of visualizing scalar magnitude (e.g., temperature) through color [S1, unofficial]. It applies a fixed, global linear normalization over the dataset bounds to index a pre-defined color table [S1, S2, unofficial].

## How it works
**Inputs:**
- Scalar tensor $V$ of shape $(H, W)$ or $(D, H, W)$ containing data values [S2, unofficial].
- Data bounds $V_{min}$ and $V_{max}$ (minimum and maximum values in $V$) [S1, unofficial].
- Colormap name string `cmap` (default: 'turbo') [S2, unofficial].

**Outputs:**
- RGB tensor of shape $(3, H, W)$ with values in $[0, 1]$ [S2, unofficial].

**Data Structures:**
- `colors`: A tensor of shape $(256, 3)$ containing the RGB values for the colormap, loaded from the library [S2, unofficial].

**Steps:**
1. Load the colormap colors into a tensor `colors` of shape $(256, 3)$ on the target device [S2, unofficial].
2. Normalize the input data $V_i$ to the range $[0, 1]$ using the global bounds:
   $$ V_{norm} = \frac{V_i - V_{min}}{V_{max} - V_{min}} $$
   [S1, unofficial].
3. Scale the normalized value to the integer index range $[0, 255]$:
   $$ C_i = V_{norm} \times 255 $$
   [S1, unofficial].
4. Round $C_i$ to the nearest integer and convert to a long integer type [S2, unofficial].
5. Squeeze dimensions of the index tensor to remove singleton dimensions [S2, unofficial].
6. Index into the `colors` tensor using the integer indices [S2, unofficial].
7. Permute the resulting tensor dimensions to $(2, 0, 1)$ to yield $(3, H, W)$ (RGB, Height, Width) [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import matplotlib.cm as cm
import numpy as np

def colormap_conversion(V: torch.Tensor, cmap_name: str = 'turbo') -> torch.Tensor:
    # 1. Load colormap colors
    cmap = cm.get_cmap(cmap_name)
    # Generate 256 colors, take RGB channels
    colors = torch.tensor(cmap(np.linspace(0, 1, 256))[:, :3], dtype=torch.float32)
    colors = colors.to(V.device)
    
    # 2. Normalize input
    V_min = V.min()
    V_max = V.max()
    if V_max == V_min:
        V_norm = torch.zeros_like(V)
    else:
        V_norm = (V - V_min) / (V_max - V_min)
        
    # 3. Scale to 0-255
    C_i = V_norm * 255
    
    # 4. Round and convert to long integers
    indices = torch.round(C_i).long()
    
    # 5. Squeeze dimensions
    indices = torch.squeeze(indices)
    
    # 6. Index into colors tensor
    # Advanced indexing: colors[indices] produces shape (..., 3)
    rgb = colors[indices]
    
    # 7. Permute to (3, H, W)
    rgb = rgb.permute(2, 0, 1)
    
    return rgb
```

**Parameters:**
- `cmap`: Colormap name string. Default: 'turbo' [S2, unofficial].
- $V_{min}, V_{max}$: Data bounds used for normalization [S1, unofficial].

**Edge Cases and Pitfalls:**
- If $V_{min} == V_{max}$, division by zero occurs. The implementation must handle this by setting all normalized values to 0 [G].
- Precision errors may cause $V_i$ to fall slightly outside $[V_{min}, V_{max}]$, leading to indices $<0$ or $>255$. Clamping indices to $[0, 255]$ is required [G].
- The method assumes a linear relationship between data magnitude and color index [S1, unofficial].

## Subtopics

### [Map to Colormap](18265-map-to-colormap.md)
`map_to_colormap` maps a continuous colormap directly to a specified range of scalar values (typically density), assigning a single constant alpha value to all values within that range. Unlike layered transfer function methods that distribute opacity across different scalar bands, this method allows all density values in the dataset to contribute to the volume rendering simultaneously at a uniform visibility level. It is primarily used for projection-like visualizations where the goal is to display the full dynamic range of a field without layering effects.

### Depth to Colormap Conversion
This variant converts depth buffers to color maps using OpenCV's colormap utilities [S3, unofficial]. It identifies valid depth pixels where depth $> -0.9$ [S3, unofficial]. It normalizes valid depth values to $[0, 1]$ using min-max scaling: $(d_{valid} - d_{valid.min()}) / (d_{valid.max()} - d_{valid.min()})$ [S3, unofficial]. It scales normalized depth to $[0, 255]$, converts to uint8, and applies `cv2.COLORMAP_JET` [S3, unofficial]. If no valid depth pixels exist (`valid_mask.sum() == 0`), it returns zero arrays and prints a warning [S3, unofficial].

## Operating characteristics
Linear time $O(N)$ and memory $O(N)$ for the output tensor [G]. The method is simple, fast, and deterministic, and is easy to implement on GPU using tensor operations [S2, unofficial]. Linear scaling may not highlight subtle variations in data if the distribution is non-uniform [G]. It is suitable when visualizing scalar magnitude from simulation results, such as temperature in fire dynamics [S1, unofficial]. It should not be used when per-voxel or piecewise transfer functions are required for better contrast or feature isolation [G].

## Background
The method was described in the Smokeview Verification Guide, part of the Fire Dynamics Simulation (FDS) toolset [S1, unofficial]. It uses linear scaling to convert temperature data (e.g., 20°C to 100°C) into colors (blue to red) for visualization [S1, unofficial]. The implementation details using matplotlib colormaps were also observed in the 2D Gaussian Splatting repository [S2, unofficial].

## Sources
[S1] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume III : Veriﬁcation Guide, https://doi.org/10.6028/nist.sp.1017-3, fetched 2026-10-26 (unofficial)
[S2] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
[S3] GAP-LAB-CUHK-SZ/gaustudio, https://github.com/GAP-LAB-CUHK-SZ/gaustudio, fetched 2026-10-26 (unofficial)
