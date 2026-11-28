# Equal-Event Event Binning

Also called: Temporal blur prior binning

## Overview
Equal-Event Event Binning divides an event stream into bins containing an equal number of events to leverage the temporal blur prior for event loss optimization in HDR NeRF training [S1, unofficial]. It sets itself apart from uniform time-step binning by adapting bin boundaries to event density, ensuring that each bin contributes equally to the temporal integration of the blur model [S1, unofficial].

## How it works
**Inputs:**
- Event stream $B(t_{\text{start}}, t_{\text{end}})$, a list of events sorted by timestamp.
- Number of bins $b$.

**Outputs:**
- A sequence of $b$ bins $B(t_i, t_{i+1})$ for $i=0 \dots b-1$, where each bin contains exactly $N/b$ events (assuming $N$ is divisible by $b$).
- For each bin, the timestamps and polarities of the first and last triggered events at each pixel are extracted for calibration.

**Data Structures:**
- The event stream is stored as a list of tuples or objects: `(timestamp, x, y, polarity)`.
- Bins are represented by their start and end timestamps $(t_i, t_{i+1})$.
- Per-pixel metadata for a bin $i$ includes:
  - $t_{\text{first}}^i$: Timestamp of the first event in bin $i$ at a specific pixel.
  - $t_{\text{last}}^i$: Timestamp of the last event in bin $i$ at a specific pixel.
  - $p_{\text{first}}^i$: Polarity of the first event in bin $i$.
  - $p_{\text{last}}^i$: Polarity of the last event in bin $i$.

**Steps:**
1. **Count Total Events:** Let $N$ be the total number of events in $B(t_{\text{start}}, t_{\text{end}})$.
2. **Determine Bin Size:** Calculate the target number of events per bin: $k = N / b$. [S1, unofficial]
3. **Split Stream:** Iterate through the sorted event stream. Assign the first $k$ events to bin 0, the next $k$ events to bin 1, and so on, until the last $k$ events are assigned to bin $b-1$. [S1, unofficial]
4. **Set Boundaries:** Define $t_0 = t_{\text{start}}$ and $t_b = t_{\text{end}}$. For $i=1 \dots b-1$, set $t_i$ to the timestamp of the first event in bin $i$. [S1, unofficial]
5. **Extract Per-Pixel Extremes:** For each bin $i$ and each pixel $(x,y)$ that has events in this bin:
   - Identify the event with the minimum timestamp as the "first" event. Record its timestamp $t_{\text{first}}^i$ and polarity $p_{\text{first}}^i$.
   - Identify the event with the maximum timestamp as the "last" event. Record its timestamp $t_{\text{last}}^i$ and polarity $p_{\text{last}}^i$.
   - If a pixel has no events in bin $i$, these values are undefined for that pixel/bin pair.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def equal_event_binning(events, b):
    """
    events: list of (timestamp, x, y, polarity), sorted by timestamp.
    b: number of bins.
    Returns: list of bins, where each bin is a dict with 'start_time', 'end_time',
             and 'pixel_extremes' (dict mapping (x,y) to {'t_first', 't_last', 'p_first', 'p_last'}).
    """
    n = len(events)
    if n == 0:
        return []
    
    k = n // b  # Events per bin
    bins = []
    
    # Global bin boundaries
    t_start = events[0][0]
    t_end = events[-1][0]
    
    current_idx = 0
    for i in range(b):
        # Determine start and end indices for this bin
        bin_start_idx = current_idx
        bin_end_idx = current_idx + k
        
        # Handle the last bin if n is not perfectly divisible by b
        if i == b - 1:
            bin_end_idx = n
            
        bin_events = events[bin_start_idx:bin_end_idx]
        
        if not bin_events:
            continue
            
        # Bin time boundaries
        t_i = bin_events[0][0]
        t_i_plus_1 = bin_events[-1][0]
        
        # Per-pixel extremes
        pixel_extremes = {}
        for event in bin_events:
            ts, x, y, p = event
            key = (x, y)
            
            if key not in pixel_extremes:
                pixel_extremes[key] = {
                    't_first': ts,
                    't_last': ts,
                    'p_first': p,
                    'p_last': p
                }
            else:
                # Update last if current timestamp is greater
                if ts > pixel_extremes[key]['t_last']:
                    pixel_extremes[key]['t_last'] = ts
                    pixel_extremes[key]['p_last'] = p
                
                # Update first if current timestamp is smaller (should be first event)
                # Since events are sorted, the first event encountered is the earliest
                # But we initialized with the first event, so we only need to check if we missed any
                # Actually, since we iterate in order, the first time we see a key, it is the first event.
                # We just need to update the last.
                pass 
        
        # Note: The pseudocode above assumes the first event seen for a pixel is the earliest.
        # This is true because the global event list is sorted by timestamp.
        
        bins.append({
            'index': i,
            'start_time': t_i,
            'end_time': t_i_plus_1,
            'pixel_extremes': pixel_extremes
        })
        
        current_idx = bin_end_idx
        
    return bins
```

**Parameters:**
- $b$: Number of bins. Typical values are not explicitly given in the sources, but it is a hyperparameter chosen to balance temporal resolution and noise robustness [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** Timestamps are typically stored with high precision (e.g., microseconds or nanoseconds). Ensure floating-point or integer types used for timestamps do not overflow or lose precision when calculating differences [S1, unofficial].
- **Boundaries:** $t_0$ and $t_b$ are fixed to the start and end of the entire stream. Intermediate $t_i$ values are determined by event counts, not time intervals [S1, unofficial].
- **Degenerate Input:** If the event stream is empty, no bins are created. If a pixel has no events in a specific bin, its calibration function inputs are undefined, requiring handling in the photometric calibration step [S1, unofficial].

## Subtopics

### Event Calibration Function
Computes the photometric quantity calibration offset for an event bin $B(t_i, t_{i+1})$ using the first and last triggered events at each pixel [S1, unofficial]. It sets itself apart by using a conditional function $\phi$ that checks for adjacent events and polarity consistency to determine the offset [S1, unofficial]. Choose this when integrating event data into the NeRF loss to account for temporal blur and exposure [S1, unofficial].

**Formula:**
$$h(B(t_i, t_{i+1})) = \phi\left(\frac{t_{i+1}-t_{\text{last}}^i}{t_{\text{first}}^{i+1}-t_{\text{last}}^i}, p_{\text{first}}^{i+1}\right) - \phi\left(\frac{t_i-t_{\text{last}}^{i-1}}{t_{\text{first}}^i-t_{\text{last}}^{i-1}}, p_{\text{first}}^i\right)$$

$$\phi(z, p_{\text{first}}^i) = \begin{cases} z p_{\text{first}}^i & \text{if } p_{\text{first}}^i = p_{\text{last}}^{i-1} \\ 0 & \text{if } p_{\text{first}}^i \neq p_{\text{last}}^{i-1} \\ 0.5 & \text{if } \nexists p_{\text{first}}^i \lor \nexists p_{\text{last}}^{i-1} \end{cases}$$

**Parameters:**
- $t_{\text{first}}^i, t_{\text{last}}^i$: Start and end times of the first/last event in bin $i$ at a specific pixel [S1, unofficial].
- $p_{\text{first}}^i, p_{\text{last}}^i$: Polarities of those events [S1, unofficial].
- $z$: The normalized time ratio passed to $\phi$ [S1, unofficial].

**Pitfalls:**
- The $(x,y)$ coordinates are omitted from the equations because the operation is identical for all pixels, but the values $t_{\text{first}}, t_{\text{last}}, p_{\text{first}}, p_{\text{last}}$ are pixel-specific [S1, unofficial].
- If adjacent events do not exist (e.g., at the start or end of the stream, or if a pixel has no events in bin $i-1$ or $i+1$), the function returns 0.5 [S1, unofficial].
- Polarity mismatch between adjacent bins results in a zero contribution for that term [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ to sort and split the event stream, where $N$ is the total number of events. Per-pixel extraction is also $O(N)$ in total across all bins.
- **Memory Complexity:** $O(N)$ to store the event stream and bin metadata.
- **Measured Performance:** The method achieves better performance in HDR NeRF training compared to uniform binning by leveraging the temporal blur prior [S1, unofficial].
- **Strengths:** Adapts to varying event densities, ensuring consistent statistical contribution from each bin.
- **Limitations:** Requires sorted events and careful handling of edge cases where adjacent events or polarities are missing.
- **When to Use:** When training HDR NeRF from single-exposure images and events, particularly when event density varies significantly over time [S1, unofficial].
- **When Not to Use:** When real-time processing is critical and sorting events is too expensive, or when event data is sparse and uniform time bins are sufficient.

## Background
Originated in the context of HDR NeRF training from single-exposure images and events, specifically in the "Seeing through Light and Darkness" paper [S1, unofficial]. It builds on the E3 NeRF method [S1, unofficial], which first leveraged the temporal blur prior in event distribution for event loss optimization. The method is used in sensor-physics grounded deblurring HDR NeRF frameworks to improve novel view synthesis performance [S1, unofficial].

## Sources
[S1] Seeing through Light and Darkness: Sensor-Physics Grounded Deblurring HDR NeRF from Single-Exposure Images and Events, https://arxiv.org/abs/2601.15475v4, fetched 2026-10-26 (unofficial)
