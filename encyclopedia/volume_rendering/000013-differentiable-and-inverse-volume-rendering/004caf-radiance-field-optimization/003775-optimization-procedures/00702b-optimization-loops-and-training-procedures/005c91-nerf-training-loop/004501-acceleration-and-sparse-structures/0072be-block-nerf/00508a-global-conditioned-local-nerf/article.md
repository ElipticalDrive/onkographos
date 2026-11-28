# Global Conditioned Local NeRF

Also called: Local Model with Shared Decoder

## Overview
Global Conditioned Local NeRF (GCL-NeRF) scales neural radiance fields to large scenes by training separate local NeRF models for spatial blocks while sharing and fine-tuning a decoder initialized from a coarse global model [S1, unofficial]. It differs from [Block-NeRF](13537-block-nerf.md) by replacing inverse distance weighting with a shared decoder that aligns feature spaces across blocks, mitigating foggy blending artifacts [S1, unofficial].

## How it works
**Inputs:** A set of training images with camera poses $P = \{P_1, \dots, P_N\}$ and a scene bounding box [S1, unofficial].
**Outputs:** A set of local NeRF models (one per block) and a fused final image [S1, unofficial].

**Data Structures:**
*   **Hash Table:** A multi-resolution hash table with $L=16$ levels, each with dimensions of 4, and total entries $T = 2^{19}$ per level [S1, unofficial].
*   **Decoder:** A multi-layer perceptron (MLP) with depth 5 and width 128 [S1, unofficial].
*   **Occupancy Grid:** A spatial structure used to skip empty space during sampling [S1, unofficial].

**Steps:**
1.  **Coarse Global Reconstruction:** Train a single NeRF model on the entire scene using all training images [S1, unofficial].
2.  **Block Splitting:** Partition the scene into overlapping blocks by scaling up the bounding boxes that cover all images in local blocks [S1, unofficial].
3.  **Local Model Initialization:** For each block, initialize a local NeRF model consisting of a local hash table (initialized with coarse global weights) and a copy of the decoder (initialized with coarse global weights) [S1, unofficial].
4.  **Local Model Training:** Train each local model independently using only images associated with that block [S1, unofficial]. The local hash encoder adapts to the specific block, while the decoder is fine-tuned to better condition the hash encodings [S1, unofficial].
5.  **Image Blending:** Render images from each block and fuse them using a 'winner-takes-all' strategy based on camera proximity to block centers [S1, unofficial].

**Formulas:**
*   **Hash Function:** Maps a 3D point $x$ to a hash entry index:
    $$h(x) = \bigoplus_{i=1}^{d} x_i \pi_i \mod T$$
    where $\oplus$ denotes bitwise XOR, $d$ is the dimensionality (3), $\pi_i$ are large unique prime numbers, and $T$ is the total number of hash entries per level [S1, unofficial].

**Parameters:**
*   **Optimizer:** Adam [S1, unofficial].
*   **Iterations:** 200,000 [S1, unofficial].
*   **Learning Rate:** Fine-tuning the shared decoder uses $5 \times 10^{-5}$; baseline methods use $1 \times 10^{-3}$ [S1, unofficial].
*   **Warm-up:** 5,000 iterations with linear LR increase [S1, unofficial].
*   **LR Decay:** Exponential decay with rate 0.1 after warm-up [S1, unofficial].

**Pitfalls:**
*   **Hash Collisions:** The coarse global model suffers from more hash collisions than local models because the same hash table size $T$ is used for a much larger volume of points [S1, unofficial]. Collided entries take a weighted mean of gradients from different scene parts, potentially degrading reconstruction quality [S1, unofficial].
*   **Memory Constraints:** The fine branch of the original vanilla NeRF cannot be enabled due to memory limits (e.g., 24GB GPU) and increased training time [S1, unofficial].

## Subtopics

### Coarse Global Reconstruction
A single NeRF model trained on the entire scene using the same hash table size as local models [S1, unofficial]. It provides global guidance for image blending and initializes the shared decoder [S1, unofficial]. Unlike local models, it suffers from higher hash collision rates due to the larger volume of points mapped to the same table size $T$ [S1, unofficial].

## Operating characteristics
*   **Complexity:** Time and memory complexity are not given in the sources.
*   **Hardware:** Tested on NVIDIA A5000 GPU with 24GB memory [S1, unofficial].
*   **Memory:** Reduces footprint by using local hash tables and skipping empty space via an occupancy grid sampler [S1, unofficial].
*   **Performance:** Designed for large-scale unbounded scenes (e.g., Tanks-and-Temple datasets) [S1, unofficial].
*   **Strengths:** Aligns feature spaces across blocks via a shared decoder, avoiding foggy artifacts from simple blending [S1, unofficial].
*   **Limitations:** Coarse global model quality is limited by hash collisions; the fine branch of NeRF is disabled, potentially limiting detail recovery [S1, unofficial].
*   **Usage:** Suitable for city-scale or large unbounded scene reconstruction where memory limits prevent training a single dense global model [S1, unofficial].

## Background
GCL-NeRF (SCALAR-NeRF) was introduced to address the scalability issues of NeRF in large environments [S1, unofficial]. It builds on [Block-NeRF](13537-block-nerf.md) by introducing a shared decoder mechanism to improve consistency across block boundaries [S1, unofficial]. The method is used for large-scale unbounded scene reconstruction and relies on hash-based encoding with occupancy grid sampling for efficiency [S1, unofficial].

## Sources
[S1] SCALAR-NeRF: SCAlable LARge-scale Neural Radiance Fields for Scene Reconstruction, https://arxiv.org/abs/2311.16657v1, fetched 2026-10-26 (unofficial)
