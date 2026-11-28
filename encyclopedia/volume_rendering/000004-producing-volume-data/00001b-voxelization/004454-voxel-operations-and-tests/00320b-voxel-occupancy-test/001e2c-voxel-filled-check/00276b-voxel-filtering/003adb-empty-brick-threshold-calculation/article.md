# Empty-Brick Threshold Calculation

Also called: emptyBrickThresholdFor

## Overview
Computes the voxel intensity threshold below which a volume brick is considered empty and can be skipped during streaming [S1, unofficial].

## How it works
**Inputs:**
- `metadata.bitDepth`: Integer, 8 or 16, determining intensity range scaling [S1, unofficial].
- `channelWindows`: Array of window/level pairs per channel, or undefined [S1, unofficial].
- `isFloat`: Boolean indicating if data is floating-point [S1, unofficial].

**Outputs:**
- Threshold value [S1, unofficial].

**Data Structures:**
- `channelWindows`: Array containing window (width) and level (center) values per channel [S1, unofficial].

**Steps:**
1. Call `emptyBrickThresholdFor` with `metadata.bitDepth`, `channelWindows`, and `isFloat` [S1, unofficial].

**Formulas:**
Not given in the sources.

**Parameters:**
- `bitDepth`: 8 or 16 [S1, unofficial].
- `channelWindows`: Array of window/level pairs or undefined [S1, unofficial].
- `isFloat`: Boolean flag [S1, unofficial].

**Edge Cases and Pitfalls:**
- If `channelWindows` is undefined, the threshold defaults to 0.0 [S1, unofficial].
- The source indicates `isFloat` is a flag but does not detail the specific threshold calculation for floating-point data [S1, unofficial].
- Precision handling for non-integer window/level results should use consistent rounding or flooring [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def emptyBrickThresholdFor(bitDepth: int, channelWindows: list | None, isFloat: bool) -> float:
    # Determine max intensity value based on bit depth
    if bitDepth == 8:
        max_val = 255
    elif bitDepth == 16:
        max_val = 65535
    else:
        raise ValueError("Unsupported bit depth")

    # Calculate threshold based on channel windows or default
    # Note: The exact formula for channelWindows and isFloat handling is not specified in the sources
    if channelWindows is not None and len(channelWindows) > 0:
        win = channelWindows[0]["window"]
        lev = channelWindows[0]["level"]
        # Exact calculation formula not provided in sources [G]
        threshold = 0.0
    else:
        threshold = 0.0

    return threshold
```

## Subtopics

### Background Thresholding via K-Means
Fits two intensity populations on a bounded, seeded sample of nonzero data to determine a background threshold [S2, unofficial]. It extracts nonzero values from the volume (`values = volume[volume > 0]`); if empty, it returns 0.0 [S2, unofficial]. It creates a random number generator with a seed [S2, unofficial]. It samples up to 10,000 values without replacement [S2, unofficial]. If the unique sample size is less than 2, it returns 0.0 to preserve constant-valued foreground [S2, unofficial]. It fits a KMeans model with `n_clusters=2`, `n_init=3`, and `random_state=seed` on the sample reshaped to a column vector [S2, unofficial]. It returns the minimum cluster center rounded to 6 decimal places [S2, unofficial].

## Operating characteristics
**Time Complexity:** Not given in the sources. [G]
**Memory Complexity:** Not given in the sources. [G]
**Strengths:** Enables efficient streaming by skipping empty bricks [S1, unofficial].
**Limitations:** Relies on accurate metadata; if the threshold is set incorrectly, relevant data may be skipped [G].
**When to use:** When streaming large volume datasets and bricks can be safely skipped if their intensity is below a known background level [S1, unofficial].
**When not to use:** When the background intensity is not uniform or when the threshold cannot be determined from metadata alone [G].

## Background
Not given in the sources. [G]

## Sources
[S1] Show HN: Kiln – WebGPU-native out-of-core volume rendering for multi-GB datasets, https://github.com/MPanknin/kiln-render, fetched 2026-10-26 (unofficial)
[S2] Winson-L/VolumeSegmenter, https://github.com/Winson-L/VolumeSegmenter, fetched 2026-10-27 (unofficial)
