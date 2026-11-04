# Ray Compositing and Accumulation

Also called: Hardware Ray Compositing, Ray Beam FIFO, Ray Shift Register, Ray Slice FIFO

## Overview
Ray Compositing and Accumulation is the process of combining color and opacity contributions from volume samples into final pixel values for semi-transparent rendering [S1, unofficial]. It solves the problem that volume samples are often generated in slice-by-slice or beam-by-beam order rather than strict ray order, requiring buffering to maintain correct compositing sequence [S1, unofficial]. Unlike Alpha Compositing Ray Casting, which assumes a linear ray traversal, this method handles arbitrary view directions and angled rays by using hardware buffers to align predecessor values for each sample [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A stream of color ($C_{sample}$) and opacity ($\alpha_{sample}$) values at sample points, presented in slice-by-slice, beam-by-beam order [S1, unofficial].
*   **Output:** Final pixel values written to Pixel Memory, which are then transferred to a companion 3D graphics card for image warping [S1, unofficial].

**Data Structures:**
The method relies on three specific hardware buffers to store partially accumulated ray values [S1, unofficial]:
1.  **Ray Slice FIFO:** Stores partially accumulated pixel values for compositing with the next slice [S1, unofficial].
2.  **Ray Beam FIFO:** Stores values for an additional beam time duration to handle downward-angled rays, allowing input from the beam above and before the current sample [S1, unofficial].
3.  **Ray Shift Register:** Stores values for one additional cycle to handle samples to the left of the current one [S1, unofficial].

**Parameters:**
*   **Precision:** 12 bits are maintained for accumulated color ($C_{acc}$), accumulated opacity ($\alpha_{acc}$), and all intermediate compositing values to ensure correct alpha blending in large volumes [S1, unofficial].

**Steps:**
1.  **Determine Predecessor:** For a sample $S(x,y,z)$, identify the correct predecessor value from the previous slice. The predecessor is one of $S(x,y,z-1)$, $S(x-1,y,z-1)$, $S(x,y-1,z-1)$, or $S(x-1,y-1,z-1)$, selected based on the view direction and the $z$ value of the sample [S1, unofficial].
2.  **Select Predecessor:** Use a Ray Alignment unit to drive two multiplexers (MUX) that select the correct predecessor value from the Ray Slice FIFO, Ray Beam FIFO, or Ray Shift Register [S1, unofficial].
3.  **Composite:** Combine the current sample's color and opacity with the selected predecessor value using the chosen blending mode (e.g., Front-to-back blending) [S1, unofficial].
4.  **Buffer:** Store the resulting partially accumulated value back into the appropriate buffer (Ray Slice FIFO, Ray Beam FIFO, or Ray Shift Register) for the next sample in the stream [S1, unofficial].
5.  **Output:** When a ray passes through the back of the volume (maximum $z$) or a side face, output the final accumulated pixel value [S1, unofficial].
6.  **Write:** Write base plane pixel values to Pixel Memory [S1, unofficial].

**Formulas:**
For Front-to-back blending, the accumulated color $C_{acc}$ and accumulated opacity $\alpha_{acc}$ are updated as follows [S1, unofficial]:
$$ C_{acc}^{new} = C_{acc}^{old} + (1 - \alpha_{acc}^{old}) \cdot C_{sample} $$
$$ \alpha_{acc}^{new} = \alpha_{acc}^{old} + (1 - \alpha_{acc}^{old}) \cdot \alpha_{sample} $$

Other blending modes supported include Minimum Intensity and Maximum Intensity, which compare $C_{sample}$ against $C_{acc}$ based on thresholds [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Hardware simulation of Ray Compositing and Accumulation
# Assumes a stream of samples (C_sample, alpha_sample) in slice/beam order

# Initialize buffers
# Ray Slice FIFO: stores values for next slice compositing
# Ray Beam FIFO: stores values for angled ray compositing
# Ray Shift Register: stores values for left-neighbor compositing
ray_slice_fifo = Buffer()
ray_beam_fifo = Buffer()
ray_shift_register = Buffer()

# Precision setting
PRECISION_BITS = 12

def composite_front_to_back(c_acc, alpha_acc, c_sample, alpha_sample):
    """Perform front-to-back alpha blending."""
    # Apply precision constraints if necessary
    c_acc = quantize(c_acc, PRECISION_BITS)
    alpha_acc = quantize(alpha_acc, PRECISION_BITS)
    
    c_new = c_acc + (1.0 - alpha_acc) * c_sample
    alpha_new = alpha_acc + (1.0 - alpha_acc) * alpha_sample
    
    return quantize(c_new, PRECISION_BITS), quantize(alpha_new, PRECISION_BITS)

def get_predecessor(view_direction, z, x, y):
    """
    Determine which buffer holds the correct predecessor value.
    This is a simplified logic based on [S1, unofficial].
    """
    # Logic depends on view direction and z value
    # Returns value from one of the FIFOs/Registers
    # Note: Exact mapping requires Ray Alignment unit logic not fully detailed in text
    # [G] Placeholder for Ray Alignment unit logic
    return ray_slice_fifo.read() # Simplified assumption

def process_sample_stream(samples):
    """
    Process a stream of samples.
    samples: list of (C_sample, alpha_sample, x, y, z, view_direction)
    """
    pixel_memory = {} # Map (x,y) -> (C_acc, alpha_acc)
    
    for sample in samples:
        c_sample, alpha_sample, x, y, z, view_dir = sample
        
        # 1. Determine Predecessor
        # [G] In hardware, this is done by Ray Alignment unit driving MUXes
        # We assume the correct buffer is selected based on geometry
        c_acc, alpha_acc = get_predecessor(view_dir, z, x, y)
        
        # 2. Composite
        c_new, alpha_new = composite_front_to_back(c_acc, alpha_acc, c_sample, alpha_sample)
        
        # 3. Buffer for next steps
        # [G] Logic to write back to FIFOs/Registers based on view direction
        # This is implicit in the hardware description
        update_buffers(x, y, z, view_dir, c_new, alpha_new)
        
        # 4. Output if ray ends
        if is_ray_end(z, view_dir):
            pixel_memory[(x, y)] = (c_new, alpha_new)
            
    return pixel_memory

def update_buffers(x, y, z, view_dir, c_acc, alpha_acc):
    """
    Update Ray Slice FIFO, Ray Beam FIFO, and Ray Shift Register.
    [G] Specific update logic depends on view direction and beam alignment
    """
    # Simplified: always update slice FIFO for next slice
    ray_slice_fifo.write((x, y), (c_acc, alpha_acc))
    # [G] Ray Beam FIFO and Shift Register updates depend on specific ray angles
```

## Subtopics

### Scatter-Add Accumulation
Scatter-Add Accumulation accumulates weighted values along rays using scatter-add operations, supporting both feature accumulation and weight-only accumulation [S2, unofficial]. It is used when values are provided to compute weighted features ($weighted = weights[:, None] * values$) or when accumulating weights only ($output = output + weights[:, None]$) [S2, unofficial]. This method initializes an output buffer of shape $[n\_rays, C]$ and performs scatter-add using ray indices ($output = output.at[ray\_indices].add(weighted)$) [S2, unofficial]. It is suitable for implementations where ray indices are known and random access to the output buffer is efficient [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of samples, as each sample is processed once [S1, unofficial].
*   **Memory Complexity:** Proportional to the number of rays and the depth of the FIFOs/Registers, which depend on the volume size and view angle [S1, unofficial].
*   **Strengths:** Handles non-ray-order sample presentation efficiently in hardware; supports multiple blending modes [S1, unofficial].
*   **Limitations:** Requires significant hardware resources for FIFOs and alignment logic; precision loss can occur if bit depth is insufficient for large volumes [S1, unofficial].
*   **When to use:** Real-time ray casting on hardware architectures like VolumePro [S1, unofficial].
*   **When not to use:** Software implementations where memory bandwidth is limited and FIFO simulation is expensive [S1, unofficial].

## Background
Ray Compositing and Accumulation was developed as part of the VolumePro real-time ray-casting system [S1, unofficial]. It was designed to handle the specific data flow of slice-by-slice, beam-by-beam rendering on dedicated hardware [S1, unofficial]. The method addresses the challenge of maintaining correct compositing order when samples are not generated in strict ray order [S1, unofficial].

## Sources
[S1] The VolumePro Real-Time Ray-Casting System, https://www.merl.com/publications/docs/TR99-19.pdf, fetched 2026-10-26 (unofficial)
[S2] RobotFlow-Labs/gsplat-mlx, https://github.com/RobotFlow-Labs/gsplat-mlx, fetched 2026-10-27 (unofficial)
