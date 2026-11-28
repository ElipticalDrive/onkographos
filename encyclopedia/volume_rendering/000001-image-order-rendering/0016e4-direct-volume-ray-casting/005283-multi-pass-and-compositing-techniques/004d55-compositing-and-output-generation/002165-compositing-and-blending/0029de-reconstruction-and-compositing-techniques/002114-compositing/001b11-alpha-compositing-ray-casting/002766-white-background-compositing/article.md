# White Background Compositing
Also called: Alpha Blending over White
## Overview
White Background Compositing is a final-stage image processing step that converts an RGBA volume rendering into an RGB image by blending the rendered pixels over a fixed white background. It assumes the alpha channel represents opacity against a white backdrop, effectively simulating the visualization of the volume data on a white sheet or screen. It is a specific instance of general background blending, distinguished by its hardcoded background color of white ($1.0, 1.0, 1.0$), which simplifies the compositing equation and is standard for medical and scientific visualization where contrast against white is desired.
## How it works
**Inputs and Outputs:**
- Input: An RGBA image buffer where each pixel has components $(R_{in}, G_{in}, B_{in}, \alpha)$, with $\alpha \in [0, 1]$.
- Output: An RGB image buffer where each pixel has components $(R_{out}, G_{out}, B_{out})$.

**Data Structures:**
- The input is typically a 2D array or tensor of shape $(H, W, 4)$, where $H$ is height, $W$ is width, and the last dimension corresponds to RGBA channels.
- The output is a 2D array or tensor of shape $(H, W, 3)$, corresponding to RGB channels.

**Formula:**
For each pixel at position $(x, y)$, the output RGB values are computed as:
$$
C_{out} = C_{in} \cdot \alpha + (1 - \alpha) \cdot C_{bg}
$$
where:
- $C_{out}$ is the output color vector $(R_{out}, G_{out}, B_{out})$.
- $C_{in}$ is the input color vector $(R_{in}, G_{in}, B_{in})$.
- $\alpha$ is the input alpha value at $(x, y)$.
- $C_{bg}$ is the background color vector. For White Background Compositing, $C_{bg} = (1.0, 1.0, 1.0)$.

Thus, the specific equations are:
$$
R_{out} = R_{in} \cdot \alpha + (1 - \alpha) \cdot 1.0
$$
$$
G_{out} = G_{in} \cdot \alpha + (1 - \alpha) \cdot 1.0
$$
$$
B_{out} = B_{in} \cdot \alpha + (1 - \alpha) \cdot 1.0
$$

**Parameters:**
- Background Color: Fixed at $(1.0, 1.0, 1.0)$ (white).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def white_background_compositing(rgba_image: np.ndarray) -> np.ndarray:
    """
    Composites an RGBA image over a white background.
    
    Args:
        rgba_image: A numpy array of shape (H, W, 4) with dtype float32 or float64.
                    Values are expected to be in [0, 1].
    
    Returns:
        A numpy array of shape (H, W, 3) with dtype float32 or float64.
        Values are in [0, 1].
    """
    # Extract RGB and Alpha channels
    # Assuming input is (H, W, 4)
    rgb_in = rgba_image[..., :3]
    alpha = rgba_image[..., 3:4]  # Keep dimensions for broadcasting
    
    # Background color is white (1.0, 1.0, 1.0)
    background_color = 1.0
    
    # Apply compositing formula: C_out = C_in * alpha + (1 - alpha) * C_bg
    rgb_out = rgb_in * alpha + (1.0 - alpha) * background_color
    
    return rgb_out
```

**Edge Cases and Pitfalls:**
- **Alpha Range:** The method assumes $\alpha \in [0, 1]$. If $\alpha$ is outside this range, the result may be clipped or exhibit unexpected behavior. [G]
- **Precision:** Floating-point precision issues may arise if $\alpha$ is very close to 0 or 1, but standard float32 is generally sufficient for visual applications.
- **Input Format:** The input must be in premultiplied or straight alpha? The formula assumes straight alpha (non-premultiplied) input. If the input is premultiplied, the formula would differ. [G] Standard volume rendering outputs straight alpha unless specified otherwise.
- **Background Assumption:** This method is only appropriate when the intended background is white. For other backgrounds, use [General Background Blending](18469-general-background-blending.md).

## Subtopics

### [General Background Blending](18469-general-background-blending.md)
General Background Blending composites a rendered RGBA image over an arbitrary background color to produce a final RGB image. Unlike [White Background Compositing](4015-white-background-compositing.md), which assumes a fixed white backdrop, this method allows the background to be specified dynamically, enabling the visualization of volume data against non-white environments or specific scene contexts. It serves as the final compositing step in many ray-casting pipelines where the alpha channel represents opacity against the chosen background.

## Operating characteristics
- **Time Complexity:** $O(H \times W)$, where $H$ and $W$ are the image dimensions. The operation is a simple per-pixel arithmetic operation.
- **Memory Complexity:** $O(H \times W)$ for the output buffer. The input buffer is read-only.
- **Performance:** Very fast, as it involves only basic arithmetic operations and can be easily parallelized on GPUs or vectorized on CPUs.
- **Strengths:** Simple to implement, computationally inexpensive, and provides a standard visualization for volume data on white backgrounds.
- **Limitations:** Only suitable for white backgrounds. Not flexible for arbitrary background colors or textures.
- **When to Use:** When the volume rendering needs to be displayed on a white background, such as in medical imaging or scientific visualization.
- **When Not to Use:** When the background is not white, or when transparency effects against a non-white background are required.

## Background
White Background Compositing is a common technique in volume rendering and image processing, particularly in fields like medical imaging and scientific visualization. It is a specific case of alpha compositing, which was formalized by Thomas Porter and Tom Duff in 1984 [S1, unofficial]. The method is widely used in tools like VTK, ParaView, and various Python libraries for visualizing 3D data.

## Sources
[S1] yashtmy/nerf-3d-vision, https://github.com/yashtmy/nerf-3d-vision, fetched 2026-10-26 (unofficial)
