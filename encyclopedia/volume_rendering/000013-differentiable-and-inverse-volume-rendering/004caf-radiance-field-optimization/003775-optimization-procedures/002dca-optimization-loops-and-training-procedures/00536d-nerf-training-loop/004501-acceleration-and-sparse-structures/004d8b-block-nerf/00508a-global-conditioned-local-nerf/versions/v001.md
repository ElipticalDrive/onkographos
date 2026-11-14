# Global Conditioned Local NeRF

Also called: Local Model with Shared Decoder

## Overview
Global Conditioned Local NeRF (GCL-NeRF) scales neural radiance fields to city-scale scenes by training separate local NeRF models for spatial blocks while sharing and fine-tuning a decoder initialized from a coarse global model [S1, unofficial]. It differs from Block-NeRF by replacing simple image blending with a shared decoder that aligns feature spaces across blocks, mitigating the foggy artifacts caused by inverse distance weighting [S1, unofficial].

## How it works
**Inputs:** A set of training images with camera poses $P = \{P_1, \dots, P_N\}$ and a scene bounding box.
**Outputs:** A set of local NeRF models (one per block) and a fused final image.

**Data Structures:**
*   **Hash Table:** A multi-resolution hash table with $L=16$ levels. Each level has dimensions of 4. The total number of hash entries per level is $T = 2^{19}$ [S1, unofficial].
*   **Hash Function:** Maps a 3D point $x$ to a hash entry index:
    $$ h(x) = \bigoplus_{i=1}^{d} (x_i \cdot \pi_i) \mod T $$
    where $\oplus$ is bitwise XOR, $d$ is the dimensionality (3), and $\pi_i$ are large unique prime numbers [S1, unofficial].
*   **Decoder:** A multi-layer perceptron (MLP) with depth 5 and width 128 [S1, unofficial].
*   **Occupancy Grid:** Used to skip empty space during sampling [S1, unofficial].

**Steps:**
1.  **Coarse Global Reconstruction:** Train a single NeRF model on the entire scene using all training images. This model uses the same hash table size $T$ as the local models. This step provides the initial weights for the shared decoder and establishes a global prior [S1, unofficial].
2.  **Block Splitting:** Partition the scene into overlapping blocks. The bounding boxes of local blocks are scaled up to create overlapping regions [S1, unofficial].
3.  **Local Model Initialization:** For each block, initialize a local NeRF model. The local model consists of:
    *   A local hash table (initialized with the coarse global hash table weights).
    *   A copy of the decoder (initialized with the coarse global decoder weights) [S1, unofficial].
4.  **Local Model Training:** Train each local model independently using only images associated with that block. The local hash encoder is trained to adapt to the specific block, while the decoder is fine-tuned to better condition the hash encodings [S1, unofficial].
5.  **Image Blending:** Render images from each block. Fuse them using a 'winner-takes-all' strategy based on the proximity of the camera to the block center, avoiding complex weighting schemes [S1, unofficial].

**Parameters:**
*   **Optimizer:** Adam [S1, unofficial].
*   **Iterations:** 200,000 [S1, unofficial].
*   **Learning Rate (LR):**
    *   Fine-tuning shared decoder: $5 \times 10^{-5}$ [S1, unofficial].
    *   Other methods (baseline): $1 \times 10^{-3}$ [S1, unofficial].
*   **Warm-up:** 5,000 iterations with linear LR increase [S1, unofficial].
*   **LR Decay:** Exponential decay with rate 0.1 after warm-up [S1, unofficial].

**Pitfalls:**
*   **Hash Collisions:** The coarse global model suffers from more hash collisions than local models because the same hash table size $T$ is used for a much larger volume of points. Collided entries take a weighted mean of gradients from different scene parts, potentially degrading reconstruction quality [S1, unofficial].
*   **Memory Constraints:** The fine branch of the original vanilla NeRF cannot be enabled due to memory limits (e.g., 24GB GPU) and increased training time [S1, unofficial].

## Subtopics

### Coarse Global Reconstruction
A single NeRF model trained on the entire scene using the same hash table size as local models. It provides global guidance for image blending and initializes the shared decoder. Unlike local models, it suffers from higher hash collision rates due to the larger volume of points mapped to the same table size $T$ [S1, unofficial].

## Operating characteristics
*   **Hardware:** Tested on NVIDIA A5000 GPU with 24GB memory [S1, unofficial].
*   **Memory:** The method reduces memory footprint by using local hash tables and skipping empty space via an occupancy grid sampler [S1, unofficial].
*   **Performance:** The method is designed for large-scale unbounded scenes (e.g., Tanks-and-Temple datasets) [S1, unofficial].
*   **Limitations:** The coarse global model's quality is limited by hash collisions. The fine branch of NeRF is disabled, potentially limiting detail recovery in complex areas [S1, unofficial].

## Background
GCL-NeRF (SCALAR-NeRF) was introduced to address the scalability issues of NeRF in large environments. It builds on Block-NeRF by introducing a shared decoder mechanism to improve consistency across block boundaries [S1, unofficial].

## Sources
[S1] SCALAR-NeRF: SCAlable LARge-scale Neural Radiance Fields for Scene Reconstruction, https://arxiv.org/abs/2311.16657v1, fetched 2026-10-26 (unofficial)
