# Similarity-based Classification


## Overview
Similarity-based classification methods determine optical properties by measuring the distance or correlation between a sample point's feature vector and a set of reference exemplars or annotations, rather than relying solely on raw intensity or simple gradient magnitudes [S1][S3, S4, unofficial]. This family is distinct from its siblings because it operates in a high-dimensional feature space (range-space) to capture semantic or structural similarity, enabling the segmentation of complex, multidimensional, or spatiotemporal data where traditional scalar thresholds fail [S1][S3, S4, unofficial]. Unlike [Curvature-based Transfer Function](5452-curvature-based-transfer-function.md) which focuses on local geometric shape, or [Machine Learning and Feature Extraction](12097-machine-learning-and-feature-extraction.md) which may use supervised labels, similarity-based methods often rely on user-provided exemplars or self-supervised proximity tasks to define classification boundaries [S1][S4, unofficial].

## Shared principles
All methods in this family share the core mechanism of evaluating a classification function $C(\mathbf{x})$ for each sample point $\mathbf{x}$ in the volume to produce optical properties [G].

**Input Domain:**
The input to the classification function is typically a feature vector $\mathbf{f}(\mathbf{x})$ associated with position $\mathbf{x}$, which may consist of raw intensity, gradient magnitude, or high-dimensional embeddings (e.g., from a Vision Transformer) [S1][S3, S4, unofficial]. The classification relies on comparing $\mathbf{f}(\mathbf{x})$ against a reference set $M = \{\mathbf{m}_1, \dots, \mathbf{m}_m\}$ of exemplar vectors or annotated samples [S1][S3, S4, unofficial].

**Similarity Metric:**
A distance or similarity metric $d(\mathbf{u}, \mathbf{v})$ is defined over the feature space. Common metrics include:
*   **Euclidean Distance:** $\|\mathbf{u} - \mathbf{v}\|_2$, often weighted to emphasize specific feature dimensions [S1].
*   **Cosine Similarity:** $\frac{\mathbf{u} \cdot \mathbf{v}}{\|\mathbf{u}\|_2 \|\mathbf{v}\|_2}$, used to measure angular similarity independent of magnitude [S1, S2].
*   **Spatiotemporal Proximity:** In spatiotemporal data, similarity may incorporate both feature distance and spatial-temporal coordinates [S4, unofficial].

**Classification Output:**
The output is a classification field $L(\mathbf{x})$ assigning each point to a class (e.g., the index of the nearest exemplar) or a continuous similarity score $S(\mathbf{x})$ [S1][S3, S4, unofficial]. This field drives the transfer function, mapping the similarity score or class label to opacity $\alpha$ and color $C$ [S1][S3, unofficial].

**Evaluation Order:**
Similarity computation is typically performed as a preprocessing step or during interactive annotation, separate from the final ray-casting accumulation, allowing for high-dimensional feature analysis before rendering [S1][S3, unofficial].

## Subtopics

### [Range-Space Segmentation](13745-range-space-segmentation.md)
Range-Space Segmentation projects high-dimensional functions at each spatial location into scalar fields based on their distance to user-defined exemplar functions. It is chosen when visualizing function field data (e.g., hyperspectral or time-varying data) where preserving spatial structure while segmenting based on functional similarity is required. Unlike [High-Dimensional Classification with Color Data](6327-high-dimensional-classification-with-color-data.md) which may use complex color spaces, this method explicitly constructs Voronoi-like tessellations in the range-space of the functions.

### Cosine Similarity Map Computation
This variant computes a similarity volume $S_L$ by comparing annotated feature vectors against the full feature volume $F$ using cosine similarity, enabling interactive feedback during transfer function design [S1]. It is used when users need immediate visual feedback on how annotations propagate through semantically similar regions (e.g., identifying all kidneys in a CT scan) [S1].
*   **Formula:** The base similarity map $S_L^C$ for class $C$ is computed as:
    $$ S_L^C = \max_{a \in A_C} \frac{1}{|F|} \sum_{i \in F} \frac{F_a \cdot F_i}{\|F_a\|_2 \|F_i\|_2} $$
    where $A_C$ is the set of annotations for class $C$, $F_a$ is the feature vector at annotation $a$, and $F_i$ is the feature vector at voxel $i$ [S1].
*   **Proximity Scaling:** To select spatially local structures, the similarity map is scaled by a proximity parameter $\tilde{p} \in [0, 1]$:
    $$ P(x) = \max_{a \in A_C} e^{-10\tilde{p}|x-a|} $$
    where $x$ is the location in $\mathbb{R}^3$ and $a$ is an annotation location [S1].
*   **Performance:** Computation takes a few milliseconds on CPU or GPU, allowing for real-time interaction [S1].

### Thresholded Cosine Similarity Segmentation
This method computes the probability of a query by using thresholded cosine similarity instead of softmax with negative queries, suitable for hard decomposition in feature field distillation [S2]. It is chosen when a binary or hard segmentation is required based on feature similarity thresholds.
*   **Step:** Set $p = 1$ if the cosine similarity exceeds a user-defined threshold, and $p = 0$ otherwise [S2].

### Discrete Classification via Voxel Similarity
A discrete voxel classification technique for multidimensional signatures where users perform merge or bipartition operations on existing classes [S3, unofficial]. It is chosen for interactive volume visualization where users need to rapidly refine classes based on voxel similarity criteria.
*   **Step 1:** Preprocessing automatically establishes an initial classification respecting voxel similarity criteria.
*   **Step 2:** The user performs merge or bipartition operations on classes.
*   **Step 3:** The user defines visual properties for each class.
*   **Step 4:** Bipartition operations are executed rapidly, respecting the initial similarity criteria [S3, unofficial].

### S4
S4 is a self-supervised approach to spatiotemporal similarity learning using a Siamese neural network trained on a binary pretext task to predict spatial-temporal proximity, eliminating the need for labeled data [S4, unofficial]. It is chosen for exploring large spatiotemporal ensembles where manual annotation is infeasible.
*   **Step 1:** Train a Siamese neural network on a binary pretext task: predict if two spatiotemporal patches originate from nearby locations in space and time.
*   **Step 2:** Use the resulting latent space for similarity queries.
*   **Step 3:** Users supply example patches as a query; S4 ranks the rest of the ensemble by learned similarity [S4, unofficial].

## Background
Similarity-based classification emerged from the need to handle multidimensional and high-dimensional volume data where scalar intensity is insufficient for meaningful segmentation. Early work by Tzeng and Ma introduced discrete classification via voxel similarity for multidimensional signatures, allowing interactive merging and bipartitioning of classes based on feature proximity [S3, unofficial]. More recently, the integration of self-supervised vision transformers (e.g., DINO) has enabled semantic similarity mapping, where cosine similarity in feature space replaces hand-crafted transfer functions [S1]. Concurrently, self-supervised spatiotemporal learning (S4) has extended similarity concepts to time-varying scientific data, using proximity as a label-free training signal [S4, unofficial]. These methods are used in medical imaging (e.g., organ segmentation), hyperspectral remote sensing, and scientific visualization of complex ensembles.

## Sources
[S1] Leveraging Self-Supervised Vision Transformers for Segmentation-Based Transfer Function Design., https://doi.org/10.1109/tvcg.2024.3401755, fetched 2026-10-26
[S2] Decomposing NeRF for Editing Via Feature Field Distillation, https://doi.org/10.52202/068431-1694, fetched 2026-10-26
[S3] Especificação de funções de transferência unidimensionais e multidimensionais para visualização volumétrica direta, http://hdl.handle.net/10183/12077, fetched 2026-10-30 (unofficial)
[S4] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
