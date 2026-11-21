# Random Camera Sampling

Also called: Camera Sampling

## Overview
Random Camera Sampling selects a fixed number of cameras from a predefined viewpoint stack by uniformly sampling indices without replacement [S1, unofficial]. It solves the problem of generating a diverse set of training views for 3D reconstruction or novel view synthesis when a limited pool of existing captures is available [S1, unofficial]. It differs from its sibling [Random Pose Selection](8187-random-pose-selection.md) by drawing exclusively from a discrete, existing list of poses rather than generating continuous pose parameters from a distribution [S1, unofficial].

## How it works
**Inputs:**
- `my_viewpoint_stack`: A list of camera objects or pose data structures representing available viewpoints.
- `num_cams`: An integer specifying the number of cameras to sample.

**Outputs:**
- `camlist`: A list of `num_cams` camera objects, shuffled relative to their original order in the stack.

**Data Structures:**
- `my_viewpoint_stack`: A dynamic array (list) of camera poses.
- `camlist`: A dynamic array (list) of selected camera poses.

**Parameters:**
- `num_cams`: The count of cameras to extract. The source implementation fixes this value at 10 [S1, unofficial].

**Steps:**
1. Initialize an empty list `camlist`.
2. Set `num_cams` to 10 [S1, unofficial].
3. Loop `num_cams` times:
    a. Generate a random integer `loc` uniformly distributed in the range $[0, \text{len}(my\_viewpoint\_stack) - 1]$ [S1, unofficial].
    b. Remove the camera at index `loc` from `my_viewpoint_stack` using a pop operation [S1, unofficial].
    c. Append the removed camera to `camlist` [S1, unofficial].
4. Return `camlist`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def random_camera_sampling(my_viewpoint_stack, num_cams=10):
    """
    Samples num_cams cameras from my_viewpoint_stack without replacement.
    
    Args:
        my_viewpoint_stack: List of camera pose objects.
        num_cams: Number of cameras to sample (default 10).
        
    Returns:
        List of sampled camera pose objects.
    """
    camlist = []
    # Note: The source implementation hardcodes num_cams = 10 [S1, unofficial]
    # but the function signature accepts it for flexibility.
    
    for _ in range(num_cams):
        if len(my_viewpoint_stack) == 0:
            raise ValueError("Not enough viewpoints in stack to sample requested number.")
        
        # Generate random index in [0, len(stack) - 1]
        loc = random.randint(0, len(my_viewpoint_stack) - 1)
        
        # Pop camera at loc and append to result list
        camera = my_viewpoint_stack.pop(loc)
        camlist.append(camera)
        
    return camlist
```

**Edge Cases and Pitfalls:**
- **Stack Exhaustion:** If `num_cams` exceeds the length of `my_viewpoint_stack`, the algorithm will fail or return fewer cameras than requested. The implementation must guard against this [G].
- **Replacement:** The use of `pop` ensures sampling without replacement, preventing duplicate cameras in the output list [S1, unofficial].

## Subtopics

### Structured Camera Batch Generation
Structured Camera Batch Generation creates a list of camera samples for a 3D dataset by distributing them between close-up and general views based on a configurable ratio, sampling directions uniformly or within a hemisphere [S2, unofficial]. It differs from basic Random Camera Sampling by introducing a structured bias towards specific viewing distances (close-up vs. general) and shuffling the final list to randomize order [S2, unofficial]. It is chosen when the application requires a balanced mix of detailed and contextual views [S2, unofficial].
- **Definition:** Generates camera samples with a mix of close-up and general views [S2, unofficial].
- **Steps:**
    1. Initialize a random number generator `rng` if not provided [S2, unofficial].
    2. Calculate `close_up_count` as `int(round(total_frames * close_up_ratio))` [S2, unofficial].
    3. Loop `index` from 0 to `total_frames - 1`:
        a. Determine `is_close_up` as `index < close_up_count` [S2, unofficial].
        b. Sample `radius` using `_sample_radius` with `min_radius`, `max_radius`, `is_close_up`, and `rng` [S2, unofficial].
        c. Sample `direction` using `_sample_direction` with `radius`, `view_distribution`, and `rng` [S2, unofficial].
        d. Create a `CameraSample` with the computed position and `is_close_up` flag [S2, unofficial].
        e. Append the sample to the `samples` list [S2, unofficial].
    4. Shuffle the `samples` list [S2, unofficial].
    5. Return the shuffled `samples` [S2, unofficial].
- **Parameters:**
    - `CLOSE_UP_BAND_FRACTION`: A constant fraction of the radius span used for the close-up maximum radius, defaulting to 0.15 [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of cameras sampled, as each sample involves a constant-time random index generation and a list pop operation [S1, unofficial].
- **Memory Complexity:** $O(N)$ to store the output list of sampled cameras [S1, unofficial].
- **Strengths:** Simple to implement; guarantees no duplicate views; efficient for small to medium-sized viewpoint stacks [S1, unofficial].
- **Limitations:** Does not guarantee spatial coverage of the scene; performance depends entirely on the diversity of the input `my_viewpoint_stack` [S1, unofficial].
- **When to use:** When a fixed, small set of diverse training views is needed from a pre-existing capture set, such as in FastGS training [S1, unofficial].
- **When not to use:** When continuous pose generation is required or when the input stack is too small to provide adequate coverage [S1, unofficial].

## Background
Random Camera Sampling is a technique used in Gaussian Splatting and 3D reconstruction pipelines to select training viewpoints. It is implemented in the FastGS repository, which provides optimized tools for training 3D Gaussian Splatting models [S1, unofficial]. The method is often used in conjunction with other sampling strategies like Structured Camera Batch Generation to ensure balanced dataset creation [S2, unofficial].

## Sources
[S1] fastgs/FastGS, https://github.com/fastgs/FastGS, fetched 2026-10-27 (unofficial)
[S2] yumehiko/3dgs-dataset-builder, https://github.com/yumehiko/3dgs-dataset-builder, fetched 2026-10-27 (unofficial)
