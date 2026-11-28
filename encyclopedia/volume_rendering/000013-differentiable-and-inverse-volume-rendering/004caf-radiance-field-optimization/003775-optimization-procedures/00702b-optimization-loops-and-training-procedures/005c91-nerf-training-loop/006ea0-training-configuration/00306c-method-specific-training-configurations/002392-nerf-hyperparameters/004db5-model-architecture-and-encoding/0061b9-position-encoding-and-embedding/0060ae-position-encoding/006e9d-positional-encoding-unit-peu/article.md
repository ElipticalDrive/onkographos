# Positional Encoding Unit (PEU)

Also called: Hardware Positional Encoding Unit

## Overview
The Positional Encoding Unit (PEU) is a specialized hardware accelerator designed to compute feature mapping functions for neural radiance fields (NeRF) by performing inner product operations with vector sizes $R=3$ and $R=6$ [S1]. It differs from software-based implementations by offloading the matrix multiplication and trigonometric evaluation to dedicated logic, supporting isotropic and anisotropic random Fourier features [S1]. It builds on [Position Encoding](18204-position-encoding.md) by providing a fixed-function pipeline for the specific frequency transformations required in NeRF training and rendering.

## How it works
**Inputs and Outputs:**
- **Input:** Position vectors $\mathbf{x}$ (dimension 3) or direction vectors $\mathbf{d}$ (dimension 3), and a frequency matrix $\mathbf{F}$ (dimensions $R \times N$, where $R \in \{3, 6\}$ and $N$ is the number of frequency bands, typically 128).
- **Output:** Transformed feature vectors containing $\sin(\mathbf{z})$ and $\cos(\mathbf{z})$ components, where $\mathbf{z} = \mathbf{x} \cdot \mathbf{F}$ or $\mathbf{z} = \mathbf{d} \cdot \mathbf{F}$.

**Data Structures:**
- **Frequency Matrix Memory:**
  - For $R=3$: A single memory block of size $3 \times 128$ stores the frequency matrix $\mathbf{F}$.
  - For $R=6$: Two separate memory blocks, each of size $3 \times 128$. The first block stores the first half of the $6 \times 128$ matrix, and the second block stores the second half.
- **MAC Unit:** A cascaded Multiply-Accumulate unit.
  - For $R=3$: A 3-stage cascaded MAC unit performs 3 multiplications and 2 additions in serial.
  - For $R=6$: The same hardware structure is used but operates through all 6 stages.

**Steps:**
1. **Load Frequency Matrix:** Store the frequency matrix $\mathbf{F}$ in local memory. This matrix is constant for all sample points in a particular rendering task [S1].
2. **Stream Coordinates:** Stream position $\mathbf{x}$ or direction $\mathbf{d}$ vectors into the matrix multiplication unit [S1].
3. **Matrix Multiplication:** Multiply the streamed coordinates with the frequency matrix $\mathbf{F}$ in a pipelined manner to generate the intermediate vector $\mathbf{z}$ [S1].
   - For $R=3$: Access one memory block. Process through the first 3 stages of the MAC unit. The later 3 stages are bypassed [S1].
   - For $R=6$: Access both memory blocks simultaneously. Process through all 6 stages of the cascaded MAC unit [S1].
4. **Trigonometric Transformation:** Feed the intermediate vector $\mathbf{z}$ to a Coordinate Rotation Digital Computer (CORDIC) array to generate the final transformed features $\sin(\mathbf{z})$ and $\cos(\mathbf{z})$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def peu_compute(coordinates, frequency_matrix, mode):
    """
    Simulates the PEU hardware logic.
    coordinates: list of 3 floats (x or d)
    frequency_matrix: list of lists, shape (R, N)
    mode: 'R3' or 'R6'
    """
    R = 3 if mode == 'R3' else 6
    N = len(frequency_matrix[0])
    
    # Step 1 & 2: Memory access and streaming
    # In hardware, F is pre-loaded. Coordinates are streamed.
    
    # Step 3: Matrix Multiplication via MAC unit
    # z = coordinates . F.T
    z = [0.0] * N
    for i in range(R):
        for j in range(N):
            z[j] += coordinates[i] * frequency_matrix[i][j]
            
    # Step 4: CORDIC array for sin/cos
    # Hardware uses CORDIC for efficiency
    sin_z = [math.sin(val) for val in z]
    cos_z = [math.cos(val) for val in z]
    
    return sin_z, cos_z
```

**Parameters:**
- **Frequency Matrix Dimension $R$:** Supports $R=3$ (for 3-dimensional data) and $R=6$ (for 6-dimensional data, likely combining position and direction or higher-order features) [S1].
- **Memory Size:** $3 \times 128$ per block [S1].

**Edge Cases and Pitfalls:**
- **Power Management:** In $R=3$ mode, the second memory block is set to sleep mode to reduce power consumption [S1].
- **Bypassing:** In $R=3$ mode, the later 3 stages of the MAC unit are bypassed, not disabled, to maintain pipeline integrity [S1].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of frequency bands $N$ and the dimension $R$. The pipelined nature allows for high throughput per sample point.
- **Memory Complexity:** Fixed memory footprint of $2 \times (3 \times 128)$ elements for the frequency matrix storage, regardless of the number of sample points [S1].
- **Performance:** Designed to accelerate the MLP component of NeRF, which is the most computation-intensive part due to tens of millions of MLP queries [S1].
- **Strengths:** Hardware efficiency for inner product operations; power reduction in $R=3$ mode; support for both isotropic and anisotropic random Fourier features [S1].
- **Limitations:** Fixed support for $R=3$ and $R=6$; requires pre-loading of frequency matrix; specific to NeRF rendering tasks [S1].

## Background
The PEU was developed as part of the ICARUS architecture, a specialized hardware system for Neural Radiance Fields rendering [S1]. It addresses the computational bottleneck of MLP queries in NeRF pipelines [S1]. The design supports various positional encoding algorithms, including isotropic random Fourier features [Tancik et al. 2020] and anisotropic random Fourier features [Yu et al. 2022], as cited in [S1].

## Sources
[S1] ICARUS: A Specialized Architecture for Neural Radiance Fields Rendering, https://doi.org/10.48550/arxiv.2203.01414, fetched 2026-10-26
