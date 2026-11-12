# 3D Integer Wavelet Transform


## Overview
The 3D Integer Wavelet Transform (3D-IWT) is a lossless compression method that maps integer input data to integer coefficients, ensuring perfect reconstruction with finite precision arithmetic [S1]. It sets itself apart from standard floating-point wavelet transforms by embedding rounding operations directly into the filter decomposition, thereby avoiding precision loss and infinite precision requirements [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3-D volume of integer data (e.g., medical images) [S1].
*   **Output:** A set of 3-D frequency sub-bands containing integer wavelet coefficients [S1].

**Data Structures:**
The transform decomposes the input volume into eight 3-D frequency sub-bands at each level $r$: $LLL_r$ (approximation), and seven detail sub-bands ($LLH_r, LHL_r, LHH_r, HLL_r, HLH_r, HHL_r, HHH_r$) [S1]. The $LLL_r$ sub-band is a coarser version of the original image, while the others represent details [S1].

**Steps:**
1.  **Separable Decomposition:** Perform a separable wavelet decomposition using the same wavelet filters in all three dimensions [S1].
2.  **2-D Spatial Transform:** For each image slice (2-D plane), perform a 2-D dyadic wavelet decomposition [S1].
3.  **1-D Temporal Transform:** Perform a 1-D wavelet packet decomposition along the resulting image slices (across the depth dimension) [S1].
4.  **Iteration:** Iterate the decomposition on the approximation low-pass sub-band ($LLL$) [S1].

**Formulas:**
The transform uses a lifting-based scheme where smooth ($s$) and detail ($d$) outputs for an index $n$ are calculated as [S1]:
$$ s(n) = \text{round}\left(\frac{h_0(n) + h_1(n)}{2}\right) $$
$$ d(n) = \text{round}\left(\frac{h_0(n) - h_1(n)}{2}\right) $$
Where $h_0(n)$ and $h_1(n)$ are intermediate values derived from the filter application [S1]. The text notes that while rounding appears to discard information, the transform is reversible [S1]. The inverse transform reconstructs the input from $s(n)$ and $d(n)$ [S1].

**Parameters:**
*   **VOI (Volume of Interest):** A cuboid defined in the spatial domain with potentially different length, width, and height values, used to prioritize specific regions for transmission or compression [S1].

**Edge Cases and Pitfalls:**
*   **Boundary Effects:** Non-orthogonality of the wavelet transform and the symmetric extension method used beyond the last slice can cause PSNR drops [S2].
*   **GOP Boundaries:** When different Groups of Pictures (GOPs) are separately transformed, PSNR drops occur at GOP boundaries [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def iwt_3d(volume):
    """
    Performs a 3D Integer Wavelet Transform on an integer volume.
    volume: 3D list of lists of lists of integers.
    Returns: Dictionary of sub-bands (LLL, LLH, LHL, LHH, HLL, HLH, HHL, HHH).
    """
    depth = len(volume)
    height = len(volume[0])
    width = len(volume[0][0])
    
    # Step 1 & 2: 2D Dyadic Wavelet Decomposition on each slice
    slices_2d = []
    for z in range(depth):
        slice_data = volume[z]
        ll, lh, hl, hh = iwt_2d_slice(slice_data)
        slices_2d.append([ll, lh, hl, hh])
    
    # Step 3: 1D Wavelet Packet Decomposition along the slices
    sub_bands = {}
    spatial_indices = ['LL', 'LH', 'HL', 'HH']
    
    for idx in spatial_indices:
        arr_3d = []
        for z in range(depth):
            arr_3d.append(slices_2d[z][spatial_indices.index(idx)])
        
        low_pass, high_pass = iwt_1d_depth(arr_3d)
        
        if idx == 'LL':
            sub_bands['LLL'] = low_pass
            sub_bands['LLH'] = high_pass
        elif idx == 'LH':
            sub_bands['LHL'] = low_pass
            sub_bands['LHH'] = high_pass
        elif idx == 'HL':
            sub_bands['HLL'] = low_pass
            sub_bands['HLH'] = high_pass
        elif idx == 'HH':
            sub_bands['HHL'] = low_pass
            sub_bands['HHH'] = high_pass
            
    # Step 4: Iterate on LLL (recursive call for multi-level decomposition)
    if len(sub_bands['LLL']) > 1:
        sub_bands['LLL'] = iwt_3d(sub_bands['LLL'])
        
    return sub_bands

def iwt_1d_depth(arr_3d):
    """
    Performs 1D IWT along the depth axis (first dimension of arr_3d).
    arr_3d: list of 2D arrays.
    Returns: (low_pass, high_pass) lists of 2D arrays.
    """
    depth = len(arr_3d)
    height = len(arr_3d[0])
    width = len(arr_3d[0][0])
    
    low_pass = []
    high_pass = []
    
    for z in range(0, depth, 2):
        z_low = z
        z_high = z + 1 if z + 1 < depth else z # Symmetric extension for boundary [G]
        
        lp_slice = []
        hp_slice = []
        
        for y in range(height):
            row_lp = []
            row_hp = []
            for x in range(width):
                h0 = arr_3d[z_low][y][x]
                h1 = arr_3d[z_high][y][x]
                # Apply lifting formula [S1]
                s = round((h0 + h1) / 2)
                d = round((h0 - h1) / 2)
                row_lp.append(s)
                row_hp.append(d)
            lp_slice.append(row_lp)
            hp_slice.append(row_hp)
        low_pass.append(lp_slice)
        high_pass.append(hp_slice)
        
    return low_pass, high_pass

def iwt_2d_slice(slice_data):
    """
    Performs 2D IWT on a single slice (list of lists).
    Returns: (LL, LH, HL, HH) lists of lists.
    """
    height = len(slice_data)
    width = len(slice_data[0])
    
    # Apply 1D IWT along rows (width)
    ll_row, lh_row = [], []
    for y in range(height):
        lp, hp = iwt_1d_row(slice_data[y])
        ll_row.append(lp)
        lh_row.append(hp)
        
    # Apply 1D IWT along columns (height) on the results
    ll_col, lh_col = [], []
    hl_col, hh_col = [], []
    
    for x in range(width):
        col_lp = [ll_row[y][x] for y in range(height)]
        col_hp = [lh_row[y][x] for y in range(height)]
        
        lp_lp, hp_lp = iwt_1d_row(col_lp)
        lp_hp, hp_hp = iwt_1d_row(col_hp)
        
        for y in range(len(lp_lp)):
            if len(ll_col) <= y:
                ll_col.append([])
                lh_col.append([])
                hl_col.append([])
                hh_col.append([])
            ll_col[y].append(lp_lp[y])
            lh_col[y].append(lp_hp[y])
            hl_col[y].append(hp_lp[y])
            hh_col[y].append(hp_hp[y])
            
    return ll_col, lh_col, hl_col, hh_col

def iwt_1d_row(row):
    """
    Performs 1D IWT on a 1D list.
    Returns: (low_pass, high_pass) lists.
    """
    n = len(row)
    lp = []
    hp = []
    for i in range(0, n, 2):
        h0 = row[i]
        h1 = row[i+1] if i+1 < n else row[i] # Symmetric extension [G]
        s = round((h0 + h1) / 2)
        d = round((h0 - h1) / 2)
        lp.append(s)
        hp.append(d)
    return lp, hp
```

## Subtopics

### [Lifting-Based Integer Wavelet Transform Construction](8255-lifting-based-integer-wavelet-transform-construction.md)
This method constructs integer-preserving wavelet transforms by approximating traditional non-integer wavelet filters using the lifting scheme with rounding operations. It solves the problem of perfect reconstruction in lossless compression by ensuring that integer input data remains integer throughout the transform, avoiding the precision loss and infinite precision requirements of floating-point arithmetic. It sets itself apart from standard floating-point wavelet transforms by embedding the rounding steps directly into the filter decomposition, allowing for exact invertibility with finite precision arithmetic.

### Memory-constrained 3-D Integer Wavelet Transform
A technique to implement the 3-D integer wavelet transform with limited memory, which eliminates PSNR drops across GOP boundaries [S2]. It addresses the pitfall where PSNR drops occur at GOP boundaries due to the nonorthogonality of the wavelet transform and the symmetric extension method used beyond the last slice when different GOPs are separately transformed [S2]. The memory-constrained transform eliminates these PSNR drops across GOP boundaries, except at the beginning and end of the data volume [S2].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources, though a memory-constrained variant exists [S2].
*   **Measured Performance:** 3-D SPIHT (which uses 3D-IWT) is 7.65 and 6.21 dB better than 2-D SPIHT at 0.1 and 0.5 b/p, respectively [S2]. 3-D ESCOT gains an additional 0.69 and 0.93 dB over 3-D SPIHT [S2]. Compared with 3-D CB-EZW, lossy coding results from 3-D SPIHT and 3-D ESCOT are 2.6–3.0 and 3.3–4.0 dB better at 0.1 and 0.5 b/p, respectively [S2].
*   **Strengths:** Perfect reconstruction with finite precision arithmetic; suitable for lossless compression; supports VOI coding for scalable bit-streams [S1].
*   **Limitations:** PSNR drops can occur at boundaries (GOP or volume ends) if not handled by memory-constrained techniques [S2].
*   **When to use:** Interactive telemedicine applications where different remote clients with limited bandwidth may request different VOIs of the same compressed 3-D image [S1].

## Background
The 3D-IWT is primarily concerned with interactive telemedicine applications, where efficient compression of 3-D medical images is required [S1]. The method was proposed in the context of scalable compression using EBCOT with Volume of Interest coding [S1].

## Sources
[S1] Scalable Compression of 3-D Medical Image Data Using EBCOT with Volume of Interest Coding, https://doi.org/10.51983/ajcst-2012.1.1.1673, fetched 2026-10-26
[S2] Lossy-to-lossless compression of medical volumetric data using three-dimensional integer wavelet transforms, https://doi.org/10.1109/tmi.2003.809585, fetched 2026-11-01
