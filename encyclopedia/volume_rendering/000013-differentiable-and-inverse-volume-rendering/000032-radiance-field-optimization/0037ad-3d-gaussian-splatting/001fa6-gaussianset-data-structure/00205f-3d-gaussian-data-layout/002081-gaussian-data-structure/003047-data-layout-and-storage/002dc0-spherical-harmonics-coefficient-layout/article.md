# Spherical Harmonics Coefficient Layout

Also called: Per-Dimension SH Storage, SH Buffer Layout

## Overview
This method defines the memory layout for storing Spherical Harmonics (SH) coefficients for 3D Gaussian primitives in a Structure-of-Arrays (SoA) format. It solves the problem of efficiently accessing per-splat color information for rendering and optimization by grouping coefficients by splat, then by coefficient index, and finally by RGB channel. It differs from texture-based or codebook-based layouts by keeping all coefficients in a single contiguous linear buffer, enabling direct indexing without texture lookups or codebook decoding.

## How it works
**Inputs:**
- $N$: Number of Gaussian splats.
- $D$: Total number of SH coefficients per channel for the chosen degree (e.g., $D=9$ for degree 2).
- Coefficient data: For each splat $i \in [0, N-1]$, for each coefficient index $d \in [0, D-1]$, and for each RGB channel $c \in \{0, 1, 2\}$, a floating-point value.

**Outputs:**
- A contiguous `FloatBuffer` of size $N \times D \times 3$.

**Data Structure:**
- `FloatBuffer`: A 1D array of floats.
- Layout order: The buffer is organized such that for a given splat $i$, all coefficients $d$ are stored sequentially, and for each coefficient, the three RGB channels are stored contiguously.

**Indexing Formula:**
To access the component $c$ (0=R, 1=G, 2=B) of coefficient $d$ for splat $i$:
$$ idx = ((i \times D) + d) \times 3 + c $$

**Steps:**
1. Initialize a `FloatBuffer` of size $N \times D \times 3$.
2. Iterate over each splat index $i$ from $0$ to $N-1$.
3. For each splat $i$, iterate over each coefficient index $d$ from $0$ to $D-1$.
4. For each coefficient $d$, write the three RGB components to the buffer at indices:
   - $((i \times D) + d) \times 3 + 0$ for Red
   - $((i \times D) + d) \times 3 + 1$ for Green
   - $((i \times D) + d) \times 3 + 2$ for Blue
5. Return the populated buffer.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def write_sh_coefficients(splats, sh_degree):
    """
    Writes SH coefficients into a flat buffer.
    
    Args:
        splats: List of splat objects, each having get_sh_r(d), get_sh_g(d), get_sh_b(d) methods.
        sh_degree: Integer, the degree of SH used.
        
    Returns:
        list: A flat list of floats representing the SH buffer.
    """
    # Calculate total number of coefficients per channel for the given degree
    # Common formula: D = (sh_degree + 1)^2
    D = (sh_degree + 1) ** 2
    
    N = len(splats)
    buffer_size = N * D * 3
    buffer = [0.0] * buffer_size
    
    for i in range(N):
        splat = splats[i]
        for d in range(D):
            # Calculate base index for this coefficient
            base_idx = (i * D + d) * 3
            
            # Write RGB components
            buffer[base_idx + 0] = splat.get_sh_r(d)
            buffer[base_idx + 1] = splat.get_sh_g(d)
            buffer[base_idx + 2] = splat.get_sh_b(d)
            
    return buffer
```

**Parameters:**
- `sh_degree`: Typically 0, 1, 2, or 3. Degree 2 ($D=9$) is common for high-quality rendering [S1, unofficial].
- `D`: Calculated as $(sh\_degree + 1)^2$.

**Edge Cases and Pitfalls:**
- **Degree Mismatch:** Ensure $D$ matches the degree used during training or optimization. Using a different degree will misalign indices [S1, unofficial].
- **Precision:** Use `float32` for GPU compatibility. `float64` may cause performance issues or incompatibility with standard shaders [S1, unofficial].
- **Boundary Conditions:** The buffer size must be exactly $N \times D \times 3$. Any deviation will cause out-of-bounds errors or data corruption [S1, unofficial].

## Subtopics

### Spherical Harmonics Texture Management
This variant stores SH coefficients in separate GPU textures keyed by block identifiers rather than a single flat buffer. It is used in streaming or LOD-based systems where coefficients are managed in chunks. It differs from the flat buffer layout by requiring texture creation and update logic, which adds overhead but allows for efficient memory management of large scenes. Choose this when dealing with very large scenes or streaming data where not all coefficients are loaded into memory at once [S2, unofficial].

## Operating characteristics
**Time Complexity:**
- Writing: $O(N \times D)$, where $N$ is the number of splats and $D$ is the number of coefficients per channel.
- Reading: $O(1)$ per coefficient access using the indexing formula.

**Memory Complexity:**
- $O(N \times D \times 3)$ floats. For $N=10^6$ and $D=9$, this is approximately 108 MB.

**Performance:**
- The flat buffer layout minimizes cache misses during batched rendering, as shaders can iterate over contiguous memory for a single attribute [Parent].
- Texture-based layouts may introduce overhead due to texture sampling and management but can be more memory-efficient for sparse or streamed data [S2, unofficial].

**Strengths:**
- Simple indexing and easy to implement.
- Efficient for batched operations.
- No need for texture management or codebook decoding.

**Limitations:**
- Not suitable for streaming or LOD-based systems without additional management logic.
- Requires loading all coefficients into memory at once.

**When to Use:**
- Use this layout for static scenes or when all coefficients are available in memory.
- Use texture-based layouts for streaming or LOD-based systems [S2, unofficial].

## Background
This layout is derived from the Structure-of-Arrays (SoA) paradigm used in 3D Gaussian Splatting implementations. It is commonly found in libraries like JSplat, which provide efficient data structures for rendering and optimization [S1, unofficial]. The texture-based variant is used in systems like Gaussian-Splat-Lite for streaming and LOD management [S2, unofficial].

## Sources
[S1] JSplat, https://github.com/javagl/JSplat, fetched 2026-10-30 (unofficial)
[S2] WilliamLiu-1997/Gaussian-Splat-Lite, https://github.com/WilliamLiu-1997/Gaussian-Splat-Lite, fetched 2026-10-27 (unofficial)
