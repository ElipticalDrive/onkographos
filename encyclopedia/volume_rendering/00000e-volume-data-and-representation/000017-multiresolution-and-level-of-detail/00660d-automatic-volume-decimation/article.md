# Automatic Volume Decimation

Also called: volumeDecimation

## Overview
Automatic Volume Decimation is a memory-management technique that reduces the resolution of a loaded volumetric dataset when its total voxel count exceeds a configured limit, ensuring that memory usage and rendering performance remain within hardware bounds [S1, unofficial]. It differs from manual decimation or static downsampling by applying the reduction automatically upon volume load based on a global threshold, specifically targeting the trade-off between visual fidelity and system stability in interactive viewers [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A volumetric dataset (typically DICOM series) with dimensions $(W, H, D)$ representing width, height, and depth (slice count).
*   **Input:** A configuration object containing `volumeAutoDecimationThreshold` and `dangerouslyTurnOffDecimationNotification`.
*   **Output:** A decimated volume dataset with reduced dimensions $(W', H', D')$ such that the total voxel count $W' \times H' \times D' \le \text{Threshold}$.
*   **Output:** Metadata indicating the original and decimated image counts for UI display [S1, unofficial].

**Data Structures:**
The configuration is passed via a `customizationService` array. The relevant structure is:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```json
{
  "customizationService": [
    {
      "volumeDecimation": {
        "volumeAutoDecimationThreshold": <number>,
        "dangerouslyTurnOffDecimationNotification": <boolean>
      }
    }
  ]
}
```
*   `volumeAutoDecimationThreshold`: The maximum allowed voxel count. If omitted, null, or undefined, automatic decimation is disabled [S1, unofficial].
*   `dangerouslyTurnOffDecimationNotification`: A boolean flag to suppress warnings when decimation is applied. Default is `false` [S1, unofficial].

**Algorithm Steps:**
1.  **Load Volume:** Upon loading a volume dataset, calculate the estimated total voxel count $V_{total} = W \times H \times D$.
2.  **Check Threshold:** Compare $V_{total}$ against the configured `volumeAutoDecimationThreshold` ($T$).
    *   If $V_{total} \le T$, load the volume at full resolution.
    *   If $V_{total} > T$, proceed to decimation.
3.  **Calculate Decimation Factors:** Determine reduction factors for the in-plane dimensions ($W, H$) and the slice dimension ($D$). The implementation chooses these factors automatically to bring the resulting voxel count at or below $T$ [S1, unofficial]. The exact mathematical formula for distributing the reduction between in-plane and slice dimensions is not specified in the source, but the goal is to minimize visual distortion while meeting the constraint [S1, unofficial].
4.  **Apply Decimation:** Resample or subsample the volume data using the calculated factors.
5.  **Notify:** If `dangerouslyTurnOffDecimationNotification` is `false`, display a warning to the user that the volume has been decimated. Update the thumbnail image count to show both original and decimated counts [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def load_volume_with_auto_decimation(volume_data, config):
    """
    volume_data: object with attributes .width, .height, .depth
    config: object containing .customizationService array
    Returns: decimated_volume_data or original_volume_data
    """
    # 1. Extract threshold from config
    decimation_config = None
    if config.customizationService:
        for service in config.customizationService:
            if 'volumeDecimation' in service:
                decimation_config = service['volumeDecimation']
                break
    
    # 2. Check if decimation is enabled
    if decimation_config is None:
        return volume_data # No decimation config, load full res
    
    threshold = decimation_config.get('volumeAutoDecimationThreshold')
    if threshold is None:
        return volume_data # Threshold is null/undefined, load full res

    # 3. Calculate total voxels
    total_voxels = volume_data.width * volume_data.height * volume_data.depth

    # 4. Compare against threshold
    if total_voxels <= threshold:
        return volume_data

    # 5. Apply automatic decimation
    # The exact method for calculating new dimensions is not detailed in sources [S1, unofficial]
    # We assume a helper function exists that returns new dimensions
    new_width, new_height, new_depth = calculate_decimated_dimensions(
        volume_data.width, 
        volume_data.height, 
        volume_data.depth, 
        threshold
    )
    
    decimated_data = resample_volume(
        volume_data, 
        new_width, 
        new_height, 
        new_depth
    )

    # 6. Handle notification
    if not decimation_config.get('dangerouslyTurnOffDecimationNotification', False):
        show_warning("Volume was automatically decimated to fit memory limits.")

    return decimated_data

def calculate_decimated_dimensions(w, h, d, threshold):
    """
    Placeholder for the automatic factor selection logic.
    Sources state the factors are chosen automatically to meet the threshold [S1, unofficial].
    """
    # Implementation detail not provided in sources [S1, unofficial]
    # Typical approach: reduce the largest dimension first or scale all dimensions proportionally
    # until w' * h' * d' <= threshold.
    pass

def resample_volume(volume, new_w, new_h, new_d):
    """
    Resamples the volume data to the new dimensions.
    """
    # Implementation detail not provided in sources [S1, unofficial]
    pass
```

**Parameters and Typical Values:**
*   `volumeAutoDecimationThreshold`: Typical value is 500,000,000 (500 million) voxels [S1, unofficial].
*   `dangerouslyTurnOffDecimationNotification`: Default `false` [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The threshold is an integer count of voxels. If the calculated decimated volume slightly exceeds the threshold due to integer rounding in dimension reduction, the system may still load it, or the decimation logic must ensure strict inequality [S1, unofficial].
*   **Disabling:** To disable automatic decimation, the user must omit the key or set it to `null`/`undefined`. Setting it to 0 would likely decimate all volumes [S1, unofficial].
*   **UI Feedback:** The thumbnail count must explicitly distinguish between original and decimated images to avoid user confusion regarding data integrity [S1, unofficial].

## Subtopics

### Simple Filter (Local Max)
A simple data reduction filter used for real-time interactive display of volumetric data (specifically Radar Cross Section data) by selecting the largest value within a specified non-overlapping window [S2, unofficial]. It reduces polygon count and increases rendering speed at the cost of minute detail [S2, unofficial]. The user specifies a window size (e.g., 5° x 5°); the filter selects the max value in the current window and moves to the next non-overlapping position [S2, unofficial]. In tested implementations, this reduced polygon count from 336,480 to 3,364, achieving a frame rate of 1.8 frames/sec (5.5 sec/frame), retaining overall shape but losing exactness [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** The decimation decision is $O(1)$ (comparison). The actual resampling cost depends on the interpolation method used, which is not specified [S1, unofficial].
*   **Memory Complexity:** Reduces memory footprint proportionally to the reduction in voxel count. For a 500M voxel threshold, volumes significantly larger than this are compressed to fit within this limit [S1, unofficial].
*   **Performance:** Prevents GPU out-of-memory errors and maintains interactive frame rates by ensuring the dataset size is manageable for the hardware [S1, unofficial].
*   **Strengths:** Automatic, requires no user intervention, configurable threshold, integrates with GPU tier detection (high/mid/low GPU settings influence related downsampling parameters) [S1, unofficial].
*   **Limitations:** Loses high-frequency detail; the specific algorithm for distributing decimation between in-plane and slice dimensions is opaque [S1, unofficial].
*   **When to use:** Interactive medical imaging viewers (e.g., OHIF) where large DICOM volumes must be loaded without crashing the browser or GPU [S1, unofficial].
*   **When not to use:** Clinical diagnosis requiring full resolution, or when the user needs precise control over which dimensions are decimated [S1, unofficial].

## Background
Automatic Volume Decimation was implemented in the OHIF Viewer and Cornerstone ecosystem to handle the loading of large DICOM volumes in web-based environments [S1, unofficial]. It addresses the challenge of web browsers and client-side GPUs having limited memory compared to desktop workstations [S1, unofficial]. The feature was introduced in Pull Request #6034 by mbellehumeur [S1, unofficial].

## Sources
[S1] feat/decimated loading, https://github.com/OHIF/Viewers/pull/6034, fetched 2026-10-27 (unofficial)
[S2] DTIC ADA259586: Scientific Visualization of Volumetric Radar Cross Section Data, https://archive.org/details/DTIC_ADA259586, fetched 2026-10-26 (unofficial)
