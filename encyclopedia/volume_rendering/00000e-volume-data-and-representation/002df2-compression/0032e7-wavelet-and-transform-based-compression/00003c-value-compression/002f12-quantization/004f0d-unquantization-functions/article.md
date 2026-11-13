# Unquantization Functions
Also called: Unquantize10, Unquantize7, Unquantize9
## Overview
Unquantization functions convert integer quantized values back to half-precision float range, serving as the inverse operation to the quantization step in transform-based volume data compression [S1, unofficial]. Unlike general scalar quantization which maps continuous domains to discrete indices, these specific functions are tailored for the BC6H compression format used in real-time graphics, handling specific bit-depths (7, 9, 10) to reconstruct floating-point color or density values from compressed integer representations [S1, unofficial].
## How it works
### Inputs and outputs
- **Input**: An integer value $\text{Int}(x)$ representing the quantized coefficient or color component, and a scaling factor $\text{Scale}$ derived from the compression endpoint selection.
- **Output**: A half-precision floating-point value $\text{Half}(x)$ in the range $[-65504, 65504]$ (or similar half-float range depending on implementation).

### Data structures
The implementation relies on standard integer types for the input and half-precision float types for the output. No complex data structures are required for the core unquantization logic itself, though it operates within the context of compressed blocks (e.g., 128-bit BC6H blocks).

### Steps and formulas
The core unquantization formula maps the integer index to a floating-point value using a linear scaling and offset:

$$
\text{Half}(x) = \frac{\text{Int}(x) \times 65536 + 32768}{\text{Scale}}
$$

Where:
- $\text{Int}(x)$ is the quantized integer value.
- $65536$ is the scaling multiplier (often related to the bit-depth or fixed-point representation).
- $32768$ is the rounding offset to ensure correct rounding during the division.
- $\text{Scale}$ is the divisor determined by the compression mode and endpoint selection.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def unquantize(int_val: int, scale: float) -> float:
    """
    Converts an integer quantized value back to a half-precision float range.
    
    Args:
        int_val: The quantized integer value.
        scale: The scaling factor derived from compression parameters.
        
    Returns:
        The unquantized float value.
    """
    # Apply scaling and rounding offset
    numerator = int_val * 65536 + 32768
    result = numerator / scale
    return result
```

### Edge cases and pitfalls
- **Division by zero**: Ensure $\text{Scale}$ is never zero. In BC6H, certain modes may have specific constraints on valid scales.
- **Precision loss**: The use of fixed-point arithmetic (multiplication by 65536) may introduce precision issues if not handled with sufficient integer width before division.
- **Range clamping**: The output should be clamped to the valid half-precision float range to avoid NaNs or infinities if the input is out of expected bounds.

## Subtopics

### Finish Unquantize
Interpolates between two unquantized endpoints using a weight and converts to final float3. This variant is used in BC6H compression to reconstruct the final color value by blending between two base unquantized values based on a weight index.
- **Formula**:
  $$
  \text{Result} = \text{f16tof32}\left( \left( \text{endpoint0Unq} \times (64 - \text{weight}) + \text{endpoint1Unq} \times \text{weight} + 32 \right) \times \frac{31}{4096} \right)
  $$
- **Details**:
  - $\text{endpoint0Unq}$ and $\text{endpoint1Unq}$ are the two unquantized endpoint values (half-precision floats).
  - $\text{weight}$ is an integer index (typically 0-63) determining the interpolation factor.
  - $64 - \text{weight}$ and $\text{weight}$ are the interpolation coefficients.
  - $+ 32$ is a rounding offset.
  - $\times \frac{31}{4096}$ scales the result to the final range.
  - $\text{f16tof32}$ converts the intermediate half-precision result to a full float32 for final output or further processing.
- **When to choose**: Use this when reconstructing pixel values from BC6H compressed blocks where interpolation between endpoints is required.

## Operating characteristics
- **Time complexity**: $O(1)$ per value, as it involves a constant number of arithmetic operations.
- **Memory complexity**: $O(1)$, requiring only temporary storage for intermediate calculations.
- **Performance**: Highly efficient on GPU hardware due to simple arithmetic operations. Suitable for real-time decompression.
- **Strengths**: Fast, deterministic, and well-suited for hardware acceleration.
- **Limitations**: Limited to specific compression formats (BC6H). Precision is constrained by the half-precision float range and the quantization step size.
- **When to use**: Real-time graphics applications requiring efficient decompression of compressed textures or volume data.
- **When not to use**: Applications requiring high-precision scientific data reconstruction where lossy compression artifacts are unacceptable.

## Background
Originating from the BC6H compression format, these unquantization functions are part of the broader family of transform-based compression techniques used in real-time graphics. BC6H is designed for high-quality compression of floating-point HDR images, and these functions play a crucial role in the decompression pipeline. The specific variants (Unquantize10, Unquantize7, Unquantize9) correspond to different bit-depths used in the compression process to balance quality and bitrate.

## Sources
[S1] Real-time BC6H compression on GPU, https://github.com/knarkowicz/GPURealTimeBC6H, fetched 2026-10-27 (unofficial)
