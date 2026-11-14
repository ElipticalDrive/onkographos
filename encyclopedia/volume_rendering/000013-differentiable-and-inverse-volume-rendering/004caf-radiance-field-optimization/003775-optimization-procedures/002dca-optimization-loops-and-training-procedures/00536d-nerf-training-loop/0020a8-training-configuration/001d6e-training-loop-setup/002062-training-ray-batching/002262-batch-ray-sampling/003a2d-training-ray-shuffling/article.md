# Training Ray Shuffling

Also called: Batch Index Generator, Ray Sampling and Shuffling, ray initialization, shuffleRays

## Overview
Training Ray Shuffling generates random batch indices for training and transfers shuffled indices to the GPU [S2, S3, unofficial]. It differs from its parent [Batch Ray Sampling](2767-batch-ray-sampling.md) by explicitly managing the permutation state and handling the CPU-to-GPU transfer [G].

## How it works
**Inputs:** Training cameras, ground truth images, scene data, dataset size $N$, batch size $BS$, and configuration flags `MaskAreaOnly` and `whiteBG` [S3, unofficial].
**Outputs:** Shuffled ray indices on the GPU [S3, unofficial].
**Data Structures:** `idx` (1D integer array storing a permutation of $[0, N)$), `top` (integer tracking the current position in `idx`), `ray` (GPU array for ray data), and `sampleRaysIndexGPU` (GPU array for shuffled indices) [S1, S3, unofficial].
**Steps:**
1. Initialize `idx` as a random permutation of $[0, N)$ and `top` to $0$ [S1, S2, unofficial].
2. Create a temporary object and call `initialSelf` with training cameras, ground truth, scene data, dataset size, `MaskAreaOnly`, `false`, `whiteBG`, and `idx` [S3, unofficial].
3. Call `shuffleRays` with `temp.totalRays`, `false`, and `configs->shuffleRays` [S3, unofficial].
4. Call `temp.shuffleTo` to transfer data to `this->ray` and `this->sampleRaysIndexGPU` [S3, unofficial].
5. Call `temp.freeSelf()` and `regenerateSamples()` [S3, unofficial].
6. In an infinite training loop, check if `top + BS > N` [S1, S2, unofficial].
7. If true, reset `idx` to a new random permutation and `top` to $0$ [S1, S2, unofficial].
8. Yield the slice `idx[top:top+BS]` and increment `top` by $BS$ [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

class TrainingRayShuffler:
    def __init__(self, N, BS, train_cameras, ground_truth, scene, config):
        self.N = N
        self.BS = BS
        self.train_cameras = train_cameras
        self.ground_truth = ground_truth
        self.scene = scene
        self.config = config
        self.idx = None
        self.top = 0
        self.ray = None
        self.sampleRaysIndexGPU = None
        self._initialize()

    def _initialize(self):
        # Step 1: Initialize idx as a random permutation of [0, N) and top to 0
        self.idx = np.random.permutation(self.N)
        self.top = 0

        # Steps 2-6: Setup temporary object, initialize ray data, shuffle, transfer to GPU, cleanup
        # temp.initialSelf(train_cameras, ground_truth, scene, N, MaskAreaOnly, False, whiteBG, idx)
        # temp.shuffleRays(temp.totalRays, False, config.shuffleRays)
        # temp.shuffleTo(&self.ray, self.sampleRaysIndexGPU)
        # temp.freeSelf()
        # regenerateSamples()
        # Note: The exact implementation of temp and regenerateSamples is not provided in the sources [S3, unofficial].

    def get_next_batch(self):
        # Step 7: Infinite training loop
        while True:
            # Check if current batch exceeds total samples
            if self.top + self.BS > self.N:
                # Reset idx to a new random permutation and top to 0
                self.idx = np.random.permutation(self.N)
                self.top = 0

            # Yield the slice idx[top:top+BS] and increment top by BS
            batch = self.idx[self.top:self.top + self.BS]
            self.top += self.BS
            yield batch
```

**Parameters:** `MaskAreaOnly` (boolean flag for masked area sampling), `whiteBG` (boolean flag for background color assumption), and `shuffleRays` (configuration flag enabling the shuffle operation) [S3, unofficial].
**Edge Cases and Pitfalls:** `torch.randperm` may produce incorrect results on certain CUDA hardware; `np.random.permutation` is recommended instead [S1, S2, unofficial]. Ensure `top` is reset correctly when `top + BS > N` to avoid out-of-bounds access [S1, S2, unofficial].

## Subtopics

### [Ray Shuffling & Reordering](5454-ray-shuffling-reordering.md)
Permutes ray data across GPU arrays using an index mapping array to improve training order or batch distribution. It sets itself apart by operating directly on GPU memory layouts rather than CPU-side index generation. Choose this variant when fine-grained control over GPU memory access patterns is required for performance optimization.

## Operating characteristics
- Time and memory complexity: Not given in the sources. [S1, S2, S3, unofficial]
- Measured performance and scaling: Not given in the sources. [S1, S2, S3, unofficial]
- Strengths: Simple implementation and effective randomization for training stability [S1, S2, unofficial].
- Limitations: Potential platform-specific bugs with PyTorch primitives and CPU-to-GPU transfer overhead if not optimized [S1, S2, unofficial].
- When to use it and when not to use: Not given in the sources. [S1, S2, S3, unofficial]

## Background
Originating from implementations in DirectVoxGO and Clear-plenoxels, this method is part of the broader NeRF training loop setup [S1, S2, S3, unofficial].

## Sources
[S1] sjtuytc/UnboundedNeRFPytorch, https://github.com/sjtuytc/UnboundedNeRFPytorch, fetched 2026-10-26 (unofficial)
[S2] sunset1995/DirectVoxGO, https://github.com/sunset1995/DirectVoxGO, fetched 2026-10-26 (unofficial)
[S3] nortonBryan/Clear-plenoxels, https://github.com/nortonBryan/Clear-plenoxels, fetched 2026-10-27 (unofficial)
