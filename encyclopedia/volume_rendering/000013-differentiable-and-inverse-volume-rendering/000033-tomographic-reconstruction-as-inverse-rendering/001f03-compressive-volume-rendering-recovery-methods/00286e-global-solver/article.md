# Global Solver
## Overview
The Global Solver is a compressive volume rendering reconstruction method that treats the entire image as a single unit for inverse rendering recovery [S1, unofficial]. It solves for missing pixel values by running a solver over the full image domain simultaneously, contrasting with block-based partitioning strategies [S1, unofficial]. It is designed to handle higher percentages of missing pixels while maintaining global consistency, though it prioritizes reconstruction quality over raw speed compared to local variants [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A volume rendering image with missing pixels (e.g., 60% missing, 40% present) [S1, unofficial].
- Output: A reconstructed full-resolution image [S1, unofficial].

**Data Structures:**
- The method operates on the entire image matrix as a single entity. No partitioning data structures are used [S1, unofficial].

**Steps:**
1. Initialize the solver for the entire image domain [S1, unofficial].
2. Define the number of iterations. This parameter can be fixed to a constant value or set via a flag to run until a desired solution convergence is reached [S1, unofficial].
3. Perform the convolution process. This is identified as the most time-consuming operation in the solver pipeline [S1, unofficial].
4. Update the image data based on the solver's iterative steps until the iteration count is met or convergence criteria are satisfied [S1, unofficial].

**Parameters:**
- **Number of iterations:** Affects solver performance. Can be fixed or dynamic based on a convergence flag [S1, unofficial].

**Pitfalls:**
- **FFT-based convolution overhead:** Using Fast Fourier Transform (FFT) for convolution causes significant performance degradation because memory is copied back and forth between CPU and GPU for each iteration [S1, unofficial]. This results in very slow reconstruction times [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def global_solver(image, missing_mask, max_iterations=None, convergence_flag=False):
    """
    Reconstructs the entire image globally using a gradient-based solver.
    
    Args:
        image: The input image with missing pixels (NaN or 0).
        missing_mask: Boolean mask indicating missing pixels.
        max_iterations: Fixed number of iterations. If None, uses convergence_flag.
        convergence_flag: If True, run until desired solution is arrived at.
        
    Returns:
        Reconstructed image.
    """
    # Initialize solver state for the entire image
    current_image = image.copy()
    
    # Determine iteration strategy
    if convergence_flag:
        # Run multiple times until desired solution is arrived at [S1, unofficial]
        while not converged(current_image, missing_mask):
            current_image = perform_convolution_step(current_image, missing_mask)
    else:
        # Fixed number of iterations
        for i in range(max_iterations):
            current_image = perform_convolution_step(current_image, missing_mask)
            
    return current_image

def perform_convolution_step(image, missing_mask):
    """
    Performs the most time-consuming operation: convolution.
    Note: FFT-based convolution causes CPU/GPU memory copies [S1, unofficial].
    """
    # Implementation detail: Convolution process [S1, unofficial]
    # In a naive FFT implementation, this would involve:
    # 1. Copy image to CPU
    # 2. FFT
    # 3. Multiply in frequency domain
    # 4. IFFT
    # 5. Copy back to GPU
    # This step is optimized in variants like the Local Solver [S1, unofficial]
    pass

def converged(image, missing_mask):
    # Placeholder for convergence check logic [G]
    return False
```

## Subtopics

### [Local Solver](4294-local-solver.md)
A parallel implementation that partitions the image into mutually exclusive blocks to reduce global memory access and improve performance using shared memory. It differs from the Global Solver by processing blocks independently, allowing for different numbers of iterations per block based on local convergence criteria.

## Operating characteristics
- **Time Complexity:** High. The convolution process is the most time-consuming operation [S1, unofficial].
- **Memory Access:** Inefficient when using FFT-based convolution due to frequent CPU/GPU memory transfers [S1, unofficial].
- **Measured Performance:** 
  - Reconstruction time of ~124 milliseconds (approx. 10 fps) when 60% of pixels are missing (40% present) [S1, unofficial].
  - Global solver takes 119 milliseconds using only 60% pixels [S1, unofficial].
- **Scaling:** Not well suited for interactivity due to the high reconstruction time [S1, unofficial].
- **Strengths:** Provides a global solution for the entire image, potentially offering better consistency than block-based methods [S1, unofficial].
- **Limitations:** Very slow reconstruction time compared to local solvers (e.g., Local Solver takes 5.6 milliseconds for the same task, making it 24 times faster) [S1, unofficial].
- **When to use:** When global consistency is prioritized over speed and interactivity is not required [S1, unofficial].
- **When not to use:** When interactive frame rates are needed [S1, unofficial].

## Background
The Global Solver was proposed in a thesis on GPU-based compressive volume rendering as one of two parallel approaches to reconstruction, the other being the Local Solver [S1, unofficial]. It builds on gradient solver techniques previously shown by Liu and Alim et al., who demonstrated reconstruction times using a CPU [S1, unofficial]. The method was developed to address the limitations of CPU-based reconstruction and to provide a baseline for comparing block-based optimizations [S1, unofficial].

## Sources
[S1] GPU-based Compressive Volume Rendering, https://doi.org/10.11575/prism/5366, fetched 2026-10-26 (unofficial)
