# VolRec

Also called: 4D real-time volumetric reconstruction, 4D spatiotemporal reconstruction

## Overview
VolRec is a volumetric reconstruction technique that generates full-resolution 3D volumes from lower-resolution acquired volumes by combining spatial interlace/interpolation with temporal interpolation using data from previous volumes [S1]. It differs from classic OCT processing by reducing acquisition resolution to increase speed, then reconstructing full resolution via 4D spatiotemporal interpolation, rather than processing full-resolution volumes directly [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Raw spectral data from lower-resolution volumes (typically $60 \times 60 \times 8192$ voxels) and previously reconstructed full-resolution volumes [S1].
*   **Output:** Full-resolution 3D volumes (typically $300 \times 300 \times 8192$ voxels) [S1].

**Data Structures:**
*   **Voxel Data Type:** 4-byte single-precision floating point [S1].
*   **Low-Resolution Volume Layout:** $60 \times 60 \times 8192$ voxels [S1].
*   **Full-Resolution Volume Layout:** $300 \times 300 \times 8192$ voxels [S1].

**Parameters:**
*   **Target Frame Rate:** 25 volumes/s [S1].
*   **Minimum Processing Rate:** 17 GigaVoxels/s [S1].
*   **Achieved Processing Rate:** 18 GigaVoxels/s [S1].

**Steps:**
1.  **Preprocessing (Stage 1):** Process raw spectral data from lower-resolution volumes using classical OCT algorithms. This includes:
    *   Subtracting the mean of each A-scan to center the data [S1].
    *   Dispersion compensation [S1].
    *   Linearization [S1].
    *   Performing Fast Fourier Transform (FFT) per A-scan [S1].
2.  **Reconstruction (Stage 2):** Reconstruct the low-resolution preprocessed volumes into final full-resolution volumes. This involves:
    *   Applying interlace operations to increase spatial resolution [S1].
    *   Applying interpolation operations [S1].
    *   Leveraging temporal coherence by using data from previous volumes to approximate current data [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def volrec_process(raw_low_res_volumes, previous_full_res_volumes):
    """
    Processes raw low-resolution OCT data into full-resolution volumes.
    
    Args:
        raw_low_res_volumes: List of raw spectral data arrays, shape (N, 60, 60, 8192)
        previous_full_res_volumes: List of previously reconstructed full-res volumes, shape (N, 300, 300, 8192)
        
    Returns:
        full_res_volumes: List of reconstructed full-res volumes, shape (N, 300, 300, 8192)
    """
    full_res_volumes = []
    
    for i in range(len(raw_low_res_volumes)):
        # Stage 1: Preprocessing
        # Get raw data for current volume
        raw_data = raw_low_res_volumes[i]
        
        # Preprocess using classical OCT algorithms
        # 1. Subtract mean of each A-scan
        # Assuming raw_data is structured as (A-scans, B-scans, Samples) or similar
        # The text specifies "subtract the mean of each A-scan"
        processed_data = subtract_mean_per_ascan(raw_data)
        
        # 2. Dispersion compensation
        processed_data = apply_dispersion_compensation(processed_data)
        
        # 3. Linearization
        processed_data = apply_linearization(processed_data)
        
        # 4. FFT per A-scan
        # The text states FFT is performed per A-scan
        preprocessed_volume = fft_per_ascan(processed_data)
        
        # Stage 2: Reconstruction
        # Reconstruct low-res preprocessed volume to full-res
        # This involves interlace and interpolation operations
        # Leveraging temporal coherence from previous volumes
        
        # Get previous full-res volume for temporal coherence
        prev_volume = previous_full_res_volumes[i-1] if i > 0 else None
        
        # Apply interlace and interpolation using temporal data
        # The exact mathematical formula for the 4D spatiotemporal interpolation is not provided in the sources
        # [G] Assuming a standard spatiotemporal interpolation kernel that combines spatial upscaling
        # with temporal weighting from the previous frame to approximate the current high-res details.
        full_res_volume = spatiotemporal_interpolation(preprocessed_volume, prev_volume)
        
        full_res_volumes.append(full_res_volume)
        
    return full_res_volumes

def subtract_mean_per_ascan(data):
    """Subtracts the mean of each A-scan from the data."""
    # Assuming data is shaped (num_a_scans, num_b_scans, num_samples)
    # Mean is calculated along the B-scan and sample dimensions for each A-scan
    mean = np.mean(data, axis=(1, 2), keepdims=True)
    return data - mean

def apply_dispersion_compensation(data):
    """Applies dispersion compensation to the spectral data."""
    # [G] Standard dispersion compensation involves phase correction in the spectral domain.
    # The exact implementation details are not provided in the sources.
    # This is a placeholder for the specific algorithm used in the referenced work.
    return data # Placeholder

def apply_linearization(data):
    """Applies linearization to the spectral data."""
    # [G] Linearization maps the non-uniformly sampled k-space data to uniform spacing.
    # The exact implementation details are not provided in the sources.
    return data # Placeholder

def fft_per_ascan(data):
    """Performs FFT along the sample dimension for each A-scan."""
    # Assuming data is shaped (num_a_scans, num_b_scans, num_samples)
    # FFT is performed along the last axis (samples)
    return np.fft.fft(data, axis=2)

def spatiotemporal_interpolation(low_res_volume, prev_full_res_volume):
    """
    Reconstructs full-resolution volume from low-resolution preprocessed volume
    using spatial interlace/interpolation and temporal coherence from previous volume.
    """
    # [G] The sources describe the use of "interlace and interpolation operations"
    # and "leveraging temporal coherence" but do not provide the specific mathematical formula.
    # A typical approach would involve:
    # 1. Spatial upscaling (e.g., interpolation) of the low-res volume to the target resolution.
    # 2. Combining this with the previous full-res volume using a temporal weighting factor
    #    to preserve details and reduce noise/artifacts, exploiting frame-to-frame coherence.
    
    # Placeholder for the actual 4D reconstruction algorithm
    # The exact method is not detailed in the provided sources.
    return low_res_volume # Placeholder
```

**Edge Cases and Pitfalls:**
*   **Precision:** Single-precision floating point is used for voxel data, which matches the raw data type and is needed for FFT calculation [S1].
*   **Boundaries:** The method relies on temporal coherence, which assumes minor changes between consecutive frames. This may fail in cases of rapid motion or discontinuities [S1].
*   **Degenerate Input:** The method requires data from previous volumes. The first volume in a sequence may lack a previous frame for temporal interpolation [G].

## Subtopics

### [VolRec Interlacing](20802-volrec-interlacing.md)
Operation that creates a full-resolution volume at each timestamp by merging lower-resolution volumes from the current and previous epochs using a constant stride and dynamic offset.

### [VolRec Cumulative Processing](20895-volrec-cumulative-processing.md)
(No details provided in sources)

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Achieved processing rate of 18 GigaVoxels/s, representing approximately a 72× speedup over a parallel CPU-based algorithm [S1].
*   **Hardware:** Implemented using CUDA on GPUs [S1].
*   **Resolution:** Low-resolution acquisition: $60 \times 60 \times 8192$ voxels; Full-resolution target: $300 \times 300 \times 8192$ voxels [S1].
*   **Scaling:** The method exploits data-level parallelism inherent to OCT data volumes [S1].
*   **Strengths:** Enables real-time visualization (25 volumes/s) by reducing acquisition resolution and reconstructing full resolution via 4D spatiotemporal interpolation [S1].
*   **Limitations:** Requires temporal coherence, which may be affected by rapid motion [S1].
*   **When to use:** OCT systems requiring real-time visualization during surgical interventions or exams with restless patients, and for telemedicine techniques like telesurgery or image-guided medical robots [S1].
*   **When not to use:** Not given in the sources.

## Background
VolRec was developed by Arturo Vicente-Jaén, Juan Mompeán, Juan L. Aragón, and Pablo Artal from the Laboratorio de Óptica, Universidad de Murcia, and the Departamento de Ingeniería y Tecnología de Computadores, Universidad de Murcia [S1]. The method is designed for Optical Coherence Tomography (OCT) systems, which are commonly based on a light source whose output is split into two arms [S1]. The typical 3D volume is composed of 700 million voxels whose acquisition takes around 2 seconds [S1]. The goal of VolRec is to maintain maximum resolution but improve acquisition time by reducing the resolution of the acquired 3D volumes and applying intelligent processing to recover the original resolution [S1]. The use of data from previous volumes to approximate current data is a novel approach for OCT systems but well known in the fields of video compression and real-time graphics rendering, commonly referred to as frame-to-frame temporal coherence [S1].

## Sources
[S1] VolRec: 4D real-time volumetric reconstruction of OCT data, https://doi.org/10.1007/s11227-025-06969-6, fetched 2026-10-26
