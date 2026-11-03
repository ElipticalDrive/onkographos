# Progressive Encoding of Time-Varying Sequences

Also called: I-frame and P-frame encoding for volume data, Temporal codebook reuse

## Overview
This method compresses time-varying volumetric data by encoding frames individually while reusing codebooks from previous frames to improve compression performance and encoding speed [S1]. It distinguishes itself from static vector quantization by introducing temporal dependencies between frames, treating the sequence as a series of I-frames (independently encoded) and P-frames (predicted via codebook retraining) [S1]. This approach significantly reduces memory requirements and encoding time compared to encoding each frame independently [S1].

## How it works
The method applies Vector Quantization (VQ) to each time step of a volumetric sequence [S1]. It utilizes a hierarchical quantization scheme where codebooks are either generated fresh or retrained based on previous frames [S1].

### Inputs and Outputs
*   **Input:** A sequence of $T$ volumetric frames, where each frame is a 3D grid of data values (e.g., scalar fields) [S1].
*   **Output:** For each frame, an index texture mapping grid positions to codebook indices. For I-frames, the associated local codebooks are also stored [S1].

### Data Structures
*   **Index Texture:** A 3D texture storing quantization indices for the volume. The source notes that 16-bit mean values can be encoded in the R- and alpha-components of the index texture, using the alpha-channel to store higher-resolution data [S1].
*   **Codebook:** A set of representative vectors (centroids) used for quantization. In the described scheme, two local codebooks are produced for I-frames [S1].

### Algorithm Steps
The encoding process alternates between two modes: I-frame (separate encoding) and P-frame (progressive retraining) [S1].

1.  **Initialize:** For the first frame ($t=0$), encode it as an I-frame [S1].
    *   Apply the LBG (Linde-Buzo-Gray) algorithm to generate initial codebooks [S1].
    *   Store the resulting index texture and codebooks [S1].
2.  **Iterate for subsequent frames ($t > 0$):**
    *   **Decision:** Determine if frame $t$ is a key-frame (I-frame) or a predicted frame (P-frame) based on a predefined key-frame interval [S1].
    *   **If I-frame:**
        *   Encode the frame separately using the LBG algorithm from scratch [S1].
        *   Produce an index texture and 2 local codebooks [S1].
        *   Update the "last key-frame" reference to $t$ [S1].
    *   **If P-frame:**
        *   **Reuse Codebooks:** Take the codebooks from the most recent key-frame as the *initial* codebooks for the LBG algorithm [S1].
        *   **Reuse Indices:** Use the index sets from the previous frame ($t-1$) to obtain a first guess for the final quantization [S1].
        *   **Retrain:** Perform LBG relaxation on the current frame's data using the reused codebooks and initial index guess. This allows for fast and partial searches throughout the process [S1].
        *   Store the new index texture [S1].
3.  **Termination:** Repeat until all $T$ frames are encoded [S1].

### Parameters
*   **Key-frame Interval:** The number of P-frames between two I-frames. In the shock wave simulation example, a key-frame was used every 10 steps (e.g., time step 65 was encoded separately, while time step 55 was the last key-frame) [S1].
*   **LBG Algorithm:** The specific implementation of the Linde-Buzo-Gray algorithm is used for codebook generation and retraining [S1].

### Edge Cases and Pitfalls
*   **Codebook Drift:** Choosing P-frame mode for all but the first frame leads to sub-optimal codebooks because the LBG algorithm depends on the choice of initial codebooks. Over time, the codebooks may drift away from the optimal representation for the current data distribution [S1].
*   **Precision Loss:** If the original data has high precision (e.g., 16-bit), quantizing it directly to low-bit indices (e.g., 8-bit) can introduce large distortions. The source notes that the original vortex data was quantized to 8 bits in advance to allow encoding the mean value into an 8-bit color component without large distortions [S1].
*   **High-Resolution Mean Values:** To handle higher precision, the alpha-channel of the index texture can be used to encode 16-bit mean values in the R- and alpha-components, significantly improving fidelity for higher-resolved data samples [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_time_varying_sequence(frames, key_frame_interval):
    """
    Encodes a sequence of volumetric frames using progressive encoding.
    
    Args:
        frames: List of volumetric data arrays, where frames[t] is the data at time t.
        key_frame_interval: Integer, the number of P-frames between I-frames.
        
    Returns:
        encoded_sequence: List of encoded frame objects.
    """
    encoded_sequence = []
    last_key_frame_codebooks = None
    previous_indices = None
    last_key_frame_t = -key_frame_interval  # Ensures t=0 is treated as a key-frame

    for t in range(len(frames)):
        is_key_frame = (t == 0) or (t - last_key_frame_t >= key_frame_interval)

        if is_key_frame:
            # I-Frame Mode: Encode separately, producing index texture and 2 local codebooks [S1]
            # [G] Sources do not specify the exact LBG implementation. Standard LBG split-and-assign recursively splits centroids, assigns vectors, and updates centroids until convergence.
            current_codebooks = linde_buzo_gray_split_and_assign(frames[t])
            current_indices = find_nearest_codebook(frames[t], current_codebooks)
            
            encoded_sequence.append({
                'type': 'I',
                'indices': current_indices,
                'codebooks': current_codebooks,
                'time_step': t
            })
            
            last_key_frame_codebooks = current_codebooks
            last_key_frame_t = t
            previous_indices = current_indices
        else:
            # P-Frame Mode: Retrain codebooks from previous frame [S1]
            initial_codebooks = last_key_frame_codebooks
            # Reuse index sets from the previous frame to obtain a first guess [S1]
            initial_indices = previous_indices
            
            # [G] Sources do not specify the exact LBG implementation. Standard LBG relaxation performs a partial search using initial_indices and updates centroids starting from initial_codebooks.
            current_codebooks = linde_buzo_gray_relax(frames[t], initial_codebooks, initial_indices)
            current_indices = find_nearest_codebook(frames[t], current_codebooks)
            
            encoded_sequence.append({
                'type': 'P',
                'indices': current_indices,
                'time_step': t
            })
            
            previous_indices = current_indices

    return encoded_sequence
```

## Subtopics

### Vector Quantization
The foundational lossy compression method that maps n-dimensional vectors to codebook indices. Progressive encoding builds on this by adding temporal dependencies. [Vector Quantization](907-vector-quantization.md)

## Operating characteristics
*   **Memory:** Significant reduction in memory requirements. For a shock wave simulation (89 time steps, $256^3$ size), memory dropped from 1.4 GB to 70 MB. For a vortex flow simulation (100 time steps, $128^3$ size), it dropped from 200 MB to 11.1 MB [S1].
*   **Encoding Time:** Encoding time decreased from 29 minutes to 13 minutes for an entire shock wave sequence when using progressive encoding compared to separate encoding for all steps. Quantization of the vortex data set took roughly 5 minutes using the progressive scheme [S1].
*   **Rendering:** Rendering performance was 24 fps and 16 fps for the respective simulations on the target hardware (P4 2.8 GHz, 512 MB RAM, ATI 9700). A considerable speed-up is achieved by taking advantage of the advanced rendering method that allows skipping empty space [S1].
*   **Strengths:** High compression ratios, fast encoding for P-frames due to codebook reuse and partial searches, and good visual quality when key-frames are selected appropriately [S1].
*   **Limitations:** Codebook drift can occur if too many P-frames are used between key-frames, leading to sub-optimal quantization. The method requires careful selection of key-frame intervals [S1].

## Background
The method was proposed in the context of compression domain volume rendering, specifically for hardware-accelerated texture decoding [S1]. It builds on hierarchical quantization schemes to enable efficient rendering of time-varying volumetric data [S1].

## Sources
[S1] Compression Domain Volume Rendering, https://www.cs.cit.tum.de/fileadmin/w00cfj/cg/Research/Publications/2003/Compression_Domain_Volume_Rendering/vis03-vq.pdf, fetched 2026-10-26
