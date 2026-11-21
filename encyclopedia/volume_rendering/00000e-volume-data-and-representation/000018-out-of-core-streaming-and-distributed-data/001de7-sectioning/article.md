# Sectioning
## Overview
Sectioning is a hardware-oriented memory management technique used in real-time ray-casting pipelines to constrain the size of on-chip First-In-First-Out (FIFO) buffers [S1, unofficial]. It solves the problem of limited semiconductor memory by partitioning the volume dataset into smaller spatial sections, off-loading intermediate rendering values to external memory between sections, and then recirculating them [S1, unofficial]. It maintains exact visual fidelity without approximation, as the exact same values are written to and read from external memory as would have been passed directly between pipeline stages [S1, unofficial].
## How it works
### Inputs and Outputs
- **Input**: A 3D volume dataset partitioned into sections, the current section being processed, and the Section Memory containing intermediate values from the previous section [S1, unofficial].
- **Output**: A rendered image frame; intermediate values (e.g., transmittance, accumulated color) are written to and read from Section Memory [S1, unofficial].

### Data Structures
- **Section Memory**: An external memory block (e.g., SDRAM) acting as a large FIFO that holds values recirculated in the processing direction [S1, unofficial].
- **Section**: A spatial partition of the volume; in the reference implementation, sections are 32 voxels wide in the x-direction, corresponding to the hardware's memory organization [S1, unofficial].

### Steps
1. Partition the volume dataset into sections along the x, y, or z direction, though implementations may restrict this (e.g., x-direction only) [S1, unofficial].
2. Modify the voxel traversal order: voxels are read in slice-by-slice order within a single section, and within slices, in beam-by-beam order, with beams spanning only the width of the current section [S1, unofficial].
3. Process the full section front-to-back [S1, unofficial].
4. At the end of a section, shift registers in the right-hand pipeline are modified to optionally write the intermediate values needed for rendering near the right boundary to Section Memory [S1, unofficial].
5. Begin processing the next section [S1, unofficial].
6. At the start of the new section, the left-hand pipeline reads the intermediate values from Section Memory in the exact same order they were written, passing them through multiplexers to the left pipeline [S1, unofficial].
7. Continue processing; the pipeline operates as if the values had been generated on the previous cycle and passed directly, ensuring no approximation or visual artifacts at section boundaries [S1, unofficial].

### Formulas
The method guarantees exact value preservation across boundaries:
$$ V_{\text{read}} = V_{\text{written}} $$
Where $V_{\text{read}}$ is the value retrieved from Section Memory and $V_{\text{written}}$ is the value stored there [S1, unofficial].

### Parameters
- **Section Width**: 32 voxels in the x-direction (in the VolumePro implementation) [S1, unofficial].
- **Partitioning Direction**: x, y, or z (theoretically); x-direction only (in the cited VolumePro implementation) [S1, unofficial].

### Edge Cases and Pitfalls
- **Visual Artifacts**: The method ensures no visual artifacts are introduced at section boundaries because the exact same values are written and read, maintaining the logical continuity of the ray-casting pipeline [S1, unofficial].
- **Memory Organization**: The fixed section width (32 voxels) is constrained by the specific hardware memory organization and datapath width [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sectioning_render(volume_data, section_width=32, direction='x'):
    """
    Processes a volume dataset using sectioning to manage on-chip FIFO memory.
    volume_data: 3D array of voxels
    section_width: width of each section in voxels (default 32)
    direction: partitioning direction ('x', 'y', or 'z')
    """
    # Determine number of sections along the chosen axis
    if direction == 'x':
        num_sections = volume_data.shape[0] // section_width
    elif direction == 'y':
        num_sections = volume_data.shape[1] // section_width
    else:
        num_sections = volume_data.shape[2] // section_width

    for sec_idx in range(num_sections):
        section_memory = []  # External FIFO for intermediate values

        # Traverse voxels: slice-by-slice, then beam-by-beam within section
        for slice_idx in range(volume_data.shape[1]):
            for beam_idx in range(volume_data.shape[2]):
                for voxel_offset in range(section_width):
                    # Compute absolute voxel coordinates based on section and direction
                    if direction == 'x':
                        x = sec_idx * section_width + voxel_offset
                        y, z = slice_idx, beam_idx
                    elif direction == 'y':
                        x, y = slice_idx, sec_idx * section_width + voxel_offset
                        z = beam_idx
                    else:
                        x, y, z = slice_idx, beam_idx, sec_idx * section_width + voxel_offset

                    # Process voxel and compute intermediate values
                    intermediate_value = compute_intermediate_value(volume_data[x, y, z])

                    # At the end of the section, write to external memory
                    if voxel_offset == section_width - 1:
                        section_memory.append(intermediate_value)
                    else:
                        # Pass directly to next pipeline stage
                        pass_to_next_pipeline(intermediate_value)

        # At the start of the next section, left pipeline reads from Section Memory
        # Values are read in the exact same order they were written
        for value in section_memory:
            pass_to_left_pipeline(value)

def compute_intermediate_value(voxel):
    # Standard ray-casting accumulation step (e.g., transmittance and color)
    return voxel.transmittance, voxel.color

def pass_to_next_pipeline(value):
    # Standard pipeline forwarding
    pass

def pass_to_left_pipeline(value):
    # Standard pipeline forwarding for recirculated values
    pass
```
## Operating characteristics
- **Time Complexity**: Not given in the sources. The method adds overhead for writing to and reading from external memory, which is slower than on-chip FIFOs, but allows the pipeline to continue without stalling for memory constraints [G].
- **Memory Complexity**: Reduces on-chip memory requirements by off-loading intermediate values to external memory [S1, unofficial]. The size of Section Memory depends on the number of intermediate values that need to be recirculated [G].
- **Performance**: Designed for real-time rendering on hardware (e.g., VolumePro PCI card) [S1, unofficial]. Maintains visual fidelity without approximation [S1, unofficial].
- **Strengths**: Exact visual continuity; no approximation; fits within hardware memory constraints [S1, unofficial].
- **Limitations**: Requires specific hardware support (modified shift registers, multiplexers, external memory interface); limited to the memory organization of the hardware (e.g., 32-voxel sections) [S1, unofficial].
- **When to Use**: When rendering large volumes on hardware with limited on-chip memory and exact visual fidelity is required [G].
- **When Not to Use**: When software flexibility is needed, or when the hardware does not support the required memory management features [G].
## Background
Sectioning was introduced in the VolumePro real-time ray-casting system [S1, unofficial]. It is described in the technical report "The VolumePro Real-Time Ray-Casting System" [S1, unofficial]. The method is part of the hardware architecture of the VolumePro PCI card, which uses a vg500 rendering ASIC and SDRAMs [S1, unofficial]. The technique is cited as building on previous work [9], which is not provided in the sources [S1, unofficial].
## Sources
[S1] The VolumePro Real-Time Ray-Casting System, https://www.merl.com/publications/docs/TR99-19.pdf, fetched 2026-10-26 (unofficial)
