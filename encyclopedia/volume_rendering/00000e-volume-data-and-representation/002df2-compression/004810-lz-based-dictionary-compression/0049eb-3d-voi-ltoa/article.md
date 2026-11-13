# 3D-VOI-LtoA

Also called: Hybrid Lossless Compression with Selective Bounding Volume, L to A codec, L to A coding with VOI extraction, Object-based 3D medical image compression, three-dimensional-volume of interest-LtoA Method

## Overview
3D-VOI-LtoA is a lossless compression method for 3D medical images that separates the clinically significant foreground (Volume of Interest, VOI) from the zero-intensity background using a Selective Bounding Volume (SBV) technique, then encodes the VOI using a hybrid Lempel-Ziv-Welch (LZW) followed by Arithmetic Coding pipeline [S1]. It distinguishes itself from sibling methods like [LZV Algorithm](8069-lzv-algorithm.md) by explicitly removing the large, uninformative background regions prior to dictionary-based encoding, thereby reducing the data volume processed by the entropy coder and significantly increasing compression ratios for sparse medical volumes [S1].

## How it works

### Inputs and Outputs
*   **Input:** A 3D medical image volume (e.g., MRI, CT, PET) with 8-bit depth per voxel [S1]. The volume is assumed to have a zero-intensity background [S1].
*   **Output:** A compressed bitstream containing the encoded VOI data and the metadata required to reconstruct the original volume dimensions and background [S1].

### Data Structures
The method does not define complex external data structures but relies on the following internal representations during processing:
1.  **Bounding Volume Coordinates:** A set of global coordinates defining the axis-aligned bounding box (AABB) of the VOI. This includes the minimum and maximum indices along the X, Y, and Z axes, as well as the identification of any "null" (blank) slices at the foremost and end of the volume that are excluded from the bounding box [S1].
2.  **VOI Sub-volume:** The 3D array of voxel values extracted from the original volume using the Bounding Volume coordinates [S1].
3.  **LZW Dictionary:** Standard LZW dictionary mapping sequences of symbols to codewords [S1].
4.  **Arithmetic Coder State:** The probability model and current interval state for arithmetic encoding [S1].

### Steps

#### Phase 1: VOI Extraction (Selective Bounding Volume)
1.  **Input:** Original 3D volume $V$ of size $D_x \times D_y \times D_z$.
2.  **Identify Null Slices:** Scan the volume along the depth axis (Z) to identify slices that contain only zero-intensity voxels. These are considered "blank" or "null" slices [S1].
3.  **Determine Active Bounds:**
    *   Find the first slice index $z_{min}$ and last slice index $z_{max}$ that contain at least one non-zero voxel [S1].
    *   For each slice $z$ in $[z_{min}, z_{max}]$, determine the bounding box of non-zero voxels within that slice. Let $x_{min}(z), x_{max}(z), y_{min}(z), y_{max}(z)$ be the min/max coordinates for slice $z$ [S1].
4.  **Construct Global Bounding Volume:**
    *   Compute the global minimum and maximum coordinates for the entire VOI:
        $$ X_{min} = \min_{z \in [z_{min}, z_{max}]} x_{min}(z) $$
        $$ X_{max} = \max_{z \in [z_{min}, z_{max}]} x_{max}(z) $$
        $$ Y_{min} = \min_{z \in [z_{min}, z_{max}]} y_{min}(z) $$
        $$ Y_{max} = \max_{z \in [z_{min}, z_{max}]} y_{max}(z) $$
    *   The VOI is defined by the sub-volume $V_{VOI}$ spanning indices $[X_{min}, X_{max}] \times [Y_{min}, Y_{max}] \times [z_{min}, z_{max}]$ [S1].
5.  **Extract VOI:** Extract the voxel data for $V_{VOI}$ from $V$ [S1].
6.  **Store Metadata:** Record the original volume dimensions ($D_x, D_y, D_z$) and the Bounding Volume coordinates ($X_{min}, X_{max}, Y_{min}, Y_{max}, z_{min}, z_{max}$) to be sent alongside the compressed data [S1].

#### Phase 2: Hybrid Encoding (L to A Codec)
1.  **LZW Encoding:**
    *   Initialize the LZW dictionary with all possible single-symbol codes (e.g., 0-255 for 8-bit data) [S1].
    *   Parse the extracted VOI data (flattened or traversed in a consistent order, e.g., row-major) to find the longest match in the dictionary.
    *   Output the codeword for the longest match.
    *   Add the new sequence (longest match + next symbol) to the dictionary [S1].
    *   Repeat until all symbols in the VOI are processed [S1].
2.  **Arithmetic Encoding:**
    *   Take the sequence of LZW codewords generated in the previous step.
    *   Encode this sequence using an Arithmetic Encoder. The encoder requires a probability model for the LZW codewords. [G: In standard implementations, this is often a static model based on the alphabet size or a dynamic model updated during encoding. The source does not specify the exact probability distribution method, so a standard adaptive arithmetic coding approach is assumed for implementation].
    *   Output the final compressed bitstream [S1].

#### Phase 3: Decoding and Reconstruction
1.  **Input:** Compressed bitstream and Bounding Volume metadata [S1].
2.  **Arithmetic Decoding:**
    *   Decode the bitstream using the inverse Arithmetic Decoder to recover the sequence of LZW codewords [S1].
3.  **Inverse LZW:**
    *   Initialize the LZW dictionary identically to the encoder [S1].
    *   Decode the LZW codewords to reconstruct the voxel data of the VOI ($V_{VOI}$) [S1].
4.  **Reconstruction:**
    *   Create an empty 3D volume $V_{recon}$ of size $D_x \times D_y \times D_z$, initialized to zero (background) [S1].
    *   Copy the decompressed $V_{VOI}$ into $V_{recon}$ at the coordinates specified by the Bounding Volume metadata ($X_{min}, X_{max}, Y_{min}, Y_{max}, z_{min}, z_{max}$) [S1].
    *   The result is the lossless reconstruction of the original 3D medical image [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compress_3d_voi_ltoa(volume_3d):
    """
    Compresses a 3D medical image using 3D-VOI-LtoA.
    Args:
        volume_3d: 3D numpy array or list of lists of lists, 8-bit depth.
    Returns:
        compressed_bits: bytes object containing the arithmetic encoded bitstream.
        metadata: dict containing original dimensions and bounding box coordinates.
    """
    # Phase 1: VOI Extraction
    # 1. Identify null slices (slices with all zeros)
    z_min = None
    z_max = None
    for z in range(volume_3d.shape[0]):
        if not is_slice_null(volume_3d[z]):
            if z_min is None:
                z_min = z
            z_max = z
            
    # Handle case where volume is entirely background
    if z_min is None:
        return b'', {'z_min': 0, 'z_max': 0, 'x_min': 0, 'x_max': 0, 'y_min': 0, 'y_max': 0, 'dims': volume_3d.shape}

    # 2. Determine global bounding box for active slices
    x_min = float('inf')
    x_max = float('-inf')
    y_min = float('inf')
    y_max = float('-inf')

    for z in range(z_min, z_max + 1):
        slice_data = volume_3d[z]
        # Find bounding box of non-zero pixels in this slice
        s_x_min, s_x_max, s_y_min, s_y_max = get_slice_bbox(slice_data)
        
        if s_x_min < x_min: x_min = s_x_min
        if s_x_max > x_max: x_max = s_x_max
        if s_y_min < y_min: y_min = s_y_min
        if s_y_max > y_max: y_max = s_y_max

    # 3. Extract VOI
    voi_data = extract_sub_volume(volume_3d, x_min, x_max, y_min, y_max, z_min, z_max)
    
    # Flatten VOI for encoding (e.g., row-major order)
    voi_flat = flatten_3d(voi_data)

    # Phase 2: Hybrid Encoding
    # 2a. LZW Encoding
    lzw_codes = lzw_encode(voi_flat)
    
    # 2b. Arithmetic Encoding
    # Note: Standard arithmetic coding requires a probability model.
    # Assuming a simple uniform model or adaptive model as per standard LtoA implementations.
    compressed_bits = arithmetic_encode(lzw_codes)

    # Metadata for reconstruction
    metadata = {
        'dims': volume_3d.shape,
        'z_min': z_min,
        'z_max': z_max,
        'x_min': x_min,
        'x_max': x_max,
        'y_min': y_min,
        'y_max': y_max
    }
    
    return compressed_bits, metadata

def decompress_3d_voi_ltoa(compressed_bits, metadata):
    """
    Decompresses a 3D medical image using 3D-VOI-LtoA.
    Args:
        compressed_bits: bytes object from encoder.
        metadata: dict from encoder.
    Returns:
        reconstructed_volume: 3D numpy array.
    """
    # Phase 3: Decoding and Reconstruction
    # 3a. Arithmetic Decoding
    lzw_codes = arithmetic_decode(compressed_bits)
    
    # 3b. Inverse LZW
    voi_flat = lzw_decode(lzw_codes)
    
    # 3c. Reshape VOI
    voi_data = reshape_3d(voi_flat, 
                          metadata['x_max'] - metadata['x_min'] + 1,
                          metadata['y_max'] - metadata['y_min'] + 1,
                          metadata['z_max'] - metadata['z_min'] + 1)
                          
    # 3d. Reconstruct full volume
    dims = metadata['dims']
    reconstructed_volume = np.zeros(dims, dtype=np.uint8)
    
    # Place VOI into the correct position
    reconstructed_volume[
        metadata['z_min']:metadata['z_max']+1,
        metadata['y_min']:metadata['y_max']+1,
        metadata['x_min']:metadata['x_max']+1
    ] = voi_data
    
    return reconstructed_volume

def is_slice_null(slice_data):
    return np.all(slice_data == 0)

def get_slice_bbox(slice_data):
    # Returns x_min, x_max, y_min, y_max for non-zero elements
    coords = np.argwhere(slice_data != 0)
    if len(coords) == 0:
        return 0, 0, 0, 0
    # Assuming slice_data is 2D: rows=y, cols=x
    y_coords = coords[:, 0]
    x_coords = coords[:, 1]
    return np.min(x_coords), np.max(x_coords), np.min(y_coords), np.max(y_coords)

def extract_sub_volume(vol, x_min, x_max, y_min, y_max, z_min, z_max):
    return vol[z_min:z_max+1, y_min:y_max+1, x_min:x_max+1]

def flatten_3d(voi):
    return voi.flatten()

def reshape_3d(flat, d1, d2, d3):
    return flat.reshape((d1, d2, d3))

def lzw_encode(data):
    # Standard LZW implementation
    dictionary = {chr(i): i for i in range(256)}
    w = ""
    result = []
    for k in data:
        wk = w + k
        if wk in dictionary:
            w = wk
        else:
            result.append(dictionary[w])
            dictionary[wk] = len(dictionary)
            w = k
    if w:
        result.append(dictionary[w])
    return result

def lzw_decode(codes):
    # Standard Inverse LZW implementation
    dictionary = {i: chr(i) for i in range(256)}
    result = []
    w = chr(codes[0])
    result.append(w)
    for k in codes[1:]:
        if k in dictionary:
            entry = dictionary[k]
        elif k == len(dictionary):
            entry = w + w[0]
        else:
            raise ValueError("Bad compressed k: %s" % k)
        result.append(entry)
        dictionary[len(dictionary)] = w + entry[0]
        w = entry
    return [ord(c) for c in "".join(result)]

def arithmetic_encode(codes):
    # Placeholder for standard arithmetic encoding logic
    # Requires probability model. For implementation, use a standard library or 
    # adaptive arithmetic coder.
    pass

def arithmetic_decode(bits):
    # Placeholder for standard arithmetic decoding logic
    pass
```

### Parameters
*   **Image Depth:** 8-bit per voxel [S1].
*   **Background Intensity:** Assumed to be 0 (zero-intensity) [S1].
*   **LZW Dictionary Size:** Not explicitly limited in the source, but standard LZW implementations typically use 12-bit or 16-bit codewords. The source does not specify [S1].
*   **Arithmetic Coding Model:** Not specified in the source [S1].

### Edge Cases and Pitfalls
*   **Entirely Background Volume:** If the volume contains no non-zero voxels, the VOI extraction phase yields an empty VOI. The implementation must handle this by sending an empty bitstream and appropriate metadata [S1].
*   **Null Slices:** The method explicitly ignores blank slices at the foremost and end of the volume. If the volume has no null slices, the bounding box covers the entire depth [S1].
*   **Lossless Nature:** Since the method is lossless, the Mean Squared Error (MSE) is exactly 0, and the Peak Signal-to-Noise Ratio (PSNR) is theoretically infinite. The Structural Similarity Index (SSIM) is exactly 1.0 for all tested volumes [S1].

## Operating characteristics

### Time and Memory Complexity
*   **Time Complexity:** The complexity is dominated by the LZW and Arithmetic encoding steps on the VOI data. The VOI extraction is linear with respect to the volume size. The source provides specific timing results but no asymptotic complexity analysis [S1].
*   **Memory Complexity:** Requires memory for the original volume, the extracted VOI, and the LZW dictionary. The dictionary size grows with the complexity of the VOI data [S1].

### Measured Performance
The following results were reported for 8-bit 3D medical images [S1]:

| 3D Image | Dimensions | Type | CR | BPV | Enc Time (s) | Dec Time (s) |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| 1 | 128x128x27 | MRI | 3.22 | 2.48 | 19.23 | 3.17 |
| 2 | 256x256x99 | CT | 2.16 | 3.70 | 408.01 | 48.99 |
| 4 | 256x256x63 | MRI | 12.57 | 0.64 | 38.94 | 5.58 |
| 6 | 256x256x63 | MRI | 15.61 | 0.51 | 31.88 | 5.71 |
| 12 | 256x256x298 | PET | 13.66 | 0.59 | 204.98 | 29.66 |

*   **Compression Ratio (CR):** Ranges from 2.16 to 15.61.
*   **Bits Per Voxel (BPV):** Ranges from 0.51 to 3.70 bits/voxel.
*   **Comparison:** The method yields almost double the compression ratio of standard methods like Huffman, RLC, LZW, and Arithmetic Coding applied to the full volume [S1].
*   **VOI Impact:** Using VOI extraction increases the CR by an average of 7.26 and reduces the Bit Rate by an average of 1.77 bits compared to methods without VOI extraction [S1].

### Strengths and Limitations
*   **Strengths:**
    *   High compression ratios for sparse medical images due to background removal [S1].
    *   Lossless reconstruction ensures diagnostic fidelity [S1].
    *   Reduced computation time for decoding compared to encoding the full volume [S1].
*   **Limitations:**
    *   Assumes a zero-intensity background. If the background is not zero or contains noise, the SBV method may fail to correctly separate the VOI, leading to reconstruction errors or reduced compression [S1].
    *   Performance is highly dependent on the size of the VOI relative to the total volume. Smaller VOIs yield higher compression [S1].
    *   Encoding time can be significant for large volumes (e.g., 408s for Image 2) [S1].

### When to Use
*   Use for compressing 3D medical images (MRI, CT, PET) where the background is predominantly zero and lossless reconstruction is required for telemedicine or remote diagnosis [S1].
*   Use when the VOI is small relative to the total volume to maximize compression benefits [S1].

### When Not to Use
*   Do not use if the background is not zero or contains significant non-zero noise, as the SBV method relies on this assumption [S1].
*   Do not use if the VOI occupies a large portion of the volume, as the overhead of SBV processing may not yield significant compression gains over standard methods [S1].

## Background

### Origin
The 3D-VOI-LtoA method was proposed in a study on hybrid lossless compression techniques for three-dimensional medical images [S1]. The work aims to improve compression performance for telemedicine applications by combining object-based features (VOI extraction) with statistical-based coding (LZW + Arithmetic Coding) [S1].

### History and Development
The method builds on the principles of object-based image compression, where the Region of Interest (ROI) or Volume of Interest (VOI) is separated from the background to improve coding efficiency [S1]. The "L to A" codec specifically refers to the hybrid pipeline of Lempel-Ziv-Welch (LZW) followed by Arithmetic Coding, which leverages the redundancy reduction of LZW and the entropy efficiency of Arithmetic Coding [S1].

### Usage
The method has been tested on 3D brain volumes from repositories such as the Cancer Imaging Archive (TCIA), Internet Brain Segmentation Repository (IBSR), and Radiopedia [S1]. It is applicable to MRI, CT, and PET images [S1].

## Sources

[S1] On a hybrid lossless compression technique for three‐dimensional medical images, https://doi.org/10.1002/acm2.12960, fetched 2026-11-08
