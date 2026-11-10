# PPLL Resolve Pass
Also called: Parallel Prefix List List Resolve
## Overview
The PPLL Resolve Pass is a fragment shader pass that resolves a linked list of fragments per pixel into a final color by sorting and front-to-back blending [S1, unofficial]. It takes the raw fragment data collected by the [Atomic Fragment List Insertion](2465-atomic-fragment-list-insertion.md) method and produces the final image, distinguishing itself from sibling methods like [A-buffer linked-list creation and ray processing](3878-a-buffer-linked-list-creation-and-ray-processing.md) by using per-pixel atomic exchange on a start offset texture for list construction rather than a global counter [S1, unofficial].
## How it works
### Inputs and Outputs
**Inputs:**
- `uStartOffsetTex`: A texture storing the head pointer (fragment index) for each pixel, initialized to `0xFFFFFFFFu` (end of list) by the Clear Pass [S1, unofficial].
- `uFragmentBufferTex`: A texture storing the fragment nodes, with `RGBA32UI` layout and dimensions `FRAG_BUFFER_W` × `FRAG_BUFFER_H` [S1, unofficial].
- `uViewportW`, `uViewportH`: Viewport dimensions [S1, unofficial].
- `FRAG_BUFFER_W`: Width of the fragment buffer texture (constant, typically 2048) [S1, unofficial].
- `MAX_NUM_FRAGS`: Maximum number of fragments to process per pixel (constant, typically 32) [S1, unofficial].

**Outputs:**
- `gl_FragColor`: The final color for the current pixel [S1, unofficial].

### Data Structures
- **Per-pixel local storage:**
  - `colorList[MAX_NUM_FRAGS]`: Array of `uint` storing packed fragment colors.
  - `depthList[MAX_NUM_FRAGS]`: Array of `float` storing fragment depths.
  - Total size: $32 \times 12$ bytes = 384 bytes [S1, unofficial].

### Steps
1. **Boundary Check:** Get pixel coordinates `x`, `y`. If `x >= uViewportW` or `y >= uViewportH`, discard [S1, unofficial].
2. **Load Head:** Load the starting offset (head pointer) from `uStartOffsetTex` at `(x, y)` into `fragOffset` [S1, unofficial].
3. **Initialize:** Set `numFrags = 0` [S1, unofficial].
4. **Traverse List:** Loop `i` from 0 to `MAX_NUM_FRAGS - 1`: If `fragOffset == 0xFFFFFFFFu`, break (end of list) [S1, unofficial]. Calculate address `ivec2(int(fragOffset) % FRAG_BUFFER_W, int(fragOffset) / FRAG_BUFFER_W)` [S1, unofficial]. Load `frag` from `uFragmentBufferTex` at the calculated address [S1, unofficial]. Update `fragOffset = frag.b` (the next pointer) [S1, unofficial]. Store `frag.r` in `colorList[i]` [S1, unofficial]. Store `uintBitsToFloat(frag.g)` in `depthList[i]` [S1, unofficial]. Increment `numFrags` [S1, unofficial].
5. **Empty Check:** If `numFrags == 0`, discard [S1, unofficial].
6. **Sort:** Insertion sort `colorList` and `depthList` by `depthList` ascending [S1, unofficial]. Loop `i` from 1 to `numFrags - 1` [S1, unofficial]. Store `fragColor = colorList[i]`, `fragDepth = depthList[i]` [S1, unofficial]. Loop `j` from `i` down to 1: If `depthList[j-1] > fragDepth`, shift `colorList[j-1]` to `colorList[j]` and `depthList[j-1]` to `depthList[j]` [S1, unofficial]. Decrement `j` [S1, unofficial]. Place `fragColor` and `fragDepth` at `j` [S1, unofficial].
7. **Blend:** Initialize `outColor = vec4(0.0)` [S1, unofficial]. Loop `i` from 0 to `MAX_NUM_FRAGS - 1`: If `i >= numFrags`, break [S1, unofficial]. Unpack `src = unpackColor(colorList[i])` [S1, unofficial]. Update `outColor.rgb = outColor.rgb + (1.0 - outColor.a) * src.a * src.rgb` [S1, unofficial]. Update `outColor.a = outColor.a + (1.0 - outColor.a) * src.a` [S1, unofficial]. If `outColor.a > 0.995`, break (early-out optimization) [S1, unofficial].
8. **Convert:** If `outColor.a > 1e-4`, divide `outColor.rgb` by `outColor.a` (premultiplied to straight alpha conversion) [S1, unofficial].
9. **Output:** Output `outColor` to `gl_FragColor` [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import struct

def uint_bits_to_float(val):
    return struct.unpack('f', struct.pack('I', val & 0xFFFFFFFF))[0]

def unpack_color(packed):
    # Assumes 10:10:10:2 packing as per source comment
    r = ((packed >> 20) & 0x3FF) / 1023.0
    g = ((packed >> 10) & 0x3FF) / 1023.0
    b = (packed & 0x3FF) / 1023.0
    a = ((packed >> 30) & 0x3) / 3.0
    return [r, g, b, a]

def resolve_pass(x, y):
    if x >= uViewportW or y >= uViewportH:
        return None

    frag_offset = texel_fetch(uStartOffsetTex, (x, y))[0]
    num_frags = 0
    color_list = [0] * MAX_NUM_FRAGS
    depth_list = [0.0] * MAX_NUM_FRAGS

    i = 0
    while i < MAX_NUM_FRAGS and frag_offset != 0xFFFFFFFF:
        buf_x = frag_offset % FRAG_BUFFER_W
        buf_y = frag_offset // FRAG_BUFFER_W

        frag = texel_fetch(uFragmentBufferTex, (buf_x, buf_y))

        frag_offset = frag[1]
        color_list[i] = frag[0]
        depth_list[i] = uint_bits_to_float(frag[2])

        i += 1
        num_frags += 1

    if num_frags == 0:
        return None

    for i in range(1, num_frags):
        frag_color = color_list[i]
        frag_depth = depth_list[i]

        j = i
        while j > 0 and depth_list[j-1] > frag_depth:
            color_list[j] = color_list[j-1]
            depth_list[j] = depth_list[j-1]
            j -= 1

        color_list[j] = frag_color
        depth_list[j] = frag_depth

    out_color = [0.0, 0.0, 0.0, 0.0]

    for i in range(num_frags):
        src = unpack_color(color_list[i])

        out_color[0] += (1.0 - out_color[3]) * src[3] * src[0]
        out_color[1] += (1.0 - out_color[3]) * src[3] * src[1]
        out_color[2] += (1.0 - out_color[3]) * src[3] * src[2]
        out_color[3] += (1.0 - out_color[3]) * src[3]

        if out_color[3] > 0.995:
            break

    if out_color[3] > 1e-4:
        out_color[0] /= out_color[3]
        out_color[1] /= out_color[3]
        out_color[2] /= out_color[3]

    return out_color
```

### Parameters
- `MAX_NUM_FRAGS`: Maximum number of fragments to process per pixel. Typical value: 32 [S1, unofficial].
- `FRAG_BUFFER_W`: Width of the fragment buffer texture. Typical value: 2048 [S1, unofficial].

### Edge Cases and Pitfalls
- **Early-out optimization:** The blend loop breaks if `outColor.a > 0.995` to save computation when the pixel is effectively opaque [S1, unofficial].
- **Empty list handling:** If no fragments are collected (`numFrags == 0`), the fragment is discarded [S1, unofficial].
- **Buffer Full:** If the fragment buffer is full, fragments are silently dropped during the gather phase, which may result in missing fragments in the resolve pass [S1, unofficial].
- **Precision:** Depth is stored as `uintBitsToFloat(frag.g)`, implying a specific bit-packing scheme for depth in the fragment node [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N^2)$ per pixel due to insertion sort, where $N = \text{MAX\_NUM\_FRAGS}$ [G]. Traversal is $O(N)$ [S1, unofficial].
- **Memory Complexity:** $O(N)$ per pixel for local storage, where $N = \text{MAX\_NUM\_FRAGS}$ [G].
- **Strengths:** Simple implementation, efficient for low fragment counts per pixel [S1, unofficial].
- **Limitations:** Insertion sort becomes inefficient for high fragment counts [S1, unofficial]. Buffer overflow can occur if the gather phase exceeds capacity [S1, unofficial].
- **When to use:** Suitable for scenes with moderate fragment density per pixel [S1, unofficial].
- **When not to use:** Not suitable for scenes with very high fragment density per pixel where sorting overhead becomes significant [S1, unofficial].

## Background
The PPLL Resolve Pass is part of the Parallel Prefix List List (PPLL) method for volume rendering [S1, unofficial]. It was described in the context of a WebGL2-based volume renderer [S1, unofficial]. The method relies on atomic operations for fragment list construction and a subsequent resolve pass for final color computation [S1, unofficial].

## Sources
[S1] Amory0709/geresim-spike, https://github.com/Amory0709/geresim-spike, fetched 2026-10-27 (unofficial)
