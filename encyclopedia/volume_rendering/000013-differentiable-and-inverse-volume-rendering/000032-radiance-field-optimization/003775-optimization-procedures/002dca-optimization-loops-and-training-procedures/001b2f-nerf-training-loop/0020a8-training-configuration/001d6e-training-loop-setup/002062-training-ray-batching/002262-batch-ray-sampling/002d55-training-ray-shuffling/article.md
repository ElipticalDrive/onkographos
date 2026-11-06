# Training Ray Shuffling

Also called: Ray Sampling and Shuffling, ray initialization, shuffleRays

## Overview
Training Ray Shuffling initializes the training data pipeline by generating a random permutation of ray indices and transferring them to the GPU for efficient batched sampling during differentiable volume rendering optimization. It addresses the need to randomize the order in which rays are processed to prevent training bias from spatial clustering of rays [S1, S2, unofficial]. It differs from its parent [Batch Ray Sampling](2767-batch-ray-sampling.md) by explicitly handling the permutation logic and GPU transfer, and from its sibling [Pixel Batching](3134-pixel-batching.md) by operating on the ray index level rather than pixel-level aggregation.

## How it works
**Inputs:**
- Training camera parameters (intrinsics, extrinsics) [S2, unofficial].
- Ground truth images [S2, unofficial].
- Scene data and dataset size [S2, unofficial].
- Configuration flags: `MaskAreaOnly` (boolean), `whiteBG` (boolean) [S2, unofficial].
- Total number of samples $N$ (derived from dataset size and image resolution) [S1, unofficial].
- Batch size $BS$ [S1, unofficial].

**Outputs:**
- A shuffled array of ray indices on the GPU [S2, unofficial].
- A state object containing the current permutation and pointer [S1, unofficial].

**Data Structures:**
- `idx`: A 1D array of integers of length $N$, storing a permutation of $[0, N)$ [S1, unofficial].
- `top`: An integer pointer or variable tracking the current position in `idx` [S1, unofficial].
- `ray`: A GPU array holding the actual ray data (origin, direction) corresponding to the shuffled indices [S2, unofficial].
- `sampleRaysIndexGPU`: A GPU array holding the shuffled indices [S2, unofficial].

**Steps:**
1. **Initialization:** Create a temporary object and call `initialSelf` with training cameras, ground truth, scene data, dataset size, `MaskAreaOnly`, `whiteBG`, and the initial index array `idx` [S2, unofficial].
2. **Shuffling:** Call `shuffleRays` with `temp.totalRays`, a boolean flag (typically `false` for initial shuffle), and `configs->shuffleRays` [S2, unofficial].
3. **Transfer:** Call `temp.shuffleTo` to move the shuffled ray data to `this->ray` and the shuffled indices to `sampleRaysIndexGPU` [S2, unofficial].
4. **Cleanup:** Call `temp.freeSelf()` to release temporary resources [S2, unofficial].
5. **Regeneration:** Call `regenerateSamples()` to prepare for the next epoch or large batch [S2, unofficial].
6. **Iteration:** During training, yield the slice `idx[top:top+BS]` [S1, unofficial].
7. **Reset:** If `top + BS > N`, regenerate `idx` as a new random permutation of $[0, N)$ and reset `top = 0` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

class TrainingRayShuffler:
    def __init__(self, N, BS, config):
        self.N = N
        self.BS = BS
        self.config = config
        self.idx = None
        self.top = 0
        self.ray_data_gpu = None
        self.ray_indices_gpu = None
        self._initialize()

    def _initialize(self):
        # Step 1: Initialize temporary object with scene data
        # temp = create_temp_object()
        # temp.initialSelf(train_cameras, ground_truth, scene, self.N, 
        #                  self.config.MaskAreaOnly, False, self.config.whiteBG, self.idx)
        # Note: Implementation details of temp are abstracted here [S2, unofficial]
        pass

    def _shuffle_and_transfer(self):
        # Step 2: Shuffle rays
        # temp.shuffleRays(temp.totalRays, False, self.config.shuffleRays)
        
        # Step 3: Transfer to GPU
        # temp.shuffleTo(&self.ray_data_gpu, self.ray_indices_gpu)
        
        # Step 4: Free temp
        # temp.freeSelf()
        
        # Step 5: Regenerate samples
        # regenerateSamples()
        pass

    def generate_permutation(self):
        # Use numpy for permutation to avoid torch.randperm issues on some machines [S1, unofficial]
        self.idx = np.random.permutation(self.N)
        self.top = 0
        # Trigger GPU transfer if indices change significantly or at epoch start
        self._shuffle_and_transfer()

    def get_next_batch(self):
        if self.idx is None or self.top >= self.N:
            self.generate_permutation()
        
        batch_indices = self.idx[self.top:self.top + self.BS]
        self.top += self.BS
        
        # Return batch indices and corresponding ray data from GPU
        # In practice, ray_data_gpu is indexed by batch_indices
        return batch_indices, self.ray_indices_gpu[batch_indices]

    def reset(self):
        self.idx = None
        self.top = 0
```

**Parameters:**
- `MaskAreaOnly`: Boolean flag passed to initialization, likely restricting sampling to masked areas [S2, unofficial].
- `whiteBG`: Boolean flag passed to initialization, handling background color assumptions [S2, unofficial].
- `shuffleRays`: Configuration flag enabling/disabling the shuffle operation [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Precision/Platform Specificity:** `torch.randperm` may produce incorrect results on certain hardware configurations; `np.random.permutation` is recommended as a fallback [S1, unofficial].
- **Boundary Conditions:** Ensure `top` is reset correctly when `top + BS > N` to avoid out-of-bounds access [S1, unofficial].
- **Memory Management:** Properly freeing temporary objects (`temp.freeSelf()`) is critical to prevent GPU memory leaks [S2, unofficial].

## Subtopics

### [Ray Shuffling & Reordering](5454-ray-shuffling-reordering.md)
Performs the actual data movement on the GPU via a custom kernel to improve cache locality or load balancing. Choose this when fine-grained control over GPU memory access patterns is needed for performance optimization.

## Operating characteristics
- **Time Complexity:** $O(N)$ for generating the permutation and $O(N)$ for transferring indices to GPU [S1, S2, unofficial].
- **Memory Complexity:** $O(N)$ for storing the index array and ray data [S1, S2, unofficial].
- **Performance:** Depends on GPU transfer speed and permutation generation efficiency. Using `np.random.permutation` may be slower than native CUDA implementations but ensures correctness [S1, unofficial].
- **Strengths:** Simple implementation, effective randomization for training stability.
- **Limitations:** Potential platform-specific bugs with PyTorch primitives; overhead of CPU-to-GPU transfer if not optimized [S1, S2, unofficial].
- **When to Use:** Standard differentiable volume rendering training where ray order randomization is required.
- **When Not to Use:** When deterministic ray ordering is needed for reproducibility or specific spatial sampling strategies.

## Background
Originating from implementations in DirectVoxGO and Clear-plenoxels, this method is part of the broader NeRF training loop setup. It ensures that the model is exposed to a diverse set of rays in each batch, preventing overfitting to specific spatial regions [S1, S2, unofficial].

## Sources
[S1] sunset1995/DirectVoxGO, https://github.com/sunset1995/DirectVoxGO, fetched 2026-10-26 (unofficial)
[S2] nortonBryan/Clear-plenoxels, https://github.com/nortonBryan/Clear-plenoxels, fetched 2026-10-27 (unofficial)
