# Dataset Preprocessing (DeepLesion)
Also called: HU Normalization and Cropping

## Overview
This method prepares tomographic images from the DeepLesion dataset for training generative models by converting raw scanner data into standardized Hounsfield Units (HU), segmenting the patient anatomy from background noise, and normalizing spatial resolution [S1, unofficial]. It differs from generic [Volume Normalization](1823-volume-normalization.md) by using a specific anatomical threshold (-9050 HU) to define the region of interest rather than global statistical scaling, and from [Volume Utility Functions](2677-volume-utility-functions.md) by enforcing a strict minimum output dimension (256x256) [S1, unofficial].

## How it works
**Inputs:** Raw tomographic image data stored with 16-bit precision (DeepLesion format).
**Outputs:** A normalized 256x256 pixel image array in Hounsfield Units.

**Data Structures:**
- Input: 2D array of 16-bit integers representing raw scanner counts.
- Intermediate: Binary mask (boolean array) identifying voxels above the air threshold.
- Output: 2D array of floats or 16-bit integers representing HU values, resized to 256x256.

**Steps:**
1. **HU Conversion:** Convert raw pixel intensities to Hounsfield Units. The raw values are offset by 32768.
   $$ HU = \text{RawIntensity} - 32768 $$
   *Note:* In this scale, air has a density of -1000 HU and water has a density of 0 HU [S1, unofficial].

2. **Thresholding:** Create a binary mask to isolate the patient/anatomy from the background (air/scanner bore). Pixels with $HU > -9050$ are set to 1 (foreground), others to 0 (background). The threshold -9050 is chosen to be slightly higher than the density of air (-1000 HU) to exclude background noise while retaining the subject [S1, unofficial].

3. **Bounding Crop:** Find the smallest circle that contains all pixels where the binary mask is 1. Crop the original HU image to the smallest square region that contains this circle.

4. **Validation and Resizing:**
   - If the side length of the cropped square region is less than 256 pixels, discard the image.
   - If the side length is $\ge 256$ pixels, rescale (interpolate) the cropped region to exactly 256x256 pixels.
   - Add the resulting 256x256 image to the training dataset [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def preprocess_deeplesion(raw_image_16bit: np.ndarray) -> np.ndarray:
    """
    Preprocess a single DeepLesion tomographic slice.
    
    Args:
        raw_image_16bit: 2D numpy array of 16-bit integers.
        
    Returns:
        2D numpy array of shape (256, 256) with HU values.
        Returns None if the crop is too small.
    """
    # Step 1: Convert to Hounsfield Units
    # Raw values are offset by 32768
    hu_image = raw_image_16bit.astype(np.float32) - 32768.0
    
    # Step 2: Thresholding
    # Threshold is -9050 HU. Pixels > -9050 are foreground.
    binary_mask = hu_image > -9050.0
    
    # Step 3: Find smallest circle containing all binary pixels
    # Find coordinates of all foreground pixels
    coords = np.argwhere(binary_mask)
    
    if coords.size == 0:
        # No anatomy found, discard
        return None
        
    # Calculate center and radius of the smallest enclosing circle
    # For implementation simplicity, we approximate the bounding box 
    # or use a standard algorithm for minimum enclosing circle.
    # Here we assume a function `min_enclosing_circle` exists which returns (cx, cy, r).
    # To keep pseudocode self-contained without external libs, we use bounding box 
    # as a proxy for the square crop region, noting the source specifies "smallest circle".
    # A precise implementation would use Welzl's algorithm.
    
    # Approximation for pseudocode clarity: 
    # Find min/max row and col to define bounding box, then expand to square 
    # centered at the mean of the bounding box, sized to fit the diameter.
    rows, cols = coords[:, 0], coords[:, 1]
    min_r, max_r = np.min(rows), np.max(rows)
    min_c, max_c = np.min(cols), np.max(cols)
    
    # Center of the bounding box
    center_r = (min_r + max_r) / 2.0
    center_c = (min_c + max_c) / 2.0
    
    # Radius needed to cover the bounding box corners from center
    # Distance to corner
    half_h = (max_r - min_r) / 2.0
    half_w = (max_c - min_c) / 2.0
    radius = np.sqrt(half_h**2 + half_w**2)
    
    # Define square crop region based on the circle's diameter
    # The square must contain the circle, so side length = 2 * radius
    side_length = 2 * radius
    
    # Calculate crop boundaries
    r1 = int(center_r - radius)
    r2 = int(center_r + radius)
    c1 = int(center_c - radius)
    c2 = int(center_c + radius)
    
    # Ensure bounds are within image limits
    h, w = hu_image.shape
    r1 = max(0, r1)
    r2 = min(h, r2)
    c1 = max(0, c1)
    c2 = min(w, c2)
    
    cropped = hu_image[r1:r2, c1:c2]
    
    # Step 4: Validation and Resizing
    if cropped.shape[0] < 256 or cropped.shape[1] < 256:
        return None
        
    # Rescale to 256x256
    # Using bilinear interpolation as standard for medical image resizing
    resized = np.zeros((256, 256))
    resized = resize_image(cropped, (256, 256))
    
    return resized

def resize_image(image: np.ndarray, target_shape: tuple) -> np.ndarray:
    """
    Simple bilinear resize implementation for pseudocode completeness.
    """
    h, w = image.shape
    th, tw = target_shape
    
    # Calculate scaling factors
    scale_y = h / th
    scale_x = w / tw
    
    result = np.zeros((th, tw))
    
    for y in range(th):
        for x in range(tw):
            # Map target pixel to source coordinates
            src_y = y * scale_y
            src_x = x * scale_x
            
            # Get integer and fractional parts
            y0 = int(np.floor(src_y))
            x0 = int(np.floor(src_x))
            y1 = min(y0 + 1, h - 1)
            x1 = min(x0 + 1, w - 1)
            
            dy = src_y - y0
            dx = src_x - x0
            
            # Bilinear interpolation
            val = (1 - dy) * (1 - dx) * image[y0, x0] + \
                  (1 - dy) * dx * image[y0, x1] + \
                  dy * (1 - dx) * image[y1, x0] + \
                  dy * dx * image[y1, x1]
            result[y, x] = val
            
    return result
```

**Edge Cases and Pitfalls:**
- **Empty Anatomy:** If no pixels exceed -9050 HU, the mask is empty. The method discards the image [S1, unofficial].
- **Small Regions:** If the anatomical region is too small to fit a 256x256 crop, the image is discarded. This prevents training on insufficient context [S1, unofficial].
- **Precision:** The input is 16-bit, but HU values are continuous. Floating-point representation is required after subtraction [S1, unofficial].

## Subtopics

### Vannier Pre-processing for CT
A pre-processing method for craniofacial surgical planning that involves dividing CT slices into separate files, filtering them through a threshold value, and saving the data in CT format for use with interactive CAD systems [S2, unofficial]. It differs from DeepLesion preprocessing by focusing on file-level segmentation for surgical contouring rather than spatial normalization for deep learning training [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of pixels for thresholding and cropping. Resizing is $O(N)$ where $N$ is the output pixel count (256x256).
- **Memory Complexity:** Requires storage for the original image, the binary mask, and the cropped region.
- **Strengths:** Standardizes input data for neural networks; removes background noise effectively using anatomical knowledge (HU thresholds); ensures uniform input size for batch processing.
- **Limitations:** Discards images with small anatomical regions, potentially biasing the dataset towards larger pathologies or body parts; relies on accurate HU calibration of the source scanner.
- **When to use:** When training generative models on tomographic data where background noise interferes with learning anatomical structures.
- **When not to use:** When the full field of view is required for context, or when the dataset contains significant variations in scanner calibration that make a fixed HU threshold unreliable.

## Background
This method was introduced in the context of the DeepLesion dataset, a large collection of annotated tomographic images [S1, unofficial]. It was specifically designed to prepare data for a generative tomography reconstruction model that optimizes over a free parameter on a sphere, requiring differentiable forward and backward projections [S1, unofficial]. The preprocessing ensures that the input data is in a consistent physical unit (HU) and spatial resolution, facilitating the use of libraries like TorchRadon for gradient computation [S1, unofficial].

## Sources
[S1] Generative Tomography Reconstruction, https://core.ac.uk/works/91148923, fetched 2026-10-31 (unofficial)
[S2] An Overview of Rendering from Volume Data --- including Surface and Volume Rendering, https://core.ac.uk/download/63614.pdf, fetched 2026-10-30 (unofficial)
