# 3D Pixel Shuffle

Also called: 3D sub-pixel convolution, PixelShuffle3D, sub-pixel convolution (3D)

## Overview
3D Pixel Shuffle is a parameter-free upsampling method that increases the spatial resolution of a 3D volume by an integer factor by rearranging channel data into spatial dimensions [S1, unofficial]. It differs from interpolation-based siblings like [Zero Padding for Aliasing Reduction](6382-zero-padding-for-aliasing-reduction.md) and [Resized Slice Centering](12150-resized-slice-centering.md) by requiring no learnable weights or kernel operations, instead performing a deterministic tensor reshape and permutation [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A 3D tensor $I$ with shape $(B, C, D, H, W)$, where $B$ is batch size, $C$ is channels, and $D, H, W$ are depth, height, and width.
- Output: A 3D tensor $O$ with shape $(B, C', D', H', W')$, where $C' = C / f^3$, $D' = D \cdot f$, $H' = H \cdot f$, $W' = W \cdot f$, and $f$ is the upscale factor.

**Data Structures:**
- The method operates on dense 3D tensors. No auxiliary data structures (like grids or trees) are required.

**Parameters:**
- `upscale_factor` ($f$): An integer $\ge 1$ specifying the factor by which to increase spatial dimensions [S1, unofficial].

**Steps:**
1. Verify that $D$, $H$, and $W$ are exactly divisible by $f$. If not, raise a `ValueError` [S1, unofficial].
2. Reshape the input tensor $I$ from $(B, C, D, H, W)$ to an intermediate shape $(B, C, D/f, f, H/f, f, W/f, f)$ [S1, unofficial].
3. Permute the dimensions to reorder the tensor. The permutation maps indices $(0, 1, 2, 3, 4, 5, 6, 7)$ to $(0, 1, 3, 5, 7, 2, 4, 6)$ [S1, unofficial].
4. Reshape the permuted tensor to the final output shape $(B, C/f^3, D \cdot f, H \cdot f, W \cdot f)$ [S1, unofficial].

**Formula:**
$$
O = \text{reshape}\left(\text{permute}\left(\text{reshape}\left(I, \left(B, C, \frac{D}{f}, f, \frac{H}{f}, f, \frac{W}{f}, f\right)\right), (0,1,3,5,7,2,4,6)\right), \left(B, \frac{C}{f^3}, D \cdot f, H \cdot f, W \cdot f\right)\right)
$$
where $f$ is `upscale_factor` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pixel_shuffle_3d(x, upscale_factor):
    """
    Upsamples a 3D tensor using pixel shuffle.
    
    Args:
        x: Tensor of shape (B, C, D, H, W)
        upscale_factor: Integer factor f
        
    Returns:
        Tensor of shape (B, C//f^3, D*f, H*f, W*f)
    """
    B, C, D, H, W = x.shape
    
    # Check divisibility
    if D % upscale_factor != 0 or H % upscale_factor != 0 or W % upscale_factor != 0:
        raise ValueError("Input dimensions must be divisible by upscale_factor")
        
    # Step 1: Reshape to separate spatial blocks
    # Shape becomes (B, C, D//f, f, H//f, f, W//f, f)
    x = x.view(B, C, D // upscale_factor, upscale_factor,
               H // upscale_factor, upscale_factor,
               W // upscale_factor, upscale_factor)
               
    # Step 2: Permute dimensions
    # Map indices: 0->0, 1->1, 2->3, 3->5, 4->7, 5->2, 6->4, 7->6
    # Original indices: 0:B, 1:C, 2:D//f, 3:f, 4:H//f, 5:f, 6:W//f, 7:f
    # New order: B, C, f_d, f_h, f_w, D//f, H//f, W//f
    x = x.permute(0, 1, 3, 5, 7, 2, 4, 6)
    
    # Step 3: Reshape to final output
    # Shape becomes (B, C//f^3, D*f, H*f, W*f)
    x = x.contiguous().view(B, C // (upscale_factor ** 3),
                            D * upscale_factor,
                            H * upscale_factor,
                            W * upscale_factor)
                            
    return x
```

**Edge Cases and Pitfalls:**
- Input dimensions $D, H, W$ must be exactly divisible by `upscale_factor`. If not, the operation fails [S1, unofficial].
- The number of input channels $C$ must be divisible by $f^3$. If not, the final reshape will fail due to size mismatch [G].

## Operating characteristics
**Time and Memory Complexity:**
- Time complexity is $O(N)$ where $N$ is the number of elements in the input tensor, as it involves only reshaping and permutation operations [S1, unofficial].
- Memory complexity is $O(N)$, requiring no additional storage beyond the input and output tensors [S1, unofficial].

**Measured Performance:**
- Not given in the sources.

**Scaling:**
- Scales linearly with input size. Efficient on GPUs due to contiguous memory operations [S1, unofficial].

**Strengths and Limitations:**
- Strengths: Parameter-free, fast, deterministic, and preserves all input information without interpolation artifacts [S1, unofficial].
- Limitations: Only supports integer upscale factors; requires channel count to be divisible by $f^3$ [S1, unofficial].

**When to use it and when not:**
- Use when integer upsampling is required without learnable parameters or interpolation smoothing [S1, unofficial].
- Do not use when non-integer upsampling is needed or when the channel count is not a multiple of $f^3$ [S1, unofficial].

## Background
3D Pixel Shuffle is an extension of the 2D Pixel Shuffle (sub-pixel convolution) introduced by Shi et al. for 2D image super-resolution [S1, unofficial]. It has been adopted in 3D deep learning frameworks for tasks such as 3D volume super-resolution and compression [S1, unofficial].

## Sources
[S1] asheibanifard/EndtoEndCompression, https://github.com/asheibanifard/EndtoEndCompression, fetched 2026-10-27 (unofficial)
