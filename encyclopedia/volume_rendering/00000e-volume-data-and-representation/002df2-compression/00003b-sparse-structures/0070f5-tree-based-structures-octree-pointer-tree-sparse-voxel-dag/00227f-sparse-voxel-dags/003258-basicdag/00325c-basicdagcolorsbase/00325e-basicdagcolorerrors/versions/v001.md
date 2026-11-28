# BasicDAGColorErrors


## Overview
BasicDAGColorErrors is a leaf-level quality control mechanism that computes a binary error flag by comparing compressed and uncompressed color representations [S1, unofficial]. It serves as a binary validator for adaptive compression, ensuring that the color fidelity remains within a strict tolerance threshold before a leaf node is accepted or processed further. It distinguishes itself from sibling methods by returning a hard binary decision (1.0 or 0.0) based on a fixed error threshold, rather than providing a continuous error metric or performing the compression itself.

## Shared principles
This family relies on the principle of **lossy color compression verification**. The core mechanism involves retrieving both the original (uncompressed) color data and the reconstructed (compressed) color data for a specific leaf node, calculating a scalar error metric between them, and thresholding that metric to produce a binary validity flag.

The mathematical foundation is the color error computation:
$$ E = \text{ColorUtils::color_error}(C_{original}, C_{compressed}) $$

Where $C_{original}$ is the uncompressed color and $C_{compressed}$ is the color retrieved from the compressed representation. The error $E$ is compared against a fixed threshold $\tau$.

## Subtopics
This class is a specific implementation detail within the `BasicDAGColorsBase` hierarchy. It does not have further child classes in the provided tree structure. It is chosen when strict binary validation of color compression fidelity is required for adaptive rendering or streaming, specifically when the error must be quantized to a pass/fail state rather than a continuous value.

## Background
The method originates from the HashDAG project by Phyronnaz [S1, unofficial]. It is part of the optimization pipeline for Sparse Voxel DAGs, specifically addressing the need to verify the integrity of compressed color data stored in the DAG structure. The implementation is tied to the `BasicDAGColorsBase` architecture, which decouples leaf count storage from the main geometry array.

## Sources
[S1] Phyronnaz/HashDAG, https://github.com/Phyronnaz/HashDAG, fetched 2026-10-26 (unofficial)
