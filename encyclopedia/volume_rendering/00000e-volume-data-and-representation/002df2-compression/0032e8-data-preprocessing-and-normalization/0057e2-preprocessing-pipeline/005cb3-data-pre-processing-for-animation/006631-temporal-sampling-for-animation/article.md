# Temporal Sampling for Animation

Also called: Frame Extraction for Visualization, Temporal Aliasing Avoidance, Time-range Frame Extraction

## Overview
Temporal sampling for animation determines the appropriate sampling rate (key-frames per period) for animation to avoid temporal aliasing and misleading behavior [S2, unofficial]. It solves the problem of insufficient temporal resolution in high-performance simulations by mapping a user-defined time range to the nearest available data points [S1, unofficial]. Unlike [Data Pre-processing for Animation](17236-data-pre-processing-for-animation.md), which handles general normalization and preprocessing, this method specifically addresses the temporal dimension, ensuring smooth playback by selecting frames based on a fixed time step or specific interval [S1, unofficial].

## How it works
**Inputs:**
- `tomo`: Array of volume data frames, shape $(N, D)$, where $N$ is the number of available frames and $D$ is the data size per frame [S1, unofficial].
- `tomo_t`: Array of corresponding timestamps, shape $(N,)$ [S1, unofficial].
- `t0`: Start time of the desired animation range [S1, unofficial].
- `t1`: End time of the desired animation range [S1, unofficial].
- `dt`: Time step (interval) between extracted frames [S1, unofficial].

**Outputs:**
- `selected_frames`: Array of volume data frames corresponding to the sampled times [S1, unofficial].
- `selected_times`: Array of timestamps for the selected frames [S1, unofficial].

**Data Structures:**
- The input arrays `tomo` and `tomo_t` are assumed to be sorted by time; if not, they must be sorted together using the indices from `argsort(tomo_t)` [S1, unofficial].
- Output arrays maintain the same data type and layout as the input `tomo` [S1, unofficial].

**Steps:**
1. Define time range $t_0$ to $t_1$ with step $dt$ [S1, unofficial].
2. Clamp $t_0$ to `tomo_t[0]` if smaller, and $t_1$ to `tomo_t[-1]` if larger [S1, unofficial].
3. For each $t$ in `np.arange(t0, t1, dt)`, find index $i = \text{argmin}(|\text{tomo\_t} - t|)$ [S1, unofficial].
4. Append `tomo[i]` to the output frames list [S1, unofficial].
5. Return the extracted frames and their corresponding timestamps [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def temporal_sampling(tomo, tomo_t, t0, t1, dt):
    sort_idx = np.argsort(tomo_t)
    tomo = tomo[sort_idx]
    tomo_t = tomo_t[sort_idx]
    
    if t0 < tomo_t[0]:
        t0 = tomo_t[0]
    if t1 > tomo_t[-1]:
        t1 = tomo_t[-1]
        
    target_times = np.arange(t0, t1, dt)
    
    selected_frames = []
    selected_times = []
    
    for t in target_times:
        idx = np.argmin(np.abs(tomo_t - t))
        selected_frames.append(tomo[idx])
        selected_times.append(tomo_t[idx])
        
    if len(selected_frames) == 0:
        return np.array([]), np.array([])
        
    return np.array(selected_frames), np.array(selected_times)
```

**Formulas:**
The required number of key-frames is computed as $N_{\text{samp}} = \frac{\tau \cdot St}{1/N_{\text{per}}}$, where $\tau$ is simulation time units, $St$ is the Strouhal number (non-dimensional frequency), and $N_{\text{per}}$ is samples per period [S2, unofficial]. This simplifies to $N_{\text{samp}} = \tau \cdot St \cdot N_{\text{per}}$ [S2, unofficial]. The Nyquist theorem requires the sampling frequency to be $\ge 2 \times$ the highest frequency [S2, unofficial].

**Parameters:**
- Laminar vortex shedding requires $\sim 10$–$20$ samples per period for accurate representation [S2, unofficial].
- Example: $\tau=100$, $St=0.21$, $N_{\text{per}}=20 \rightarrow N_{\text{samp}}=420$ [S2, unofficial].
- For non-periodic transients, sample at least $10$ key-frames per time constant (e.g., time for an elastic longitudinal wave to propagate one characteristic length) [S2, unofficial].

**Edge Cases and Pitfalls:**
- Temporal undersampling causes vortices to appear to move upstream when they should convect downstream [S2, unofficial].
- If `tomo_t` is unsorted, the nearest neighbor search fails; sorting is required [S1, unofficial].
- If `t0 >= t1` after clamping, no frames are selected [G].
- If multiple timestamps are equally close to a target time, `argmin` returns the first occurrence [G].

## Subtopics

### [Key-Frame Animation with In-Betweening](22856-key-frame-animation-with-in-betweening.md)
Key-frame animation with in-betweening generates continuous video frames (typically 30 fps) from discrete simulation data by interpolating between computed "key-frames". It solves the problem of insufficient temporal resolution in high-performance simulations by decoupling simulation cost from visual fidelity, relying on post-processing interpolation rather than physical continuity. Choose it when physical continuity is less critical than visual smoothness and simulation cost must be minimized.

### Time Series Analysis
Automated time-domain analysis system that orders simulation outputs by time/redshift using parameter files, enabling sequential analysis across multiple datasets without manual path construction [S3]. It is chosen when dealing with cosmological simulations where data is output based on redshift lists rather than constant time intervals, causing independent naming schemes that complicate manual list construction [S3]. It differs from simple frame extraction by using parameter files to infer the temporal relationship of all datasets [S3].

## Operating characteristics
- **Time Complexity:** $O(N \log N)$ for sorting if input is unsorted, plus $O(M \cdot N)$ for finding nearest neighbors, where $N$ is the number of available frames and $M$ is the number of target frames [S1, unofficial]. With sorted input and binary search, this can be reduced to $O(N + M \log N)$ [G].
- **Memory Complexity:** $O(N + M)$ to store the input and output arrays [G].
- **Strengths:** Simple to implement; handles irregular time steps by snapping to nearest available data; robust to boundary conditions via clamping [S1, unofficial].
- **Limitations:** Does not interpolate between frames; may skip frames if `dt` is large relative to the time gaps in the source data [S1, unofficial].
- **When to use:** When creating animations from simulation outputs that are not uniformly spaced in time or when a specific frame rate is required [S1, unofficial].
- **When not to use:** When high temporal fidelity is required and interpolation is necessary, or when the source data is extremely large and memory is constrained [S1, unofficial].

## Background
The method is derived from practices in plasma tomography and astrophysical simulation analysis. In plasma tomography, temporal sampling is used to extract frames for visualization from time-series data [S1, unofficial]. In astrophysics, the `yt` toolkit's `TimeSeries` object provides a framework for analyzing time-ordered datasets, particularly for cosmological simulations where output times are irregular [S3]. The method addresses temporal aliasing in volume rendering by ensuring sampling rates match physical time scales, such as vortex shedding frequencies [S2, unofficial].

## Sources
[S1] diogoff/plasma-tomography, https://github.com/diogoff/plasma-tomography, fetched 2026-10-27 (unofficial)
[S2] Visualization methods for high-resolution, transient, 3-D, finite element situations, https://digital.library.unt.edu/ark:/67531/metadc686501/?q=tensor field visualization volume, fetched 2026-10-29 (unofficial)
[S3] YT: A Multi-Code Analysis Toolkit for Astrophysical Simulation Data, https://digital.library.unt.edu/ark:/67531/metadc834104/?q=fluid simulation volume rendering, fetched 2026-10-28
