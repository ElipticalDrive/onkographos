# Heightmap Combination & Seamless Terrain Generation

Also called: Map Generation from Heightmaps, Seamless Terrain Generation via Heightmap Stitching

## Overview
This method generates macroscopic terrain by stitching together multiple pre-existing heightmaps into a single seamless composite, using simplex noise to guide the selection and blending of regions [S1, unofficial]. It differs from pure procedural noise generation by leveraging real-world or high-fidelity source data, optimizing memory bandwidth by packing multiple heightmaps into a single RGB/RGBA texture for simultaneous retrieval [S1, unofficial].

## How it works
**Inputs:**
*   $N$ source heightmaps (typically $N=6$ in the reference implementation, each of significant resolution) [S1, unofficial].
*   A simplex noise function $S(x, y, z)$ [S1, unofficial].
*   Target water coverage percentage $P_{water}$ (e.g., 0.5 for 50%) [S1, unofficial].

**Outputs:**
*   A final terrain heightmap $H_{final}(x, y)$ at the desired resolution.
*   A determined sea level threshold $H_{sea}$ [S1, unofficial].

**Data Structures:**
*   **Combined Texture:** A single RGB or RGBA image where each channel stores one of the source heightmaps. For $N=3$, $R$ stores $H_1$, $G$ stores $H_2$, $B$ stores $H_3$. For $N=4$, $A$ stores $H_4$. This allows a single texture fetch to retrieve $N$ height values [S1, unofficial].
*   **Height Histogram:** An array or map counting pixel occurrences per height value, used to derive the sea level [S1, unofficial].

**Preprocessing:**
1.  **Seamless Stitching:** The source heightmaps must be made seamless individually or as a group in an image editor (e.g., Photoshop) to prevent visible artifacts at the boundaries of the source tiles [S1, unofficial]. This involves ensuring that the edges of the heightmap tiles match their opposite edges (periodic boundary conditions) or blending them smoothly [S1, unofficial].

**Algorithm Steps:**
1.  **Macroscopic Terrain Generation:**
    For each output pixel $(x, y)$:
    a.  Compute a simplex noise value $n = S(x, y, z_{offset})$ [S1, unofficial].
    b.  Use $n$ to select a region from the combined texture. The method describes sampling "random areas" and "stitching them together based on simplex noise," analogous to a clone stamp tool [S1, unofficial].
    c.  *Implementation Note:* The exact interpolation or selection logic is not specified in the source. A standard implementation would use $n$ to determine a weight or index into the available heightmap channels within the combined texture [G].
    d.  Retrieve the height value $h$ from the selected channel of the combined texture at the corresponding UV coordinate [S1, unofficial].
    e.  Assign $H_{macro}(x, y) = h$ [S1, unofficial].

2.  **Recursive Detail Generation:**
    a.  Recursively sample from $H_{macro}$ to generate finer details down to the voxel level [S1, unofficial].
    b.  *Implementation Note:* The specific recursive function or detail scale factor is not provided [G]. This step likely involves multi-octave noise addition or iterative refinement of the heightmap [G].

3.  **Sea Level Determination:**
    a.  Generate a histogram of heights from $H_{macro}$ (or the final terrain), counting the number of pixels $C(h)$ for each height value $h$ [S1, unofficial].
    b.  Compute the cumulative distribution function (CDF): $CDF(h) = \sum_{h' \le h} C(h')$ [S1, unofficial].
    c.  Normalize the CDF by the total number of pixels $T$: $CDF_{norm}(h) = CDF(h) / T$ [S1, unofficial].
    d.  Find the height $H_{sea}$ such that $CDF_{norm}(H_{sea}) \approx P_{water}$ [S1, unofficial]. This is done by looking "halfway down the line" for a 50% target [S1, unofficial].

**Parameters:**
*   Number of source heightmaps: 6 (reference) [S1, unofficial].
*   Target water coverage: Arbitrary (e.g., 50%) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Non-seamless sources:** If source heightmaps are not made seamless, visible seams will appear in the final terrain [S1, unofficial].
*   **Precision:** The method is described as "physically correct? No," indicating it is a visual approximation [S1, unofficial].

## Subtopics

### Sea Level Determination via Height Histogram
This variant focuses specifically on the statistical method for setting sea level. It calculates the sea level threshold to achieve a specific water coverage percentage by analyzing the height distribution of the generated terrain. It involves generating a histogram of terrain heights, computing the cumulative distribution, and selecting the height value corresponding to the target percentage (e.g., 50% water coverage) [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Dominated by the simplex noise evaluation and texture sampling for each pixel. Recursive detail generation adds further cost proportional to the depth of recursion [S1, unofficial].
*   **Memory Complexity:** Optimized by packing $N$ heightmaps into a single texture, reducing texture fetch overhead. The reference uses 6 large heightmaps, but only 1-2 texture reads are needed per pixel depending on packing [S1, unofficial].
*   **Strengths:** Allows for the use of high-quality, real-world heightmap data; efficient texture sampling; controllable water coverage [S1, unofficial].
*   **Limitations:** Requires pre-processing of source heightmaps to be seamless; not physically based; exact stitching algorithm is not detailed, potentially leading to artifacts if not implemented carefully [S1, unofficial].
*   **When to use:** When high-fidelity terrain features from existing data are desired, and performance optimization via texture packing is needed [S1, unofficial].

## Background
The method was described by the creator of VoxelQuest in a blog post detailing the game's map and world generation pipeline [S1, unofficial]. It is part of a procedural generation system that also includes city placement and road generation [S1, unofficial].

## Sources
[S1] How Does It Work?, https://www.voxelquest.com/news/how-does-it-work, fetched 2026-10-29 (unofficial)
