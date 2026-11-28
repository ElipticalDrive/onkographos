# DICOM Slice Processing
Also called: DICOM Slice Loading and Sorting

## Overview
Loads DICOM slices from a directory, determines the global intensity range, and sorts the slices into a coherent volume order [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A directory path containing DICOM slice files (specifically `.IMA` extensions in the reference implementation) [S1, unofficial].
- Output: A sorted array of `Slice` objects and global intensity bounds (`_minIntensity`, `_maxIntensity`) [S1, unofficial].

**Data Structures:**
- `Slice`: An object representing a single DICOM slice. It contains metadata fields `LargestImagePixelValue` and `SmallestImagePixelValue` used for intensity tracking [S1, unofficial]. The sorting key is not explicitly defined in the source but implies a spatial or temporal ordering derived from DICOM tags [G].

**Steps:**
1. Retrieve all files with the extension `*.IMA` from the specified directory [S1, unofficial].
2. Initialize global intensity bounds: `maxIntensity = -1`, `minIntensity = 99999` [S1, unofficial].
3. Iterate through each slice file: create a `Slice` object and update `maxIntensity` if `slice.info.LargestImagePixelValue > maxIntensity`, and update `minIntensity` if `slice.info.SmallestImagePixelValue < minIntensity` [S1, unofficial].
4. Store the final `minIntensity` and `maxIntensity` in `_minIntensity` and `_maxIntensity` [S1, unofficial].
5. Sort the array of `Slice` objects [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def process_dicom_slices(directory_path):
    import os
    # Step 1: Get all .IMA files
    files = [f for f in os.listdir(directory_path) if f.endswith('.IMA')]
    
    slices = []
    max_intensity = -1
    min_intensity = 99999
    
    for filename in files:
        filepath = os.path.join(directory_path, filename)
        # Step 3: Create Slice object and parse metadata
        slice_obj = Slice(filepath)
        
        largest = slice_obj.info.LargestImagePixelValue
        smallest = slice_obj.info.SmallestImagePixelValue
        
        if largest > max_intensity:
            max_intensity = largest
        if smallest < min_intensity:
            min_intensity = smallest
            
        slices.append(slice_obj)
        
    # Step 4: Store global bounds
    _minIntensity = min_intensity
    _maxIntensity = max_intensity
    
    # Step 5: Sort slices
    slices.sort(key=lambda s: s.sort_key)
    
    return slices, _minIntensity, _maxIntensity

class Slice:
    def __init__(self, filepath):
        # Standard DICOM parsing to populate metadata
        self.info = {'LargestImagePixelValue': 0, 'SmallestImagePixelValue': 0}
        self.sort_key = 0
```

**Parameters:**
- File extension: `*.IMA` [S1, unofficial].
- Initial min intensity: `99999` [S1, unofficial].
- Initial max intensity: `-1` [S1, unofficial].

**Edge Cases and Pitfalls:**
- Empty directory: Returns empty list and default bounds (`_minIntensity = 99999`, `_maxIntensity = -1`) [S1, unofficial].
- Sorting ambiguity: The source does not specify the exact sorting key [G].
- Missing metadata: If `LargestImagePixelValue` or `SmallestImagePixelValue` are missing, the intensity range calculation may be incorrect [G].

## Subtopics

### [Slice to Binary Volume Conversion](6714-slice-to-binary-volume-conversion.md)
Converts a directory of 2D image slices into a binary volume file (`.bin`) by assuming slices are ordered and calculating bounding box dimensions based on a fixed X width.

## Operating characteristics
- Time Complexity: $O(N \log N)$ due to sorting, where $N$ is the number of slices. Intensity scanning is $O(N)$ [G].
- Memory Complexity: $O(N)$ to store the slice objects [G].
- Strengths: Simple directory-based loading; automatic intensity normalization [G].
- Limitations: Relies on specific file extensions (`.IMA`); sorting logic is implementation-dependent [G].

## Background
- Origin: Not given in the sources.
- Usage: Used to preprocess DICOM data for volume rendering pipelines [G].

## Sources
[S1] Torbenhjl/Marching-Tetrahedra-on-CT-headscan, https://github.com/Torbenhjl/Marching-Tetrahedra-on-CT-headscan, fetched 2026-10-27 (unofficial)
