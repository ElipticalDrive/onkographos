# Structural Focus+Context Rendering

Also called: Focus+Context rendering of structured biomedical data, SF+C, SFC rendering, Structural Focus and Context, structural F+C volume rendering

## Overview
Structural Focus+Context (SFC) rendering is a volume visualization technique for multiclassified data that explicitly models and renders the logical relationships between classification criteria using a directed graph [S1, unofficial]. Unlike standard volume rendering which treats classes independently, SFC constructs an auxiliary labelled voxel model and a structural graph to compute transfer functions based on boolean queries over class inclusions [S1, unofficial]. It sets itself apart from sibling methods like [Hybrid Rendering for Stress Tensor Fields](8909-hybrid-rendering-for-stress-tensor-fields.md) by focusing on the semantic structure of multiclassified scalar volumes rather than tensor field geometry, using multimodal 3D texture mapping to simultaneously render context and focus based on cluster labels [S1, unofficial].

## How it works
**Inputs:**
1.  **Original Voxel Model ($V$):** A 3D scalar field, potentially multimodal (e.g., MR/SPECT, MR/PET) [S1, unofficial].
2.  **Classification Criteria:** A set of $n_c$ independent criteria $c_i$ ($i=1 \dots n_c$), each partitioning $V$ into $n_{ci}$ disjoint classes $c_{ij}$ [S2].
3.  **User Query:** A boolean expression defined over the initial classification criteria [S1, unofficial].

**Outputs:**
1.  **Directed Graph ($G$):** Nodes represent classification classes; edges represent logical inclusion relationships [S1, unofficial].
2.  **Labelled Voxel Model ($LV$):** A 3D volume where each voxel stores a unique integer identifier (cluster label) corresponding to the combination of classification criteria it satisfies [S1, unofficial].
3.  **Transfer Functions:**
    *   **CTF (Color/Visibility Transfer Function):** Maps cluster labels to visibility, color, and opacity [S1, unofficial].
    *   **GMTF (Gradient Modulation Transfer Function):** Maps cluster labels to a gradient modulation factor [S1, unofficial].

**Data Structures:**
*   **Directed Graph Model:** A graph $G = (N, E)$ where $N$ is the set of classification criteria (nodes) and $E$ is the set of logical implications (edges) [S1, unofficial].
*   **Auxiliary Voxel Model ($LV$):** A 3D array of integers where $LV[\mathbf{x}] = c$, and $c$ is the unique identifier of the cluster satisfied by $V[\mathbf{x}]$ [S1, unofficial].

**Classification Model:**
For each criterion $c_i$, the isoclasses are disjoint and cover the volume:
$$\forall j, k : 1 \le j, k \le n_{ci}, j \ne k : \text{isoclas}(c_{ij}) \cap \text{isoclas}(c_{ik}) = \emptyset$$
$$\bigcup_{j=1}^{n_{ci}} \text{isoclas}(c_{ij}) = V$$
A class $c_k^i$ implies $c_l^j$ if the isoclass of $c_k^i$ is a subset of the isoclass of $c_l^j$ ($i \ne j$):
$$c_k^i \Rightarrow c_l^j \text{ iff } \text{isoclas}(c_k^i) \subset \text{isoclas}(c_l^j)$$ [S2]

**Preprocessing Steps:**
1.  **Classification:** Classify the initial voxel model $V$ according to various independent criteria, potentially handling multimodal datasets [S1, unofficial].
2.  **Graph Construction:** Identify dependencies between classification criteria and construct a directed graph where nodes are the classification criteria and edges represent logical inclusion relationships [S1, unofficial].
3.  **Clustering:** Clusterize the voxel model into subsets of voxels sharing the exact same combination of classification criteria [S1, unofficial]. Construct the auxiliary voxel model $LV$ by assigning a unique integer identifier to each cluster [S1, unofficial].
4.  **Graph Completion:** Identify dependencies between the constructed clusters to complete the graph structure, ensuring the graph accurately reflects the logical structure of the multiclassified model [S1, unofficial].
5.  **Output:** Store the directed graph $G$ and the labelled voxel model $LV$ [S1, unofficial].

**Rendering Steps:**
1.  **Query Parsing:** The user expresses a rendering query as a boolean expression of the initial classification criteria [S1, unofficial]. A parser takes the boolean query and the graph structure $G$ as input [S1, unofficial]. The parser computes the CTF and GMTF on the cluster label domain using structural F+C rules derived from the graph [S1, unofficial].
2.  **Multimodal 3D Texture Mapping:** Perform rendering using a strategy that simultaneously accesses the original voxel model $V$ and the labelled voxel model $LV$ [S1, unofficial]. For each fragment (ray sample):
    *   Fetch the voxel value $v = V[\mathbf{x}]$ from the original model [S1, unofficial].
    *   Fetch the cluster label $c = LV[\mathbf{x}]$ from the auxiliary model [S1, unofficial].
    *   Use $c$ to look up rendering parameters from the computed CTF and GMTF [S1, unofficial].
    *   Apply structural F+C shading using the fetched parameters (visibility, color, opacity, gradient modulation) [S1, unofficial].
    *   Merge the rendering parameters according to a user-specified weight to produce the final image [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sample_grid(grid, x, y, z):
    Dx, Dy, Dz = len(grid), len(grid[0]), len(grid[0][0])
    ix = max(0, min(Dx - 1, int(x)))
    iy = max(0, min(Dy - 1, int(y)))
    iz = max(0, min(Dz - 1, int(z)))
    return grid[ix][iy][iz]

def preprocess(V, criteria):
    Dx, Dy, Dz = len(V), len(V[0]), len(V[0][0])
    cluster_map = {}
    next_id = 0
    LV = [[[0]*Dz for _ in range(Dy)] for _ in range(Dx)]
    for x in range(Dx):
        for y in range(Dy):
            for z in range(Dz):
                v = V[x][y][z]
                satisfied = tuple(sorted(c(v) for c in criteria))
                if satisfied not in cluster_map:
                    cluster_map[satisfied] = next_id
                    next_id += 1
                LV[x][y][z] = cluster_map[satisfied]
    num_criteria = len(criteria)
    G = {i: set() for i in range(num_criteria)}
    for i in range(num_criteria):
        for j in range(num_criteria):
            if i != j:
                implies = all(criteria[i](V[x][y][z]) <= criteria[j](V[x][y][z])
                              for x in range(Dx) for y in range(Dy) for z in range(Dz))
                if implies:
                    G[i].add(j)
    return G, LV, cluster_map

def parse_query(query_str, criteria, cluster_map):
    CTF = {}
    GMTF = {}
    for cid, satisfied_criteria in cluster_map.items():
        expr = query_str
        for idx, is_sat in enumerate(satisfied_criteria):
            expr = expr.replace(f"c{idx}", str(is_sat))
        is_selected = eval(expr)
        if is_selected:
            CTF[cid] = (1.0, 1.0, 1.0)
            GMTF[cid] = 1.0
        else:
            CTF[cid] = (0.0, 0.1, 0.0)
            GMTF[cid] = 0.2
    return CTF, GMTF

def render(V, LV, CTF, GMTF, rays, step_size=0.1):
    image = []
    for ray in rays:
        color_acc = [0.0, 0.0, 0.0]
        alpha_acc = 0.0
        t = 0.0
        while t < 100.0:
            pos = ray.origin + ray.dir * t
            v = sample_grid(V, pos.x, pos.y, pos.z)
            c = sample_grid(LV, pos.x, pos.y, pos.z)
            color, opacity, visibility = CTF.get(c, (0.0, 0.0, 0.0))
            grad_mod = GMTF.get(c, 0.0)
            if visibility:
                sample = [color[i] * v for i in range(3)]
                alpha = opacity * grad_mod
                color_acc = [color_acc[i] + sample[i] * alpha * (1 - alpha_acc) for i in range(3)]
                alpha_acc += alpha * (1 - alpha_acc)
            t += step_size
        image.append(color_acc)
    return image
```

*Parameters:*
*   **CTF:** Defined on the cluster label domain. Determines visibility, color, and opacity for each cluster [S1, unofficial].
*   **GMTF:** Defined on the cluster label domain. Determines the gradient modulation factor for each cluster [S1, unofficial].
*   **Typical Values:** Not given in the sources. [S1, unofficial]

*Edge Cases and Pitfalls:*
*   **Multimodal Data:** The method explicitly supports multimodal datasets (e.g., MR/SPECT, MR/PET), but the classification criteria must be defined across modalities [S1, unofficial].
*   **Boolean Expression Parsing:** The parser relies on the graph structure to resolve boolean expressions [S1, unofficial].
*   **Cluster Label Uniqueness:** Each cluster must have a unique identifier [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. [S1, unofficial]
*   **Memory Complexity:** Not given in the sources. [S1, unofficial]
*   **Measured Performance:** Not given in the sources. [S1, unofficial]
*   **Strengths:** Explicitly represents the structure of multiclassified datasets, allowing users to explore structural relationships between classification criteria [S1, unofficial]. Supports interactive boolean queries [S1, unofficial].
*   **Limitations:** Requires preprocessing to construct the graph and labelled model. The method is specific to multiclassified scalar volumes and may not directly apply to tensor or vector fields without adaptation. [S1, unofficial]
*   **When to Use:** When the user needs to understand the logical relationships between different classification criteria in a multiclassified volume dataset. [S1, unofficial]
*   **When Not to Use:** When the data is not multiclassified or when the structural relationships between classes are not of interest. [S1, unofficial]

## Background
Structural Focus+Context rendering was proposed by P. Abellán, A. Puig, and D. Tost [S1, unofficial]. The method was received on 31 March 2007, accepted on 1 August 2007, and posted online on 27 October 2007 [S1, unofficial]. It addresses the limitation of existing illustrative visualization systems that do not explicitly represent the structure of multiclassified datasets [S1, unofficial]. The work is associated with the handling of multimodal and time-varying data, though prior systems lacked explicit structural representation [S1, unofficial].

## Sources
[S1] Structural focus+context rendering of multiclassified volume data, http://hdl.handle.net/2117/86811, fetched 2026-10-26 (unofficial)
[S2] Focus + Context Rendering of Structured Biomedical Data, https://doi.org/10.2312/vcbm/vcbm08/109-116, fetched 2026-10-26
