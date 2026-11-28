# BasicDAGColorErrors
## Overview
BasicDAGColorErrors computes a binary error flag by comparing compressed and uncompressed color representations for adaptive quality control [S1, unofficial]. It returns a strict binary indicator (1.0 or 0.0) based on a fixed threshold, avoiding the storage of continuous floating-point error values per leaf [S1, unofficial].
## Shared principles
This family builds on [BasicDAGColorsBase](6742-basicdagcolorsbase.md), which decouples leaf count storage from the main geometry array using an enclosed leaves array [S1, unofficial]. The core mechanism relies on the `enclosedLeaves` array to determine the number of leaves, which dictates the scope of the color error computation [S1, unofficial]. The error calculation is performed per leaf, comparing the stored compressed color against a reference uncompressed color [S1, unofficial].
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_color(leaveCount):
    # Retrieve compressed and uncompressed colors (implementation details not given in the sources)
    compressed = get_compressed_color()
    uncompressed = get_uncompressed_color()
    # Compute error via ColorUtils::color_error()
    error = color_error(compressed, uncompressed)
    return 1.0 if error > 0.04 else 0.0
```
## Subtopics
Not given in the sources.
## Background
Not given in the sources.
## Sources
[S1] Phyronnaz/HashDAG, https://github.com/Phyronnaz/HashDAG, fetched 2026-10-26 (unofficial)
