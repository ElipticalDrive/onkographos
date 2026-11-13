# Bitonic Sort

Also called: ABisort, Bitonic Sort for GPGPU, Govindaraju et al. Bitonic Sort, Particle Sorting (Bitonic Sort)

## Overview
Bitonic Sort is a data-independent network sorting algorithm implemented on the GPU to provide exact depth ordering of splatted voxels for every frame [S1, unofficial]. Unlike Odd-Even Merge Sort, which offers only partial ordering suitable for particle systems spread across frames, Bitonic Sort guarantees a fully sorted state required for correct alpha compositing in splatting [S1, unofficial]. It is chosen over Adaptive Bitonic Sort (ABisort) in some implementations due to code accessibility and ease of adaptation, despite ABisort having potentially better theoretical complexity [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** An array of `<key, value>` pairs residing in GPU memory (e.g., View Transformed Voxels Texture). The `key` is the depth value $b$ of the voxel, and the `value` is the voxel index [S1, unofficial].
*   **Output:** A Depth-Sorted Voxels Texture containing the voxels ordered by depth [S1, unofficial].

**Data Structures:**
The algorithm uses the Multiple Render Target (MRT) extension to store keys and values in separate textures with a one-to-one correspondence [S1, unofficial]. This allows packing four `<key, value>` instances into a single RGBA texel [S1, unofficial]. A ping-pong strategy is used between two render targets to perform the sorting steps [S1, unofficial].

**Parameters:**
*   $N$: Number of elements to sort.
*   $p$: Number of stream processor units (relevant for ABisort complexity analysis) [S1, unofficial].
*   $uPass$: Current pass number in the sort algorithm [S2, unofficial].
*   $uStage$: Current stage number in the sort algorithm [S2, unofficial].
*   $uHalfVector$: The half-angle vector used for sorting particles [S2, unofficial].

**Algorithm Steps:**
1.  **Initialization:** If the input is on the CPU, it is transferred to the GPU. In the splatting context, view-transformed voxels are already in GPU memory [S1, unofficial]. A fragment shader fills the initial render targets with the depth value $b$ as the key and the voxel index as the value [S1, unofficial].
2.  **Sorting Passes:** The algorithm executes a series of passes and stages. For each element $i$, a partner address $adr$ is calculated based on the current pass and stage [S2, unofficial].
3.  **Comparison:** The element at $i$ and its partner at $adr$ are fetched. The comparison direction is determined by the pass, stage, and index [S2, unofficial].
4.  **Swap Logic:** The elements are sorted based on their projected lengths relative to the half vector [S2, unofficial]. If `selfProjectedLength * compare < partnerProjectedLength * compare`, the self element is kept; otherwise, the partner is kept [S2, unofficial].
5.  **Ping-Pong:** Results are written to the alternate render target.
6.  **Finalization:** After sorting is complete, a fragment shader fills the Depth-Sorted Voxels Texture from the sorted render targets [S1, unofficial].

**Pseudocode:**
The following pseudocode represents the fragment shader logic for a single comparison step within the Bitonic Sort network [S2, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Inputs:
# uHalfVector: vec3, the half-angle vector
# uPass: int, current pass number
# uStage: int, current stage number
# Texture2D prevPositions: texture containing <key, value> pairs from previous step

# Output:
# vec4 result: new <key, value> pair

vec3 halfVec = uHalfVector;
int pass = uPass;
int stage = uStage;

# Calculate particle index i from texture coordinates
vec2 texCoord = gl_FragCoord.xy / vec2(textureSize(prevPositions, 0));
int i = int(texCoord.x * textureSize(prevPositions, 0).x);

# Determine comparison direction
# Note: The exact logic for 'compare' depends on the specific Bitonic Sort stage/pass configuration
# Typically involves checking if i is in the first or second half of the current block
int compare = 1; # Placeholder for direction logic derived from pass/stage/i
if ((i & (1 << stage)) != 0) {
    compare = -1;
}

# Calculate partner address
int adr = i + compare * (1 << stage);

# Fetch partner data
# Assuming 4 elements per texel, we must calculate the texel coordinate and component
int texelIdx = adr / 4;
int component = adr % 4;
vec4 partnerTexel = texture(prevPositions, vec2(adr / textureSize(prevPositions, 0).x, texCoord.y));
vec3 partnerKey = partnerTexel[component]; # Key is depth b

# Fetch self data
int selfTexelIdx = i / 4;
int selfComponent = i % 4;
vec4 selfTexel = texture(prevPositions, vec2(i / textureSize(prevPositions, 0).x, texCoord.y));
vec3 selfKey = selfTexel[selfComponent];

# Calculate projected lengths
# Note: The source mentions "selfProjectedLength" and "partnerProjectedLength"
# This likely refers to the dot product of the position/vector with the half vector
vec3 selfPos = ...; # Derived from voxel index i
vec3 partnerPos = ...; # Derived from voxel index adr
float selfProjLen = dot(selfPos, halfVec);
float partnerProjLen = dot(partnerPos, halfVec);

# Sort based on dot product with half vector
# Output self if selfProjectedLength * compare < partnerProjectedLength * compare, else partner
vec3 newKey;
if (selfProjLen * compare < partnerProjLen * compare) {
    newKey = selfKey;
} else {
    newKey = partnerKey;
}

# The value (voxel index) must also be swapped accordingly
int newValue;
if (selfProjLen * compare < partnerProjLen * compare) {
    newValue = i;
} else {
    newValue = adr;
}

# Pack result into RGBA texel
# Note: This is a simplified representation. In reality, the shader writes to MRTs
# and the ping-pong logic handles the texture swapping.
result = vec4(newKey, 1.0); 
```

**Edge Cases and Pitfalls:**
*   **Data-Driven vs. Network Sort:** Quicksort is unsuitable for GPU due to its data-dependent branching, which performs poorly on parallel architectures [S1, unofficial].
*   **Exact vs. Partial Ordering:** Odd-Even Merge Sort provides partial ordering, which is insufficient for frame-by-frame splatting compositing where exact depth order is required [S1, unofficial].
*   **Performance:** Govindaraju et al.'s implementation is approximately twice as fast as the best CPU quicksort on an NVIDIA GeForce 7800 [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \log^2 N)$ for the standard Bitonic Sort network [S1, unofficial]. Adaptive Bitonic Sort (ABisort) achieves $O((N \log N) / p)$ where $p$ is the number of stream processors [S1, unofficial].
*   **Memory Complexity:** Requires storage for keys and values, typically using MRTs. Four elements can be packed into one RGBA texel [S1, unofficial].
*   **Performance:** On an NVIDIA GeForce 7800, Govindaraju's Bitonic Sort is about twice as fast as the best CPU quicksort implementation [S1, unofficial].
*   **Strengths:** Provides exact ordering at each frame, which is critical for correct alpha blending in splatting [S1, unofficial]. Highly parallelizable on GPU architectures [S1, unofficial].
*   **Limitations:** Data-independent nature means it performs the same number of comparisons regardless of input order, which can be less efficient than adaptive sorts for partially sorted data [S1, unofficial].
*   **When to Use:** When exact depth ordering is required for every frame in a splatting pipeline [S1, unofficial].
*   **When Not to Use:** When partial ordering is sufficient (e.g., some particle systems) or when adaptive sorting performance is critical and code complexity is acceptable [S1, unofficial].

## Background
Bitonic Sort was originally developed as a network sorting algorithm. In the context of GPU volume rendering, Govindaraju et al. [GRHM05] improved early implementations [KW05] by using cache-efficient memory accesses [S1, unofficial]. The method was adapted to sort very large billion-record databases [GGKM06]. Gress and Zachmann [GZ06] later designed Adaptive Bitonic Sort (ABisort), which has optimal complexity [S1, unofficial]. In the specific implementation described in [S1, unofficial], Govindaraju's method was chosen over ABisort because the library was fully accessible and easy to adapt, despite ABisort being faster [S1, unofficial].

## Sources
[S1] Image-space sheet-buffered splatting on the GPU, https://openalex.org/W2419486389, fetched 2026-10-27 (unofficial)
[S2] GitHub - zk8085454/VolumeParticleShadows: Half angle slicing technique ..., https://github.com/zk8085454/VolumeParticleShadows/, fetched 2026-10-28 (unofficial)
