# Dynamic Cache Depth Scheduling

Also called: Anchor-based Cache Depth Adjustment, Cache Depth Heuristic

## Overview
Dynamic Cache Depth Scheduling is a heuristic strategy that adjusts the depth of reuse for cached computation results in subsequent frames based on the intensity of decoding calculations in the current frame [S1, unofficial]. It solves the problem of maintaining rendering quality and suppressing rapid appearance changes caused by sudden cache flushing in large-scale Gaussian Splatting models [S1, unofficial]. Unlike static cache reuse, this method dynamically balances performance and visual stability by modulating how many previous frames' results are retained [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera input (pose), current frame anchors ($X_{k'}$), total anchors ($X$), previous frame cache state [S1, unofficial].
*   **Outputs:** Rendered image, updated cache depth, updated computation cache [S1, unofficial].

**Data Structures:**
*   **Computation Cache:** Stores derived 3D Gaussians from previous frames [S1, unofficial]. The cache holds results from up to $D_{max}$ previous frames [S1, unofficial].
*   **Anchor Set:** The set of anchors to be decoded into 3D Gaussians for the current frame [S1, unofficial].
*   **Cache Depth ($D$):** An integer indicating how many previous frames' Gaussians are reused [S1, unofficial].

**Parameters:**
*   **Max Cache Reuse Depth ($D_{max}$):** Set to 10 to balance performance and quality [S1, unofficial].

**Algorithm Steps:**
1.  Initialize the rendering pipeline, set the initial cache depth and the guiding function $H$ [S1, unofficial].
2.  While receiving camera input:
    a.  Perform anchors indexing and filtering [S1, unofficial].
    b.  Check if the current cache reuse depth has reached $D_{max}$ [S1, unofficial]. If yes, invalidate those computation cache lines [S1, unofficial].
    c.  Calculate the **Anchor Duplicate Rate** (the ratio of anchors that can be reused or are identical to previous frames) [S1, unofficial].
    d.  Based on the Anchor Duplicate Rate:
        i.  If Anchor Duplicate Rate == 0% or it is the first frame: Decode all anchors into 3D Gaussians [S1, unofficial].
        ii. If Anchor Duplicate Rate == 100%: Reuse all 3D Gaussians from the cache [S1, unofficial].
        iii. Otherwise: Decode new anchors and update the computation cache [S1, unofficial].
    e.  Compose the new Gaussians with the cached Gaussians [S1, unofficial].
    f.  Configure the new cache depth $D$ using the guiding function and the duplicate rate [S1, unofficial].
    g.  Update the render buffer [S1, unofficial].
    h.  Rasterize the render buffer into an image [S1, unofficial].

**Formula:**
The expected reuse depth $D$ is determined by the guiding function $H$ applied to the ratio of current anchors to total anchors:
$$ D = H\left(\frac{|X_{k'}|}{|X|}\right) $$
where:
*   $H$ is the guiding function of the heuristic strategy [S1, unofficial].
*   $|X_{k'}|$ is the number of anchors in the current frame [S1, unofficial].
*   $|X|$ is the total number of anchors [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dynamic_cache_depth_scheduling(camera_input, total_anchors_X, max_reuse_depth=10):
    # Initialize state
    cache_depth = 0
    computation_cache = {}  # Maps frame_id to Gaussian parameters
    # [G] Implementation of H not specified in sources; standard linear scaling shown
    guiding_function_H = lambda ratio: max(1, int(ratio * max_reuse_depth))
    first_frame = True

    while camera_input is not None:
        # 1. Anchors indexing and filtering
        current_anchors_X_k_prime = index_and_filter_anchors(camera_input)

        # 2. Check max cache reuse depth
        if cache_depth >= max_reuse_depth:
            computation_cache.clear()
            cache_depth = 0

        # 3. Calculate Anchor Duplicate Rate
        anchor_duplicate_rate = calculate_duplicate_rate(current_anchors_X_k_prime, computation_cache)

        # 4. Decode or Reuse based on duplicate rate
        if anchor_duplicate_rate == 0.0 or first_frame:
            new_gaussians = decode_anchors_to_gaussians(current_anchors_X_k_prime)
            cache_depth = 0
            first_frame = False
        elif anchor_duplicate_rate == 1.0:
            new_gaussians = reuse_cached_gaussians(computation_cache)
        else:
            new_anchors = get_new_anchors(current_anchors_X_k_prime)
            decoded_new_gaussians = decode_anchors_to_gaussians(new_anchors)
            update_computation_cache(computation_cache, decoded_new_gaussians)
            new_gaussians = compose_gaussians(decoded_new_gaussians, computation_cache)

        # 5. Configure cache depth based on guiding function and duplicate rate
        ratio = len(current_anchors_X_k_prime) / len(total_anchors_X)
        cache_depth = guiding_function_H(ratio)

        # 6. Update render buffer and rasterize
        render_buffer = update_render_buffer(new_gaussians, camera_input)
        image = rasterize(render_buffer)
        return image

# Helper definitions
def index_and_filter_anchors(camera_input):
    # Standard view frustum culling and spatial indexing [G]
    return camera_input.get_visible_anchors()

def calculate_duplicate_rate(current_anchors, cache):
    # Ratio of current anchors matching cached frame's anchors [G]
    if not cache: return 0.0
    cached_anchors = next(iter(cache.values())).anchors
    matches = sum(1 for a in current_anchors if a in cached_anchors)
    return matches / len(current_anchors) if current_anchors else 0.0

def decode_anchors_to_gaussians(anchors):
    # Standard Gaussian Splatting derivation [G]
    return [anchor.derive_gaussian() for anchor in anchors]

def reuse_cached_gaussians(cache):
    return next(iter(cache.values())).gaussians

def get_new_anchors(current_anchors):
    return [a for a in current_anchors if not a.is_cached()]

def update_computation_cache(cache, new_gaussians):
    cache[cache.get('frame_id', 0) + 1] = type('obj', (object,), {'anchors': new_gaussians, 'gaussians': new_gaussians})()

def compose_gaussians(new_gaussians, cache):
    return new_gaussians + next(iter(cache.values())).gaussians

def update_render_buffer(gaussians, camera_input):
    return {'gaussians': gaussians, 'camera': camera_input}

def rasterize(buffer):
    return buffer['camera'].project(buffer['gaussians'])
```

**Edge Cases and Pitfalls:**
*   **Stale Appearance:** Maintaining the same Gaussian appearance parameters in the cache for too long causes significant rendering errors, particularly on trajectories around a centered target [S1, unofficial]. This is handled by flushing the cache in time via the max reuse depth limit [S1, unofficial].
*   **Rapid Appearance Changes:** Static cache reuse depth can cause rapid changes in scene appearance when the cache is flushed [S1, unofficial]. Dynamic scheduling suppresses this by gradually adjusting the depth [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the anchor decoding stage and Gaussian rasterization stage [S1, unofficial]. By reusing cached Gaussians, the decoding stage is skipped for duplicate anchors, reducing computational load [S1, unofficial].
*   **Memory Complexity:** Proportional to the cache depth $D_{max}$ and the number of anchors [S1, unofficial]. With $D_{max}=10$, the cache stores results from up to 10 previous frames [S1, unofficial].
*   **Performance:** Tested on Scaffold-GS and OctreeGS models [S1, unofficial]. Setting $D_{max}=10$ balances performance and quality [S1, unofficial]. It is effective for both single GPU and multi-GPU resources [S1, unofficial].
*   **Strengths:** Maintains image rendering quality without significant decrease or fluctuation [S1, unofficial]. Suppresses rapid appearance changes caused by sudden cache flushing [S1, unofficial]. The method is model-agnostic and independent of the specific pipeline structure [S1, unofficial].
*   **Limitations:** Requires careful tuning of the guiding function $H$ and duplicate rate calculation [S1, unofficial]. May introduce latency if cache management is inefficient [G].

## Background
Dynamic Cache Depth Scheduling was introduced in the paper "A GS-Cache Inference Framework for Large-Scale Gaussian Splatting Models" [S1, unofficial]. It was developed to address redundancy in binocular stereo rendering and to optimize the derivation stage in large-scale Gaussian Splatting models like Scaffold-GS and OctreeGS [S1, unofficial]. The method is designed to be independent of the specific model and pipeline structure [S1, unofficial].

## Sources
[S1] A GS-Cache Inference Framework for Large-Scale Gaussian Splatting Models, https://arxiv.org/abs/2502.14938, fetched 2026-10-26 (unofficial)
