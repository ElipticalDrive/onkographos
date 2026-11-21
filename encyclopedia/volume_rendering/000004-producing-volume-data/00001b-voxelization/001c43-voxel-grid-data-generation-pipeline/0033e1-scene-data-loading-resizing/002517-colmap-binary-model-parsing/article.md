# COLMAP Binary Model Parsing
## Overview
Parses COLMAP reconstruction files stored in binary format (`.bin`) into structured Python objects (`Camera`, `Image`, `Point3D`) [S1, unofficial]. It serves as the primary data loading step in the [Scene Data Loading & Resizing](3420-scene-data-loading-resizing.md) pipeline [S1, unofficial].
## How it works
**Inputs:** Path to a COLMAP binary reconstruction file (typically `sparse/0/model.bin`) [S1, unofficial].
**Outputs:** Dictionaries or lists of `Camera`, `Image`, and `Point3D` objects [S1, unofficial].

**Data Structures:**
The parser constructs the following objects [S1, unofficial]:
*   `Camera`: `id` (int), `model` (str), `width` (int), `height` (int), `params` (np.array of floats).
*   `Image`: `id` (int), `qvec` (np.array of 4 floats), `tvec` (np.array of 3 floats), `camera_id` (int), `name` (str), `xys` (np.array), `point3D_ids` (np.array).
*   `Point3D`: `id` (int), `xyz` (np.array of 3 floats), `rgb` (np.array of 3 floats), `error` (float), `image_ids` (np.array), `point2D_idxs` (np.array).

**Binary Layout & Parsing Steps:**
The file structure follows the COLMAP binary format specification [S1, unofficial]:

1.  **Read Cameras:**
    *   Read `num_cameras` as an 8-byte unsigned long long (`struct.unpack("Q", ...)`). [S1, unofficial]
    *   For each camera:
        *   Read `camera_id` (int), `model_id` (int), `width` (Q), `height` (Q). Total 24 bytes for these fields. [S1, unofficial]
        *   Read `num_params` doubles (`d`) based on the camera model. [S1, unofficial]
        *   Map `model_id` to a string name (e.g., `SIMPLE_PINHOLE`, `PINHOLE`, `RADIAL`) [G].
        *   Store as `Camera` object.

2.  **Read Images:**
    *   Read `num_reg_images` as an 8-byte unsigned long long (`Q`). [S1, unofficial]
    *   For each image:
        *   Read `image_id` (int), `qvec` (4 doubles), `tvec` (3 doubles), `camera_id` (int). Total 64 bytes. (Note: sources list `qvec (5d)` but the 64-byte total and standard COLMAP format use 4 doubles [G]).
        *   Read `name` as a null-terminated ASCII string. [S1, unofficial]
        *   Store as `Image` object.

3.  **Read 3D Points:**
    *   Read `num_points3D` as an 8-byte unsigned long long (`Q`) [G].
    *   For each point:
        *   Read `point3D_id` (int), `xyz` (3 doubles), `rgb` (3 bytes, converted to floats), `error` (double). [S1, unofficial]
        *   Read `num_views` (int).
        *   Read `image_ids` and `point2D_idxs` as `num_views` pairs of integers. [S1, unofficial]
        *   Store as `Point3D` object.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import struct
import numpy as np
import collections

Camera = collections.namedtuple("Camera", ["id", "model", "width", "height", "params"])
Image = collections.namedtuple("Image", ["id", "qvec", "tvec", "camera_id", "name", "xys", "point3D_ids"])
Point3D = collections.namedtuple("Point3D", ["id", "xyz", "rgb", "error", "image_ids", "point2D_idxs"])

# Standard COLMAP camera model parameter counts [G]
CAMERA_MODEL_PARAMS = {
    0: 3, 1: 6, 2: 4, 3: 5, 4: 8, 5: 8, 6: 12, 7: 5, 8: 6, 9: 12
}
CAMERA_MODEL_NAMES = {
    0: "SIMPLE_PINHOLE", 1: "PINHOLE", 2: "SIMPLE_RADIAL", 3: "RADIAL",
    4: "OPENCV", 5: "OPENCV_FISHEYE", 6: "FULL_OPENCV", 7: "SIMPLE_RADIAL_FISHEYE",
    8: "RADIAL_FISHEYE", 9: "THIN_PRISM_FISHEYE"
}

def parse_colmap_binary(path):
    cameras = []
    images = []
    points = []

    with open(path, "rb") as f:
        # 1. Cameras
        num_cameras = struct.unpack("Q", f.read(8))[0]
        for _ in range(num_cameras):
            cam_id, model_id, width, height = struct.unpack("iiQQ", f.read(24))
            num_params = CAMERA_MODEL_PARAMS.get(model_id, 0)
            params = np.frombuffer(f.read(num_params * 8), dtype=np.float64)
            cameras.append(Camera(id=cam_id, model=CAMERA_MODEL_NAMES.get(model_id, f"UNKNOWN_{model_id}"), width=width, height=height, params=params))

        # 2. Images
        num_reg_images = struct.unpack("Q", f.read(8))[0]
        for _ in range(num_reg_images):
            img_id, qvec_data, tvec_data, cam_id = struct.unpack("i4d3di", f.read(64))
            qvec = np.array(qvec_data, dtype=np.float64)
            tvec = np.array(tvec_data, dtype=np.float64)
            name = b""
            while True:
                byte = f.read(1)
                if byte == b'\x00' or not byte:
                    break
                name += byte
            images.append(Image(id=img_id, qvec=qvec, tvec=tvec, camera_id=cam_id, name=name.decode("ascii"), xys=np.array([]), point3D_ids=np.array([])))

        # 3. Points
        num_points = struct.unpack("Q", f.read(8))[0]
        for _ in range(num_points):
            p_id, xyz_data, rgb_bytes, error = struct.unpack("i3d3B1d", f.read(39))
            xyz = np.array(xyz_data, dtype=np.float64)
            rgb = np.array(rgb_bytes, dtype=np.float64) / 255.0  # Standard COLMAP stores RGB as 3 bytes [G]
            num_views = struct.unpack("i", f.read(4))[0]
            views_data = f.read(num_views * 8)
            views = struct.unpack(f"{num_views}ii", views_data)
            image_ids = np.array(views[0::2], dtype=np.int32)
            point2D_idxs = np.array(views[1::2], dtype=np.int32)
            points.append(Point3D(id=p_id, xyz=xyz, rgb=rgb, error=error, image_ids=image_ids, point2D_idxs=point2D_idxs))

    return cameras, images, points
```

**Edge Cases and Pitfalls:**
*   **Quaternion Format:** COLMAP uses Hamiltonian quaternions $(w, x, y, z)$ [G].
*   **RGB Precision:** Standard COLMAP binary stores RGB as 3 bytes (0-255) [G]. The notes specify floats; the pseudocode parses bytes and normalizes to [0, 1] to match standard behavior [G].
*   **String Encoding:** Image names are null-terminated ASCII [S1, unofficial]. The parser reads until the null byte and strips it to avoid trailing characters [G].
*   **Endianness:** COLMAP binary files are typically little-endian. If parsing on a big-endian system, byte order must be swapped [G].
## Subtopics

### [COLMAP Text Model Parsing](3600-colmap-text-model-parsing.md)
Parses COLMAP reconstruction files stored in text format (`.txt`) into structured Python objects. Choose this when debugging or when binary parsing fails; it is slower but more human-readable.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of cameras, images, and points. Parsing is I/O bound [G].
*   **Memory Complexity:** $O(N)$ to store the parsed objects. Binary parsing avoids string overhead and whitespace compared to text parsing [G].
*   **Performance:** Significantly faster than text parsing for large scenes (millions of points) [G].
*   **Limitations:** Less human-readable; harder to debug if the binary format changes. Requires exact knowledge of the binary layout [G].
## Background
Originated with COLMAP (Structure-from-Motion and Multi-View Stereo software) by Johannes L. Schoenberger et al. [S1, unofficial]. The binary format was introduced to improve I/O performance for large-scale reconstructions [S1, unofficial].
## Sources
[S1] metin-yat/mini-splat, https://github.com/metin-yat/mini-splat, fetched 2026-10-28 (unofficial)
