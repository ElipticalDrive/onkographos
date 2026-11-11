# HAC++
Also called: Anchor-based compression for 3DGS, HAC, Hash-based Anchor Compression

## Overview
HAC++ is an anchor-based compression method for 3D Gaussian Splatting that couples unorganized Scaffold-GS anchors with a binarized structured hash grid to establish a context model for arithmetic entropy coding [S1, unofficial][S3, unofficial]. It achieves approximately 100× size reduction over vanilla 3DGS by leveraging mutual information between anchors and the hash grid alongside intra-anchor context [S3, unofficial]. Unlike sibling methods that rely on fixed spatial structures or simple quantization, HAC++ explicitly models hierarchical relationships among anchors to enhance representational capacity [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Unorganized 3D Gaussian anchors and a multi-resolution hash grid structure [S2][S3, unofficial].
*   **Output:** A compressed bitstream of quantized anchor features and hash grid states encoded via arithmetic coding [S1, unofficial][S3, unofficial].

**Data Structures:**
1.  **Anchor Set:** Collection of $N$ unorganized Gaussian primitives [S3, unofficial].
2.  **Binarized Structured Hash Grid:** Multi-resolution grid storing binary states to provide spatial priors [S1, unofficial][S3, unofficial].
3.  **Context Model:** Probabilistic model estimating attribute distributions conditioned on hash grid state and intra-anchor context [S3, unofficial].

**Steps:**
1.  Initialize unorganized Scaffold-GS anchors [S3, unofficial].
2.  Construct a multi-resolution binarized structured hash grid to provide spatial priors [S1, unofficial][S3, unofficial].
3.  Apply adaptive quantization by estimating per-attribute distributions with Gaussian models [S3, unofficial].
4.  Build a context model using mutual information between anchors and the hash grid, augmented by intra-anchor context [S3, unofficial].
5.  Encode quantized features and grid states via arithmetic entropy coding [S1, unofficial][S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hac_compress(anchors, grid_resolutions):
    # Step 1: Initialize unorganized Scaffold-GS anchors
    # Not given in the sources.
    anchor_set = anchors

    # Step 2: Construct binarized structured hash grid
    # Not given in the sources.
    hash_grid = {}

    # Step 3: Adaptive Quantization
    # Estimates per-attribute distributions with Gaussian models [S3, unofficial]
    quantized_attrs = []
    for anchor in anchor_set:
        q_attr = {}
        for attr in anchor.attributes:
            # Estimate Gaussian distribution parameters [G]
            mu = sum(attr) / len(attr)
            sigma = (sum((x - mu)**2 for x in attr) / len(attr))**0.5
            # Quantize attribute based on estimated distribution [G]
            q_attr[attr] = round((attr - mu) / sigma)
        quantized_attrs.append(q_attr)

    # Step 4: Context Modeling
    # Uses mutual information between anchors and hash grid plus intra-anchor context [S3, unofficial]
    # Not given in the sources.
    context_model = {}

    # Step 5: Arithmetic Entropy Coding
    # Encodes quantized features and grid states [S1, unofficial][S3, unofficial]
    # Not given in the sources.
    bitstream = []
    for attr in quantized_attrs:
        bitstream.extend(arithmetic_encode_symbol(attr, context_model))

    return bitstream
```

**Edge Cases and Pitfalls:**
*   Not given in the sources.

## Subtopics

### Scaffold-GS
Organizes Gaussians into a structured scaffold [S3, unofficial]. Differs from HAC++ by explicitly structuring anchors, whereas HAC++ treats them as unorganized but uses a hash grid for context [S3, unofficial]. Choose when structured initialization is beneficial. [Link](5-sparse-structures.md)

### Arithmetic Encoding
Entropy coding technique used by HAC++ to approach the Shannon entropy limit [S1, unofficial][S3, unofficial]. Differs from Huffman coding by allowing fractional bit lengths [G]. Choose when high compression ratios are critical. [Link](2127-arithmetic-encoding.md)

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** ~100× size reduction over vanilla 3DGS [S3, unofficial].
*   **Strengths:** High compression ratio, sophisticated context modeling, adaptive quantization [S1, unofficial][S3, unofficial].
*   **Limitations:** Not given in the sources.
*   **When to Use:** Not given in the sources.
*   **When Not to Use:** Not given in the sources.

## Background
HAC++ builds on Context-GS and CompGS, which adopt context-aware approaches to model hierarchical relationships among anchors [S1, unofficial]. HAC++ further improves this by incorporating an interpolated hash grid to provide spatial priors for unorganized anchors, establishing a more sophisticated context model for arithmetic entropy coding [S1, unofficial]. It represents an advancement in anchor-based compression for 3D Gaussian Splatting [S3, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S3] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
