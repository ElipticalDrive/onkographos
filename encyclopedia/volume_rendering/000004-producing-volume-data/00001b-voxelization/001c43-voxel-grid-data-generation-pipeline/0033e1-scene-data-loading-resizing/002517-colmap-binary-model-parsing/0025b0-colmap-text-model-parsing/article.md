# COLMAP Text Model Parsing

Also called: SfM Data Reading (Text and Binary)

## Overview
Parses COLMAP reconstruction files stored in text format (`.txt`) into structured Python objects representing cameras and images [S1, S2, unofficial]. It converts raw Structure-from-Motion (SfM) output into usable data structures compatible with downstream rendering pipelines [S2, unofficial]. Builds on [COLMAP Binary Model Parsing](3455-colmap-binary-model-parsing.md) by reading human-readable lines instead of compact binary streams [S2, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
- `camera_file_path`: Path to a text file containing camera parameters.
- `image_file_path`: Path to a text file containing image poses and 2D-3D correspondences.

**Outputs:**
- `cameras_dict`: A dictionary mapping `camera_id` (int) to `Camera` objects.
- `images_dict`: A dictionary mapping `image_id` (int) to `Image` objects.

### Data Structures

The parser constructs two primary dataclasses [S2, unofficial].

**Camera**
- `id` (int): Unique camera identifier [S2, unofficial].
- `model` (str): Camera model name (e.g., "PINHOLE") [S2, unofficial].
- `width` (int): Image width in pixels [S2, unofficial].
- `height` (int): Image height in pixels [S2, unofficial].
- `params` (np.ndarray): Array of floating-point intrinsic parameters. Length depends on the model [S2, unofficial].

**Image** (extends `BaseImage`)
- `id` (int): Unique image identifier [S2, unofficial].
- `qvec` (np.ndarray): Quaternion rotation vector `[w, x, y, z]` (4 floats) [S2, unofficial].
- `tvec` (np.ndarray): Translation vector `[tx, ty, tz]` (3 floats) [S2, unofficial].
- `camera_id` (int): ID of the camera that captured this image [S2, unofficial].
- `name` (str): Filename of the image [S2, unofficial].
- `xys` (np.ndarray): 2D point coordinates `(x, y)` for each detected feature (N, 2 floats) [S2, unofficial].
- `point3D_ids` (np.ndarray): Corresponding 3D point IDs for each 2D point (N ints) [S2, unofficial].

**Camera Models**
The parser relies on a registry `CAMERA_MODELS` mapping model names to their parameter counts [S1, unofficial]. Supported models include:
- `SIMPLE_PINHOLE`: 3 params
- `PINHOLE`: 4 params
- `SIMPLE_RADIAL`: 4 params
- `RADIAL`: 5 params
- `OPENCV`: 8 params
- `OPENCV_FISHEYE`: 8 params
- `FULL_OPENCV`: 12 params
- `FOV`: 5 params
- `SIMPLE_RADIAL_FISHEYE`: 4 params
- `RADIAL_FISHEYE`: 5 params
- `THIN_PRISM_FISHEYE`: 12 params

### Numbered Steps

1.  **Initialize Registries**: Load the `CAMERA_MODELS` dictionary to map model names to expected parameter counts [S1, unofficial].
2.  **Parse Cameras**:
    a. Open the camera text file [S1, S2, unofficial].
    b. Iterate through each line. Skip lines starting with `#` (comments) [S1, S2, unofficial].
    c. Split the line into whitespace-separated elements [S1, unofficial].
    d. Parse the first element as `camera_id` (int) [S1, unofficial].
    e. Parse the second element as `model` (str) [S1, unofficial].
    f. Parse the third and fourth elements as `width` and `height` (ints) [S1, S2, unofficial].
    g. Parse the remaining elements as floats into the `params` array [S1, unofficial].
    h. Store the constructed `Camera` object in `cameras_dict` keyed by `camera_id` [S1, unofficial].
3.  **Parse Images**:
    a. Open the image text file [S1, S2, unofficial].
    b. Iterate through each line. Skip comment lines [S1, S2, unofficial].
    c. Parse the first line of an image entry: `image_id` (int), `qvec` (4 floats), `tvec` (3 floats), `camera_id` (int), `image_name` (str) [S1, S2, unofficial].
    d. Parse the subsequent line(s) containing 2D points:
        i. The line contains alternating `x`, `y`, and `point3D_id` values [S1, unofficial].
        ii. Step through the elements with a stride of 3 [S1, unofficial].
        iii. Collect `xys` as pairs of floats and `point3D_ids` as integers [S1, S2, unofficial].
    e. Store the constructed `Image` object in `images_dict` keyed by `image_id` [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from dataclasses import dataclass
from typing import Dict, List, Tuple

@dataclass
class Camera:
    id: int
    model: str
    width: int
    height: int
    params: np.ndarray

@dataclass
class Image:
    id: int
    qvec: np.ndarray
    tvec: np.ndarray
    camera_id: int
    name: str
    xys: np.ndarray
    point3D_ids: np.ndarray

CAMERA_MODELS = {
    "SIMPLE_PINHOLE": 3, "PINHOLE": 4, "SIMPLE_RADIAL": 4, "RADIAL": 5,
    "OPENCV": 8, "OPENCV_FISHEYE": 8, "FULL_OPENCV": 12, "FOV": 5,
    "SIMPLE_RADIAL_FISHEYE": 4, "RADIAL_FISHEYE": 5, "THIN_PRISM_FISHEYE": 12
}

def read_cameras_text(path: str) -> Dict[int, Camera]:
    cameras = {}
    with open(path, 'r') as f:
        for line in f:
            line = line.strip()
            if not line or line.startswith('#'):
                continue
            parts = line.split()
            camera_id = int(parts[0])
            model = parts[1]
            width = int(parts[2])
            height = int(parts[3])
            params = np.array([float(p) for p in parts[4:]])
            cameras[camera_id] = Camera(id=camera_id, model=model, width=width, height=height, params=params)
    return cameras

def read_images_text(path: str) -> Dict[int, Image]:
    images = {}
    with open(path, 'r') as f:
        lines = f.readlines()
    idx = 0
    while idx < len(lines):
        line = lines[idx].strip()
        idx += 1
        if not line or line.startswith('#'):
            continue
        parts = line.split()
        if len(parts) < 10:
            continue
        image_id = int(parts[0])
        qvec = np.array([float(x) for x in parts[1:5]])
        tvec = np.array([float(x) for x in parts[5:8]])
        camera_id = int(parts[8])
        image_name = parts[9]
        xys = []
        point3D_ids = []
        while idx < len(lines):
            point_line = lines[idx].strip()
            idx += 1
            if not point_line or point_line.startswith('#'):
                continue
            point_parts = point_line.split()
            if len(point_parts) >= 10:
                idx -= 1
                break
            if len(point_parts) == 3:
                xys.append([float(point_parts[0]), float(point_parts[1])])
                point3D_ids.append(int(point_parts[2]))
        images[image_id] = Image(
            id=image_id, qvec=qvec, tvec=tvec, camera_id=camera_id, name=image_name,
            xys=np.array(xys) if xys else np.empty((0, 2)),
            point3D_ids=np.array(point3D_ids) if point3D_ids else np.empty((0,), dtype=int)
        )
    return images
```

### Parameters and Edge Cases
- **Precision**: Not given in the sources.
- **Empty Points**: An image may have 0 2D points. The `xys` and `point3D_ids` arrays will be empty [G].
- **Missing 3D Points**: `point3D_id` of -1 indicates a 2D point with no corresponding 3D reconstruction [G].
- **Comment Lines**: Lines starting with `#` are ignored [S1, S2, unofficial].
- **Whitespace**: Splitting is done on any whitespace [S1, unofficial].

## Operating characteristics
- **Time Complexity**: $O(N)$ where $N$ is the number of lines in the text files [G].
- **Memory Complexity**: $O(M)$ where $M$ is the number of cameras and images, plus storage for all 2D-3D correspondences [G].
- **Performance**: Not given in the sources.
- **Strengths**: Human-readable, easy to debug, no need to know byte offsets or struct packing rules [G].
- **Limitations**: Slow for large-scale SfM reconstructions with millions of points [G].

## Background
This parsing method originates from the COLMAP (Structure-from-Motion and Multi-View Stereo) software suite [S2, unofficial]. It is widely used in 3D Gaussian Splatting implementations to load scene data for training and rendering [S1, S2, unofficial].

## Sources

[S1] metin-yat/mini-splat, https://github.com/metin-yat/mini-splat, fetched 2026-10-28 (unofficial)
[S2] https://github.com/WangFeng18/3d-gaussian-splatting, https://github.com/WangFeng18/3d-gaussian-splatting, fetched 2026-10-30 (unofficial)
