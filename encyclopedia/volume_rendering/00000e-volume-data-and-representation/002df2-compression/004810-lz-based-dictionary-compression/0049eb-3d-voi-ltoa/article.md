# 3D-VOI-LtoA

Also called: Hybrid Lossless Compression with Selective Bounding Volume, L to A codec, L to A coding with VOI extraction, Object-based 3D medical image compression, three-dimensional-volume of interest-LtoA Method

## Overview
3D-VOI-LtoA is a two-phase lossless compression technique for 3D medical images that separates foreground data from zero-intensity background using a Selective Bounding Volume (SBV) method, then encodes the resulting Volume of Interest (VOI) using a hybrid Lempel-Ziv-Welch (LZW) and Arithmetic Coding pipeline [S1]. It distinguishes itself from sibling methods like [LZV Algorithm](8069-lzv-algorithm.md) by explicitly excluding large regions of empty background voxels before dictionary-based compression, thereby reducing the data volume processed by the encoder and achieving higher compression ratios on sparse medical datasets [S1].

## How it works

### Inputs and Outputs
*   **Input:** A 3D medical image volume (typically 8-bit depth) containing a foreground object (VOI) surrounded by a zero-intensity background [S1].
*   **Output:** A compressed bit stream representing the VOI, accompanied by metadata defining the Bounding Volume coordinates and the original volume dimensions [S1].

### Data Structures
The method relies on the following implicit and explicit structures:
1.  **Input Volume:** A 3D array of voxels $V[x, y, z]$. The sources specify 8-bit depth, implying values $0-255$ [S1].
2.  **Bounding Volume Metadata:** A set of global coordinates $(x_{min}, x_{max}, y_{min}, y_{max}, z_{min}, z_{max})$ defining the axis-aligned bounding box of the VOI [S1].
3.  **Compressed Stream:** The output of the Arithmetic Encoder [S1].

### Steps

#### Phase 1: VOI Extraction via Selective Bounding Volume (SBV)
1.  **Input:** 3D volume $V$.
2.  **Identify Null Slices:** Scan the volume along the depth axis (e.g., $z$-axis). Identify slices where all voxels are zero (background). These are considered "blank" or "null" slices [S1].
3.  **Determine Active Range:** Identify the minimum and maximum indices along the depth axis that contain at least one non-zero voxel. Let these be $z_{start}$ and $z_{end}$ [S1].
4.  **Construct Bounding Box:** For all slices $z$ where $z_{start} \le z \le z_{end}$, determine the minimum and maximum non-zero coordinates in the $x$ and $y$ dimensions.
    *   Let $x_{min}(z), x_{max}(z), y_{min}(z), y_{max}(z)$ be the bounds of non-zero voxels in slice $z$.
    *   Compute global bounds:
        $$ X_{min} = \min_{z} x_{min}(z) $$
        $$ X_{max} = \max_{z} x_{max}(z) $$
        $$ Y_{min} = \min_{z} y_{min}(z) $$
        $$ Y_{max} = \max_{z} y_{max}(z) $$
    *   The sources describe applying a "bounding box morphological operation" to confine these global coordinates [S1]. This effectively creates a tight axis-aligned bounding box around the union of all non-zero voxels in the active slices.
5.  **Extract VOI:** Define the VOI as the sub-volume $V_{VOI}$ bounded by $[X_{min}, X_{max}] \times [Y_{min}, Y_{max}] \times [z_{start}, z_{end}]$.
6.  **Store Metadata:** Record the global coordinates $(X_{min}, X_{max}, Y_{min}, Y_{max}, z_{start}, z_{end})$ and the original volume dimensions $(W, H, D)$ [S1].

#### Phase 2: Hybrid L to A Encoding
1.  **Input:** The extracted VOI $V_{VOI}$.
2.  **LZW Encoding:** Apply the Lempel-Ziv-Welch (LZW) algorithm to the voxel data of $V_{VOI}$.
    *   Initialize the dictionary with all possible single-voxel values (0-255 for 8-bit).
    *   Parse the input stream to find the longest match in the dictionary.
    *   Output the code for the match and add the new sequence (match + next symbol) to the dictionary.
    *   Let the output of this stage be a sequence of LZW codes $C_{LZW}$ [S1].
3.  **Arithmetic Encoding:** Apply Arithmetic Coding to the sequence $C_{LZW}$.
    *   Estimate symbol probabilities for the LZW codes (typically uniform or based on frequency analysis of the specific dataset) [S1].
    *   Encode the sequence $C_{LZW}$ into a binary bit stream $B_{final}$ [S1].
4.  **Output:** Transmit $B_{final}$ along with the Bounding Volume metadata [S1].

#### Phase 3: Decoding and Reconstruction
1.  **Input:** Compressed bit stream $B_{final}$ and Bounding Volume metadata.
2.  **Arithmetic Decoding:** Decode $B_{final}$ using Arithmetic Decoding to recover the sequence of LZW codes $C_{LZW}$ [S1].
3.  **Inverse LZW:** Apply Inverse LZW to $C_{LZW}$ to reconstruct the voxel data of the VOI, $V'_{VOI}$ [S1].
4.  **Reconstruction:**
    *   Create an empty 3D volume $V_{recon}$ of the original dimensions $(W, H, D)$, initialized to zero [S1].
    *   Map $V'_{VOI}$ into $V_{recon}$ at the position defined by the Bounding Volume coordinates $(X_{min}, Y_{min}, z_{start})$ [S1].
    *   The background remains zero, preserving the original structure [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_3d_voi_ltoa(volume_3d):
    """
    Encodes a 3D medical image using 3D-VOI-LtoA.
    volume_3d: 3D numpy array or list of lists of lists, 8-bit integers.
    Returns: (compressed_bits, metadata)
    """
    # Phase 1: SBV Extraction
    # 1. Identify active slices (non-zero)
    active_slices = []
    for z in range(volume_3d.shape[2]):
        if np.any(volume_3d[:, :, z] != 0):
            active_slices.append(z)
    
    if not active_slices:
        return b'', {'z_min': 0, 'z_max': 0, 'x_min': 0, 'x_max': 0, 'y_min': 0, 'y_max': 0, 'dims': volume_3d.shape}

    z_min = min(active_slices)
    z_max = max(active_slices)
    
    # 2. Determine global X/Y bounds for active slices
    x_min_global = volume_3d.shape[1]
    x_max_global = -1
    y_min_global = volume_3d.shape[0]
    y_max_global = -1
    
    for z in range(z_min, z_max + 1):
        slice_data = volume_3d[:, :, z]
        # Find coordinates of non-zero elements
        coords = np.argwhere(slice_data != 0)
        if len(coords) > 0:
            # coords are (y, x)
            ys = coords[:, 0]
            xs = coords[:, 1]
            if np.min(xs) < x_min_global: x_min_global = np.min(xs)
            if np.max(xs) > x_max_global: x_max_global = np.max(xs)
            if np.min(ys) < y_min_global: y_min_global = np.min(ys)
            if np.max(ys) > y_max_global: y_max_global = np.max(ys)
            
    # 3. Extract VOI
    # Note: The source implies the bounding box is tight around non-zero voxels.
    # We extract the sub-volume defined by these bounds.
    voi = volume_3d[y_min_global:y_max_global+1, x_min_global:x_max_global+1, z_min:z_max+1]
    
    metadata = {
        'z_min': z_min,
        'z_max': z_max,
        'x_min': x_min_global,
        'x_max': x_max_global,
        'y_min': y_min_global,
        'y_max': y_max_global,
        'dims': volume_3d.shape
    }
    
    # Phase 2: L to A Encoding
    # 1. LZW Encoding
    lzw_codes = lzw_encode(voi.flatten())
    
    # 2. Arithmetic Encoding
    # Assuming uniform distribution for simplicity as specific probabilities are not detailed in source
    # In practice, frequency table of lzw_codes would be used.
    compressed_bits = arithmetic_encode(lzw_codes)
    
    return compressed_bits, metadata

def decode_3d_voi_ltoa(compressed_bits, metadata):
    """
    Decodes a 3D medical image using 3D-VOI-LtoA.
    Returns: 3D numpy array
    """
    # Phase 3: Decoding
    # 1. Arithmetic Decoding
    lzw_codes = arithmetic_decode(compressed_bits)
    
    # 2. Inverse LZW
    voi_data = lzw_decode(lzw_codes)
    
    # 3. Reconstruction
    # Create empty volume
    reconstructed = np.zeros(metadata['dims'], dtype=np.uint8)
    
    # Map VOI back
    # voi_data is flattened, need to reshape to extracted dimensions
    voi_shape = (
        metadata['y_max'] - metadata['y_min'] + 1,
        metadata['x_max'] - metadata['x_min'] + 1,
        metadata['z_max'] - metadata['z_min'] + 1
    )
    voi_3d = voi_data.reshape(voi_shape)
    
    reconstructed[
        metadata['y_min']:metadata['y_max']+1,
        metadata['x_min']:metadata['x_max']+1,
        metadata['z_min']:metadata['z_max']+1
    ] = voi_3d
    
    return reconstructed

def lzw_encode(data):
    """Standard LZW encoding."""
    dictionary = {chr(i): i for i in range(256)}
    code = 256
    result = []
    w = ""
    for c in data:
        # Handle integer data by converting to bytes or string representation
        # Assuming data is flattened list of integers
        wc = w + chr(c)
        if wc in dictionary:
            w = wc
        else:
            result.append(dictionary[w])
            dictionary[wc] = code
            code += 1
            w = chr(c)
    if w:
        result.append(dictionary[w])
    return result

def lzw_decode(codes):
    """Standard LZW decoding."""
    dictionary = {i: chr(i) for i in range(256)}
    code = 256
    result = []
    if not codes:
        return []
    
    w = chr(codes[0])
    result.append(w)
    
    for k in codes[1:]:
        if k in dictionary:
            entry = dictionary[k]
        elif k == code:
            entry = w + w[0]
        else:
            raise ValueError("Bad compressed k: %d" % k)
        
        result.append(entry)
        dictionary[code] = w + entry[0]
        code += 1
        w = entry
        
    # Convert back to integers
    return [ord(c) for c in "".join(result)]

def arithmetic_encode(data):
    """Placeholder for Arithmetic Encoding implementation."""
    # Implementation depends on probability model.
    # Source does not provide specific arithmetic coding algorithm details.
    # [G] Standard arithmetic coding implementation required here.
    pass

def arithmetic_decode(bits):
    """Placeholder for Arithmetic Decoding implementation."""
    # [G] Standard arithmetic decoding implementation required here.
    pass
```

### Parameters
*   **Image Depth:** 8-bit (values 0-255) [S1].
*   **Background Intensity:** Assumed to be 0 (zero-intensity) [S1].
*   **Dictionary Size:** Standard LZW initial size of 256 entries (for 8-bit data) [S1].

### Edge Cases and Pitfalls
*   **Empty Volume:** If the volume contains no non-zero voxels, the VOI is empty. The method should handle this by sending an empty bit stream and metadata indicating no active region [S1].
*   **Precision:** The method is lossless; therefore, MSE is exactly 0, PSNR is infinite, and SSIM is 1 [S1].
*   **Boundary Conditions:** The bounding box calculation must strictly include all non-zero voxels. Missing a single non-zero voxel results in data loss during reconstruction [S1].
*   **Blank Slices:** Slices at the foremost and end that are entirely zero must be correctly identified and excluded from the VOI extraction to maximize compression [S1].

## Operating characteristics

### Time and Memory Complexity
*   **Time Complexity:** Dominated by the LZW and Arithmetic Coding stages on the VOI. The SBV extraction requires a single pass or multiple passes to find bounds.
    *   Encoding times reported: 19.23s to 445.20s depending on volume size [S1].
    *   Decoding times reported: 3.17s to 50.28s [S1].
*   **Memory Complexity:** Requires storing the original volume, the extracted VOI, and the LZW dictionary. The dictionary size grows with the complexity of the VOI data [S1].

### Measured Performance
Performance was tested on 3D medical images (MRI, CT, PET) from TCIA, IBSR, and Radiopedia [S1].

| 3D Image | Dimensions | CR | BPV | Enc Time (s) | Dec Time (s) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Image 1 | 128x128x27 | 3.22 | 2.48 | 19.23 | 3.17 |
| Image 2 | 256x256x99 | 2.16 | 3.70 | 408.01 | 48.99 |
| Image 4 | 256x256x63 | 12.57 | 0.64 | 38.94 | 5.58 |
| Image 6 | 256x256x63 | 15.61 | 0.51 | 31.88 | 5.71 |
| Image 12 | 256x256x298 | 13.66 | 0.59 | 204.98 | 29.66 |

*   **Compression Ratio (CR):** Ranges from 2.16 to 15.61 [S1].
*   **Bits Per Voxel (BPV):** Ranges from 0.51 to 3.70 [S1].
*   **Comparison:** The method yields almost double the compression ratio of standard methods like Huffman, RLC, LZW, and Arithmetic Coding applied directly to the full volume [S1].
*   **VOI Impact:** Using VOI extraction increases CR by an average of 7.26 and reduces Bit Rate by an average of 1.77 bits compared to methods without VOI extraction [S1].

### Strengths and Limitations
*   **Strengths:** High compression ratios for sparse medical images; lossless fidelity; reduced computation time for decoding compared to full-volume encoding [S1].
*   **Limitations:** Performance depends heavily on the size of the background; larger backgrounds yield better compression. The method is specific to 3D medical images with zero-intensity backgrounds [S1].

## Background

### Origin
The 3D-VOI-LtoA method was proposed in a study on hybrid lossless compression techniques for three-dimensional medical images [S1]. It combines object-based features (VOI extraction) with statistical-based coding (LZW and Arithmetic Coding) [S1].

### History and Development
The method addresses the need for efficient transmission of high-quality medical data for telemedicine and remote diagnosis [S1]. It builds upon the concept of Region of Interest (ROI) or Volume of Interest (VOI) coding, which separates clinically important objects from background [S1]. The "L to A" codec specifically refers to the pipeline of LZW followed by Arithmetic Coding [S1].

### Usage
It is used for compressing 3D medical images (MRI, CT, PET) from repositories such as the Cancer Imaging Archive (TCIA), Internet Brain Segmentation Repository (IBSR), and Radiopedia [S1].

## Sources

[S1] On a hybrid lossless compression technique for three‐dimensional medical images, https://doi.org/10.1002/acm2.12960, fetched 2026-11-08
