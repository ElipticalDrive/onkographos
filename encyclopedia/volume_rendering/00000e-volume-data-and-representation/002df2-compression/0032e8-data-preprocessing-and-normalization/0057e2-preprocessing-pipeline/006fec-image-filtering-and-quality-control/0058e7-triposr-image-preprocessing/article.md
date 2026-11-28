# TripoSR Image Preprocessing
Also called: Sim2Real on Real Car Images Preprocessing
## Overview
This method prepares single-view images for the TripoSR 3D reconstruction model by removing backgrounds, centering the subject, and converting the image to a normalized 50% gray background format [S2, unofficial]. It solves the problem of inconsistent object scales and distracting backgrounds that degrade reconstruction accuracy [S1]. It differs from standard LDR preprocessing by explicitly compositing the subject over a fixed gray background rather than preserving transparency or white backgrounds [S2, unofficial].
## How it works
**Inputs and Outputs:**
- Input: A single RGB or RGBA image file.
- Output: A uint8 RGBA image with the subject centered on a 50% gray background, normalized to $[0, 1]$ during processing.

**Data Structures:**
- `image`: PIL Image object.
- `rembg_session`: Session object from the `rembg` library for background removal.
- `foreground_ratio`: Float, default 0.85.
- `target_size`: Integer, default 256 pixels, derived from the 1/4 side length constraint.

**Steps:**
1. **Background Removal:** Apply PointRend to images to obtain foreground masks and bounding boxes [S1]. Initialize a `rembg` session and remove the background using the mask [S2, unofficial].
2. **Foreground Resizing & Centering:** Translate and rescale the image so the bounding box center aligns with the image center, and the shorter bounding box side equals 1/4 of the target image side length (128 pixels) [S1]. Resize the foreground to a target ratio (default 0.85) relative to the image dimensions [S2, unofficial].
3. **Normalization:** Convert the image array to `float32` and normalize pixel values to $[0, 1]$ by dividing by 255.0 [S2, unofficial].
4. **Compositing:** Composite the RGB channels over a 50% gray background using the alpha channel [S2, unofficial]. The formula is:
   $$I_{out} = I_{rgb} \cdot \alpha + (1 - \alpha) \cdot 0.5$$
   where $I_{rgb}$ is the RGB channel, $\alpha$ is the alpha channel, and $0.5$ represents the 50% gray background [S2, unofficial].
5. **Final Conversion:** Convert the resulting float32 image back to `uint8` by multiplying by 255.0 and casting [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from PIL import Image
from rembg import new_session, remove

def preprocess_for_triposr(image: Image.Image, foreground_ratio: float = 0.85, target_size: int = 256) -> Image.Image:
    """
    Preprocess an image for TripoSR model input.
    
    Args:
        image: Input PIL Image.
        foreground_ratio: Ratio of foreground to image size. Default 0.85.
        target_size: Target canvas size. Default 256.
        
    Returns:
        Preprocessed PIL Image with 50% gray background.
    """
    # Step 1: Remove background
    # Apply PointRend to obtain foreground masks and bounding boxes [S1].
    session = new_session()
    image = remove(image, session=session)
    
    # Step 2: Resize foreground and center
    w, h = image.size
    # Scale so shorter side of bounding box is 1/4 of target size (128) [S1].
    scale = (target_size / 4) / min(w, h)
    new_w, new_h = int(w * scale), int(h * scale)
    image = image.resize((new_w, new_h), Image.LANCZOS)
    
    # Center on target canvas with 50% gray background
    gray_val = int(0.5 * 255)
    canvas = Image.new("RGBA", (target_size, target_size), (gray_val, gray_val, gray_val, 255))
    offset_x = (target_size - new_w) // 2
    offset_y = (target_size - new_h) // 2
    canvas.paste(image, (offset_x, offset_y), image)
    
    # Step 3: Convert to float32 and normalize to [0, 1]
    img_array = np.array(canvas).astype(np.float32) / 255.0
    
    # Step 4: Composite RGB over 50% gray background
    rgb = img_array[:, :, :3]
    alpha = img_array[:, :, 3:4]
    gray_bg = 0.5
    composite = rgb * alpha + (1 - alpha) * gray_bg
    
    # Step 5: Convert back to uint8
    result = Image.fromarray((composite * 255.0).astype(np.uint8))
    
    return result
```

**Parameters:**
- `foreground_ratio`: Default 0.85 [S2, unofficial].
- `target_size`: Default 256 pixels, derived from the 1/4 side length constraint [S1].

**Edge Cases and Pitfalls:**
- Precision: Use `float32` during normalization to avoid quantization artifacts [S2, unofficial].
- Boundaries: Handle images with no significant subject by returning as-is if the subject mask sum is below a threshold (e.g., 100 pixels) [S2, unofficial].
## Subtopics

### [HDR Image Preprocessing Pipeline](19206-hdr-image-preprocessing-pipeline.md)
This pipeline converts a series of bracketed Low Dynamic Range (LDR) exposures into a normalized 32-bit High Dynamic Range (HDR) radiance image suitable for training Neural Radiance Fields (NeRF) and diffusion-based inverse rendering models.

## Operating characteristics
- Time Complexity: Not given in the sources. [G]
- Memory Complexity: Not given in the sources. [G]
- Measured Performance: Not given in the sources. [G]
- Scaling: Not given in the sources. [G]
- Strengths: Removes distracting backgrounds and normalizes object scale, improving 3D reconstruction quality [S1].
- Limitations: Relies on `rembg` for background removal, which may fail on complex backgrounds [S2, unofficial].
- When to use: When preparing images for TripoSR model input [S2, unofficial].
- When not to use: When preserving original background is required [S1].
## Background
The TripoSR Image Preprocessing method is part of the image-to-voxel pipeline developed by Mohamed Mansour [S2, unofficial]. It is designed to preprocess images for the TripoSR model, which is used for 3D reconstruction from single images [S1].
## Sources
[S1] pixelNeRF: Neural Radiance Fields from One or Few Images, https://doi.org/10.1109/cvpr46437.2021.00455, fetched 2026-10-26
[S2] mohamedmansour/image-to-voxel, https://github.com/mohamedmansour/image-to-voxel, fetched 2026-10-26 (unofficial)
