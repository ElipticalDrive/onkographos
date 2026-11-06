# COLMAP Text Model Parsing

Also called: SfM Data Reading (Text and Binary)

## Overview
Parses COLMAP reconstruction files stored in text format (`.txt`) into structured Python objects representing cameras, images, and 3D points [S1, S2, unofficial]. It solves the problem of converting raw Structure-from-Motion (SfM) output into a usable scene graph for downstream rendering or reconstruction tasks. Unlike its sibling [COLMAP Binary Model Parsing](3455-colmap-binary-model-parsing.md), which reads compact binary streams, this method reads human-readable lines, making it slower but easier to debug and inspect manually.

## How it works

### Inputs and Outputs
**Inputs:**
- `camera_file_path`: Path to a text file containing camera parameters.
- `image_file_path`: Path to a text file containing image poses and 2D-3D correspondences.

**Outputs:**
- `cameras_dict`: A dictionary mapping `camera_id` (int) to `Camera` objects.
- `images_dict`: A dictionary mapping `image_id` (int) to `Image` objects.

### Data Structures

The parser constructs two primary dataclasses.

**Camera**
- `id` (int): Unique camera identifier.
- `model` (str): Camera model name (e.g., "PINHOLE").
- `width` (int): Image width in pixels.
- `height` (int): Image height in pixels.
- `params` (np.ndarray): Array of floating-point intrinsic parameters. Length depends on the model.

**Image** (extends `BaseImage`)
- `id` (int): Unique image identifier.
- `qvec` (np.ndarray): Quaternion rotation vector `[w, x, y, z]` (4 floats).
- `tvec` (np.ndarray): Translation vector `[tx, ty, tz]` (3 floats).
- `camera_id` (int): ID of the camera that captured this image.
- `name` (str): Filename of the image.
- `xys` (np.ndarray): 2D point coordinates `(x, y)` for each detected feature (N, 2 floats).
- `point3D_ids` (np.ndarray): Corresponding 3D point IDs for each 2D point (N ints). Values of -1 indicate no corresponding 3D point.

**Camera Models**
The parser relies on a registry `CAMERA_MODELS` mapping model names to their parameter counts. Supported models include:
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
    a. Open the camera text file.
    b. Iterate through each line. Skip lines starting with `#` (comments) [S1, S2, unofficial].
    c. Split the line into whitespace-separated elements.
    d. Parse the first element as `camera_id` (int).
    e. Parse the second element as `model` (str).
    f. Parse the third and fourth elements as `width` and `height` (ints) [S1, S2, unofficial].
    g. Parse the remaining elements as floats into the `params` array. The number of params must match the model definition [S1, unofficial].
    h. Store the constructed `Camera` object in `cameras_dict` keyed by `camera_id`.
3.  **Parse Images**:
    a. Open the image text file.
    b. Iterate through each line. Skip comment lines [S1, S2, unofficial].
    c. Parse the first line of an image entry:
        i. `image_id` (int).
        ii. `qvec` (4 floats).
        iii. `tvec` (3 floats).
        iv. `camera_id` (int).
        v. `image_name` (str) [S1, S2, unofficial].
    d. Parse the subsequent line(s) containing 2D points:
        i. The line contains alternating `x`, `y`, and `point3D_id` values.
        ii. Step through the elements with a stride of 3.
        iii. Collect `xys` as pairs of floats and `point3D_ids` as integers [S1, S2, unofficial].
    e. Store the constructed `Image` object in `images_dict` keyed by `image_id`.

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

# Registry of supported camera models
CAMERA_MODELS = {
    "SIMPLE_PINHOLE": 3,
    "PINHOLE": 4,
    "SIMPLE_RADIAL": 4,
    "RADIAL": 5,
    "OPENCV": 8,
    "OPENCV_FISHEYE": 8,
    "FULL_OPENCV": 12,
    "FOV": 5,
    "SIMPLE_RADIAL_FISHEYE": 4,
    "RADIAL_FISHEYE": 5,
    "THIN_PRISM_FISHEYE": 12
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
            
            # Remaining elements are parameters
            params = np.array([float(p) for p in parts[4:]])
            
            cameras[camera_id] = Camera(
                id=camera_id,
                model=model,
                width=width,
                height=height,
                params=params
            )
    return cameras

def read_images_text(path: str, cameras: Dict[int, Camera]) -> Dict[int, Image]:
    images = {}
    with open(path, 'r') as f:
        lines = f.readlines()
    
    i = 0
    while i < len(lines):
        line = lines[i].strip()
        i += 1
        
        if not line or line.startswith('#'):
            continue
        
        parts = line.split()
        image_id = int(parts[0])
        qvec = np.array([float(x) for x in parts[1:5]])
        tvec = np.array([float(x) for x in parts[5:8]])
        camera_id = int(parts[8])
        image_name = parts[9]
        
        # Read 2D points
        # The next line(s) contain x, y, point3D_id triples
        # In COLMAP text format, all 2D points for an image are on subsequent lines
        # or potentially split across multiple lines if the file is formatted that way.
        # Standard COLMAP text output puts all points for an image in the lines following the header.
        
        xys = []
        point3D_ids = []
        
        # We need to read until the next image header or EOF.
        # A robust way is to look ahead or read until a line that looks like an image header.
        # However, standard implementation reads the immediate next line(s) containing points.
        # Let's assume the points are on the immediate next line(s) until the next image block.
        
        # To simplify, we read the next line. If it's empty or comment, skip.
        # If it contains numbers, it's a point line.
        
        while i < len(lines):
            point_line = lines[i].strip()
            i += 1
            
            if not point_line or point_line.startswith('#'):
                continue
            
            # Check if this line starts a new image (has 10+ elements and first is int)
            # This is a heuristic. A stricter parser might rely on file structure guarantees.
            # Standard COLMAP text format: Image header (10 tokens) followed by N lines of 3 tokens.
            # If we encounter a line with > 3 tokens, it's likely a new image header.
            point_parts = point_line.split()
            if len(point_parts) > 3:
                # This is likely the next image header. Push back? 
                # Since we are iterating, we just break and the outer loop will handle it?
                # No, we are inside the inner while. We need to decrement i or handle carefully.
                # For this pseudocode, we assume standard formatting where points are strictly on following lines.
                # If we hit a new image header, we stop reading points for the current image.
                # But we already incremented i. So we need to process this line as the next image.
                # We break the inner loop, and the outer loop continues. 
                # But the outer loop increments i again? No, the outer loop is a while i < len.
                # We need to not increment i for the current line if it's a new header.
                # Let's restructure slightly for correctness.
                break
            
            # Parse point
            x = float(point_parts[0])
            y = float(point_parts[1])
            point3D_id = int(point_parts[2])
            
            xys.append([x, y])
            point3D_ids.append(point3D_id)
        
        # If we broke out because of a new image header, we need to process that line
        # in the next iteration of the outer loop. But we already consumed it into point_line.
        # We need to prepend it back or handle it.
        # A cleaner approach:
        
        # Re-reading logic for robustness:
        # The outer loop should just iterate lines.
        pass

    # Refined implementation for clarity and correctness
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
            continue # Skip malformed lines
            
        image_id = int(parts[0])
        qvec = np.array([float(x) for x in parts[1:5]])
        tvec = np.array([float(x) for x in parts[5:8]])
        camera_id = int(parts[8])
        image_name = parts[9]
        
        xys = []
        point3D_ids = []
        
        # Read subsequent lines for 2D points
        while idx < len(lines):
            point_line = lines[idx].strip()
            idx += 1
            
            if not point_line or point_line.startswith('#'):
                continue
            
            point_parts = point_line.split()
            
            # Heuristic: If a line has more than 3 elements, it's likely a new image header
            if len(point_parts) > 3:
                # This line belongs to the next image. We need to process it in the outer loop.
                # We can't easily "push back" in a simple while loop without complex state.
                # Instead, we assume standard COLMAP output where point lines have exactly 3 tokens.
                # If we encounter a line with > 3 tokens, it's a new image.
                # We must decrement idx to re-process this line as an image header.
                idx -= 1
                break
            
            if len(point_parts) == 3:
                x = float(point_parts[0])
                y = float(point_parts[1])
                point3D_id = int(point_parts[2])
                xys.append([x, y])
                point3D_ids.append(point3D_id)
        
        images[image_id] = Image(
            id=image_id,
            qvec=qvec,
            tvec=tvec,
            camera_id=camera_id,
            name=image_name,
            xys=np.array(xys) if xys else np.empty((0, 2)),
            point3D_ids=np.array(point3D_ids) if point3D_ids else np.empty((0,), dtype=int)
        )
        
    return images
```

### Parameters and Edge Cases
- **Precision**: Floats are parsed as standard double-precision (64-bit) in the source text, stored as `float` (32-bit) or `double` (64-bit) in numpy depending on implementation. [S1, S2, unofficial]
- **Empty Points**: An image may have 0 2D points. The `xys` and `point3D_ids` arrays will be empty.
- **Missing 3D Points**: `point3D_id` of -1 indicates a 2D point with no corresponding 3D reconstruction.
- **Comment Lines**: Lines starting with `#` are ignored.
- **Whitespace**: Splitting is done on any whitespace.

## Operating characteristics

- **Time Complexity**: $O(N)$ where $N$ is the number of lines in the text files. Parsing is linear.
- **Memory Complexity**: $O(M)$ where $M$ is the number of cameras and images, plus storage for all 2D-3D correspondences.
- **Performance**: Significantly slower than binary parsing due to string parsing overhead and lack of compact memory layout. Suitable for small to medium datasets or debugging.
- **Strengths**: Human-readable, easy to debug, no need to know byte offsets or struct packing rules.
- **Limitations**: Slow for large-scale SfM reconstructions with millions of points.

## Background

Originating from the COLMAP (Structure-from-Motion and Multi-View Stereo) software suite, this parsing method is part of the standard utility library provided by ETH Zurich and UNC Chapel Hill [S2, unofficial]. It is widely used in 3D Gaussian Splatting implementations (e.g., [S1, S2, unofficial]) to load scene data for training and rendering.

## Sources

[S1] metin-yat/mini-splat, https://github.com/metin-yat/mini-splat, fetched 2026-10-28 (unofficial)
[S2] https://github.com/WangFeng18/3d-gaussian-splatting, https://github.com/WangFeng18/3d-gaussian-splatting, fetched 2026-10-30 (unofficial)
