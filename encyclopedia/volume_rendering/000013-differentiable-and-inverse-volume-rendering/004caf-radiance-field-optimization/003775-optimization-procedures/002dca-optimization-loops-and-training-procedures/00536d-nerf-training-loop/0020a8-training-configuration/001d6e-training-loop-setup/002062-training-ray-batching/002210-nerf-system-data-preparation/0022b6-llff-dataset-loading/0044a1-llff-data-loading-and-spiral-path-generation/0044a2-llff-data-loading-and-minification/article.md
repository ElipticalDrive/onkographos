# LLFF Data Loading and Minification

Also called: LLFF Data Loading with ImageMagick Minification, LLFF Dataset Loading and Minification, LLFF Dataset Preparation

## Overview
This method prepares LLFF-style datasets for NeRF training by loading poses and images, applying geometric normalization, and downsampling image resolution to reduce computational cost. It differs from standard loading by explicitly handling the minification step via external tools and enforcing a specific train/test split protocol (every 8th image for testing) [S1][S2, unofficial].

## How it works
**Inputs:**
- Directory containing LLFF data: `poses_bounds.npy`, `train.txt`, `test.txt`, and raw images (`.jpg` or `.png`).
- `factor`: Downscale factor for images (default: 8) [S2, unofficial].
- `target_size`: Optional target resolution (e.g., 378×504) [S1].

**Outputs:**
- `images`: Normalized tensor of shape `[N, H', W', 3]` with values in `[0, 1]` [S2, unofficial].
- `poses`: Camera poses matrix of shape `[N, 3, 4]` (or `[3, 5, N]` intermediate) [S2, unofficial].
- `bounds`: Near/far depth bounds of shape `[2, N]` [S2, unofficial].
- `train_indices`, `test_indices`: Lists of image indices for training and testing [S1].

**Data Structures:**
- `poses_bounds.npy`: Loaded as a NumPy array. Reshaped to `poses` `[3, 5, N]` and `bounds` `[2, N]` [S2, unofficial].
- `train.txt`/`test.txt`: Text files listing image filenames. Filenames are adjusted to `.png` if necessary [S2, unofficial].

**Steps:**
1. **Load Metadata:** Read `train.txt` and `test.txt` to get image file lists. Adjust extensions to `.png` if the source files are `.jpg` [S2, unofficial].
2. **Minification:** If `factor > 1` or `target_size` is specified:
   - Create a new directory for minified images.
   - Use ImageMagick `mogrify` to resize images by `factor` or to `target_size` (e.g., 1/8th original size, resulting in 378×504 for standard LLFF) [S1][S2, unofficial].
   - Update file lists to point to the minified directory [S2, unofficial].
3. **Load Poses and Bounds:** Load `poses_bounds.npy`. Reshape to extract:
   - `poses`: `[3, 5, N]` (first 3 rows are rotation, last 2 are translation/bounds info) [S2, unofficial].
   - `bounds`: `[2, N]` (near and far depth values) [S2, unofficial].
4. **Split Data:**
   - Use every 8th image as the test set [S1].
   - Sample training views evenly from the remaining images [S1].
5. **Load and Normalize Images:**
   - Load minified images.
   - Normalize pixel values to `[0, 1]` [S2, unofficial].
   - Stack into a single array `images` of shape `[N, H', W', 3]` [S2, unofficial].
6. **Adjust Poses:**
   - Divide camera centers in `poses` by the scaling factor `factor` to account for the image resize [S2, unofficial].
   - Convert `poses` from `[3, 5, N]` to standard `[N, 3, 4]` format if required by the renderer [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import imageio
import os
import subprocess

def load_llff_data_minified(basedir, factor=8, target_size=None):
    # Step 1: Load file lists
    with open(os.path.join(basedir, 'train.txt'), 'r') as f:
        train_files = [line.strip() + '.png' for line in f if line.strip()]
    with open(os.path.join(basedir, 'test.txt'), 'r') as f:
        test_files = [line.strip() + '.png' for line in f if line.strip()]
    
    all_files = train_files + test_files
    N = len(all_files)
    
    # Step 2: Minification
    minified_dir = os.path.join(basedir, 'minified')
    os.makedirs(minified_dir, exist_ok=True)
    
    if factor > 1 or target_size:
        # Use ImageMagick mogrify
        cmd = ['mogrify', '-path', minified_dir]
        if target_size:
            cmd.extend(['-resize', f'{target_size[0]}x{target_size[1]}'])
        else:
            cmd.extend(['-resize', f'{100//factor}%'])
        cmd.extend([os.path.join(basedir, '*.jpg')]) # Assuming jpg source
        subprocess.run(cmd, check=True)
        
        # Update file lists to minified dir
        train_files = [os.path.join(minified_dir, f) for f in train_files]
        test_files = [os.path.join(minified_dir, f) for f in test_files]
        all_files = train_files + test_files
    
    # Step 3: Load Poses and Bounds
    poses_bounds = np.load(os.path.join(basedir, 'poses_bounds.npy'))
    # Reshape to [N, 17] or [3, 5, N] depending on source layout
    # Standard LLFF format is [N, 17] where last 2 are bounds
    poses = poses_bounds[:, :15].reshape([-1, 3, 5]) # [N, 3, 5]
    bounds = poses_bounds[:, -2:] # [N, 2]
    
    # Adjust camera centers by scaling factor
    if factor > 1:
        poses[:, :3, 3] /= factor
    
    # Step 4: Split Data
    # Use every 8th image as test set
    test_indices = list(range(0, N, 8))
    train_indices = [i for i in range(N) if i not in test_indices]
    
    # Step 5: Load and Normalize Images
    images = []
    for f in all_files:
        img = imageio.imread(f)
        if img.shape[2] == 4:
            img = img[:, :, :3] # Remove alpha
        images.append(img.astype(np.float32) / 255.0)
    images = np.stack(images, axis=0) # [N, H, W, 3]
    
    # Step 6: Prepare Poses for Renderer
    # Convert [N, 3, 5] to [N, 3, 4] and bounds to [N, 2]
    # Note: LLFF poses often have extra columns; standard NeRF expects [N, 3, 4]
    # The first 3x4 are rotation/translation, the 5th column might be unused or part of bounds
    # Adjust based on specific renderer requirements [S2, unofficial]
    poses_final = poses[:, :, :4] # [N, 3, 4]
    bounds_final = bounds # [N, 2]
    
    return images, poses_final, bounds_final, train_indices, test_indices
```

**Parameters:**
- `factor`: Downscale factor (default: 8) [S2, unofficial].
- `target_size`: Target resolution (e.g., 378×504) [S1].

**Edge Cases:**
- **Precision:** Image normalization to `[0, 1]` requires float32 to avoid clipping [S2, unofficial].
- **Boundaries:** Ensure `mogrify` command handles file extensions correctly (jpg vs png) [S2, unofficial].
- **Degenerate Input:** If `poses_bounds.npy` is missing or malformed, loading fails [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Dominated by I/O and ImageMagick resizing. Linear in number of images $N$ and resolution $H \times W$ [S1][S2, unofficial].
- **Memory Complexity:** $O(N \cdot H' \cdot W')$ for storing minified images [S2, unofficial].
- **Performance:** Reduces training time significantly by lowering resolution (e.g., 1/8th) [S1].
- **Strengths:** Standardized preprocessing for LLFF; reduces VRAM usage [S1][S2, unofficial].
- **Limitations:** Relies on external ImageMagick tool; fixed split ratio (1/8) may not suit all sparse scenarios [S1][S2, unofficial].

## Background
Originated in the NeRF paper and subsequent LLFF-specific implementations [S1][S2, unofficial]. The 1/8th downsampling and every-8th-image test split became community standards for LLFF evaluation [S1].

## Sources
[S1] SPARF: Neural Radiance Fields from Sparse and Noisy Poses, https://doi.org/10.1109/cvpr52729.2023.00408, fetched 2026-10-26
[S2] weiyithu/NerfingMVS, https://github.com/weiyithu/NerfingMVS, fetched 2026-10-27 (unofficial)
