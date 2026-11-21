# Temporal Sampling for Animation

Also called: Frame Extraction for Visualization, Time-range Frame Extraction

## Overview
Temporal sampling for animation extracts a discrete subset of frames from a time-series volume dataset to create a coherent animation sequence. It solves the problem of irregular or excessive temporal output by mapping a user-defined time range to the nearest available data points. Unlike [Data Pre-processing for Animation](17236-data-pre-processing-for-animation.md), which handles general normalization and preprocessing, this method specifically addresses the temporal dimension, ensuring smooth playback by selecting frames based on a fixed time step or specific interval.

## How it works
**Inputs:**
- `tomo`: Array of volume data frames, shape $(N, D)$, where $N$ is the number of available frames and $D$ is the data size per frame.
- `tomo_t`: Array of corresponding timestamps, shape $(N,)$.
- `t0`: Start time of the desired animation range.
- `t1`: End time of the desired animation range.
- `dt`: Time step (interval) between extracted frames.

**Outputs:**
- `selected_frames`: Array of volume data frames corresponding to the sampled times.
- `selected_times`: Array of timestamps for the selected frames.

**Data Structures:**
- The input arrays `tomo` and `tomo_t` are assumed to be sorted by time. If not, they must be sorted together using the indices from `argsort(tomo_t)` [S1, unofficial].
- Output arrays maintain the same data type and layout as the input `tomo`.

**Steps:**
1. **Generate Target Times:** Create an array of target times $T_{target}$ using a range from `t0` to `t1` with step `dt`. In Python, this is `np.arange(t0, t1, dt)`.
2. **Clamp Boundaries:**
   - If `t0` is less than `tomo_t[0]`, clamp `t0` to `tomo_t[0]`.
   - If `t1` is greater than `tomo_t[-1]`, clamp `t1` to `tomo_t[-1]`.
   - Regenerate `T_{target}` with clamped values if boundaries changed.
3. **Find Nearest Indices:** For each time $t$ in $T_{target}$, find the index $i$ in `tomo_t` that minimizes the absolute difference $|tomo_t[i] - t|$. This is the "nearest neighbor" in time.
4. **Extract Frames:** Append `tomo[i]` to the output list for each selected index.
5. **Return Results:** Return the extracted frames and their corresponding timestamps.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def temporal_sampling(tomo, tomo_t, t0, t1, dt):
    """
    Extract frames from a time-series volume dataset for animation.
    
    Args:
        tomo: Array of volume data, shape (N, D).
        tomo_t: Array of timestamps, shape (N,).
        t0: Start time.
        t1: End time.
        dt: Time step between frames.
        
    Returns:
        selected_frames: Array of selected volume frames.
        selected_times: Array of selected timestamps.
    """
    # Ensure inputs are sorted by time [S1, unofficial]
    sort_idx = np.argsort(tomo_t)
    tomo = tomo[sort_idx]
    tomo_t = tomo_t[sort_idx]
    
    # Clamp t0 and t1 to the available data range [S1, unofficial]
    if t0 < tomo_t[0]:
        t0 = tomo_t[0]
    if t1 > tomo_t[-1]:
        t1 = tomo_t[-1]
        
    # Generate target times [S1, unofficial]
    target_times = np.arange(t0, t1, dt)
    
    selected_frames = []
    selected_times = []
    
    for t in target_times:
        # Find index of nearest time [S1, unofficial]
        # argmin returns the index of the minimum value in the array
        idx = np.argmin(np.abs(tomo_t - t))
        
        selected_frames.append(tomo[idx])
        selected_times.append(tomo_t[idx])
        
    if len(selected_frames) == 0:
        return np.array([]), np.array([])
        
    return np.array(selected_frames), np.array(selected_times)
```

**Parameters:**
- `dt`: Typical values depend on the desired frame rate and total duration. For smooth animation, `dt` should be small enough to avoid aliasing but large enough to reduce data volume. Not given in the sources.

**Edge Cases and Pitfalls:**
- **Empty Range:** If `t0 >= t1` after clamping, no frames are selected.
- **Precision:** Floating-point comparison for `argmin` is generally robust, but if multiple timestamps are equally close, `argmin` returns the first occurrence.
- **Unsorted Input:** If `tomo_t` is not sorted, the nearest neighbor search may fail. Sorting is required [S1, unofficial].

## Subtopics

### Time Series Analysis
Automated time-domain analysis system that orders simulation outputs by time/redshift using parameter files, enabling sequential analysis across multiple datasets without manual path construction [S2]. It is chosen when dealing with cosmological simulations where data is output based on redshift lists rather than constant time intervals, causing independent naming schemes that complicate manual list construction [S2]. It differs from simple frame extraction by using parameter files to infer the temporal relationship of all datasets [S2].

## Operating characteristics
- **Time Complexity:** $O(N \log N)$ for sorting if input is unsorted, plus $O(M \cdot N)$ for finding nearest neighbors, where $N$ is the number of available frames and $M$ is the number of target frames. With sorted input and binary search, this can be reduced to $O(N + M \log N)$.
- **Memory Complexity:** $O(N + M)$ to store the input and output arrays.
- **Strengths:** Simple to implement; handles irregular time steps by snapping to nearest available data; robust to boundary conditions via clamping.
- **Limitations:** Does not interpolate between frames; may skip frames if `dt` is large relative to the time gaps in the source data.
- **When to use:** When creating animations from simulation outputs that are not uniformly spaced in time or when a specific frame rate is required.
- **When not to use:** When high temporal fidelity is required and interpolation is necessary, or when the source data is extremely large and memory is constrained.

## Background
The method is derived from practices in plasma tomography and astrophysical simulation analysis. In plasma tomography, temporal sampling is used to extract frames for visualization from time-series data [S1, unofficial]. In astrophysics, the `yt` toolkit's `TimeSeries` object provides a framework for analyzing time-ordered datasets, particularly for cosmological simulations where output times are irregular [S2].

## Sources
[S1] diogoff/plasma-tomography, https://github.com/diogoff/plasma-tomography, fetched 2026-10-27 (unofficial)
[S2] YT: A Multi-Code Analysis Toolkit for Astrophysical Simulation Data, https://digital.library.unt.edu/ark:/67531/metadc834104/?q=fluid simulation volume rendering, fetched 2026-10-28
