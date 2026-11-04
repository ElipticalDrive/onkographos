# Volume Data Loading and Normalization

Also called: VolumeHandler

## Overview
This method implements a C++ class for loading 3D and 4D volumetric data from `.npy` files, normalizing the scalar values, and managing frame indices for time-varying volumes [S1, unofficial]. It differs from sibling methods like the Raw Volume Data Reader and DICOM loader by strictly relying on the NumPy binary format and applying a specific min-subtraction normalization strategy rather than standard min-max scaling to $[0, 1]$ [S1, unofficial]. It solves the problem of ingesting pre-saved NumPy volume arrays into a rendering pipeline while ensuring consistent data types and value ranges.

## How it works
**Inputs and Outputs:**
*   **Input:** A file path to a `.npy` file containing 3D or 4D data. The template type $T$ (e.g., `float`, `uint8_t`) must match the data stored in the file [S1, unofficial].
*   **Output:** A populated `VolumeFrames` structure containing normalized float data, shape metadata, and min/max values [S1, unofficial].

**Data Structures:**
The method uses the following internal state:
*   `VolumeFrames`: A `std::vector<VolumeData3D>`. Each `VolumeData3D` element contains:
    *   `Shape`: A 3D array of `uint32_t` representing dimensions $(X, Y, Z)$ [S1, unofficial].
    *   `Data`: A `std::vector<float>` storing the voxel values [S1, unofficial].
    *   `MinValue`: `float`, the minimum value in the original data before normalization [S1, unofficial].
    *   `MaxValue`: `float`, the maximum value in the original data before normalization [S1, unofficial].
*   `IsTimeVaryingVolume`: `bool`. `true` if the input is 4D, `false` otherwise [S1, unofficial].
*   `FrameIndex`: `int`. The current frame index for time-varying volumes [S1, unofficial].

**Steps:**
1.  **Validation:** Check if the file exists and has the `.npy` extension. If not, throw a `runtime_error` [S1, unofficial].
2.  **Reading:** Load the data using `npy::read_npy<T>`. Let the loaded shape be $d.shape$ and data be $d.data$ [S1, unofficial].
3.  **Dimensionality Check:** Verify $d.shape.size()$ is either 3 or 4. If not, throw a `runtime_error` [S1, unofficial].
4.  **Processing 3D Volumes (Single Frame):**
    *   If $d.shape.size() == 3$:
        *   Resize `VolumeFrames` to size 1.
        *   Set `Shape` to $d.shape$.
        *   Resize `Data` to size $Shape[0] \times Shape[1] \times Shape[2]$.
        *   Transform $d.data$ to `float`.
        *   Find $MinValue = \min(d.data)$ and $MaxValue = \max(d.data)$ [S1, unofficial].
        *   Normalize: For each element $v$ in `Data`, $v_{new} = v - MinValue$.
        *   Set `IsTimeVaryingVolume` to `false` [S1, unofficial].
5.  **Processing 4D Volumes (Time-Varying):**
    *   If $d.shape.size() == 4$:
        *   Let $N = d.shape[0]$ (number of frames).
        *   Let $X = d.shape[1], Y = d.shape[2], Z = d.shape[3]$.
        *   Resize `VolumeFrames` to size $N$.
        *   Set `IsTimeVaryingVolume` to `true` [S1, unofficial].
        *   For each frame $t$ from $0$ to $N-1$:
            *   Extract the sub-array for frame $t$ from $d.data$.
            *   Set `Shape` to $(X, Y, Z)$.
            *   Resize `Data` for this frame to $X \times Y \times Z$.
            *   Copy frame data and transform to `float`.
            *   Find $MinValue_t = \min(\text{frame}_t)$ and $MaxValue_t = \max(\text{frame}_t)$.
            *   Set `MinValue` and `MaxValue` for this frame.
            *   Normalize: For each element $v$ in the frame's `Data`, $v_{new} = v - MinValue_t$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class VolumeData3D:
    Shape: list[int]  # [x, y, z]
    Data: list[float]
    MinValue: float
    MaxValue: float

class VolumeLoader:
    VolumeFrames: list[VolumeData3D]
    IsTimeVaryingVolume: bool
    FrameIndex: int

    def load_and_normalize(self, filepath: str, T: type):
        # 1. Validation
        if not filepath.endswith('.npy') or not file_exists(filepath):
            raise RuntimeError("File must exist and be .npy")

        # 2. Reading
        # npy::read_npy<T> returns an object with .shape (list) and .data (list of T)
        d = read_npy(filepath, T)

        # 3. Dimensionality Check
        if len(d.shape) not in (3, 4):
            raise RuntimeError("Shape dimensionality must be 3 or 4")

        self.VolumeFrames = []
        self.IsTimeVaryingVolume = False

        if len(d.shape) == 3:
            # 4. 3D Volume Handling
            self.VolumeFrames.resize(1)
            frame = self.VolumeFrames[0]
            frame.Shape = d.shape
            size = d.shape[0] * d.shape[1] * d.shape[2]
            frame.Data.resize(size)
            
            # Transform to float and find min/max
            float_data = [float(x) for x in d.data]
            frame.MinValue = min(float_data)
            frame.MaxValue = max(float_data)
            
            # Normalize: subtract MinValue
            frame.Data = [x - frame.MinValue for x in float_data]
            
            self.IsTimeVaryingVolume = False

        elif len(d.shape) == 4:
            # 5. 4D Volume Handling
            num_frames = d.shape[0]
            x, y, z = d.shape[1], d.shape[2], d.shape[3]
            self.VolumeFrames.resize(num_frames)
            self.IsTimeVaryingVolume = True

            for t in range(num_frames):
                frame = self.VolumeFrames[t]
                frame.Shape = [x, y, z]
                frame_size = x * y * z
                frame.Data.resize(frame_size)
                
                # Extract frame data (assuming d.data is flattened 4D array)
                # Note: Exact slicing logic depends on npy layout, assumed row-major here
                start_idx = t * frame_size
                end_idx = start_idx + frame_size
                frame_raw = d.data[start_idx:end_idx]
                
                float_data = [float(x) for x in frame_raw]
                frame.MinValue = min(float_data)
                frame.MaxValue = max(float_data)
                
                # Normalize
                frame.Data = [x - frame.MinValue for x in float_data]

        self.FrameIndex = 0
```

**Parameters:**
*   $T$: Template type for input data (e.g., `float`, `uint8_t`) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Normalization Shift:** The method subtracts `MinValue` from all elements. The resulting data range is $[0, MaxValue - MinValue]$. It does *not* scale to $[0, 1]$ [S1, unofficial].
*   **Unsupported Formats:** Only `.npy` files are supported. Other formats will trigger an error [S1, unofficial].
*   **Dimensionality:** Inputs with dimensions other than 3 or 4 are rejected [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the total number of voxels, as it requires a single pass to read, convert, and find min/max, followed by a pass to normalize [S1, unofficial].
*   **Memory Complexity:** $O(N)$ for storing the float data. The original integer data is converted to float, potentially increasing memory usage if the source was 8-bit or 16-bit [S1, unofficial].
*   **Strengths:** Simple implementation for NumPy-based pipelines; handles both static and dynamic volumes uniformly [S1, unofficial].
*   **Limitations:** Strict format dependency (`.npy` only); normalization preserves absolute magnitude differences rather than normalizing to a standard range, which may require downstream adjustment [S1, unofficial].

## Background
This method originates from the DDVR project, an unofficial implementation of a volume rendering system [S1, unofficial]. It is designed to bridge NumPy data structures with C++ rendering components.

## Sources
[S1] AoGao-Kedoka/DDVR, https://github.com/AoGao-Kedoka/DDVR, fetched 2026-10-26 (unofficial)
