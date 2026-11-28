# Box Cover with Box Summaries

Also called: Bit string code word, Box Compression with Box Summaries, Box compression algorithm output, Indexed Vietoris-Rips Box Compression

## Overview
Box Cover with Box Summaries is the specific data structure and encoding format produced by a box compression algorithm for voxelized vector fields. It solves the problem of lossy compression by representing a set of vectors $X$ as a collection of boxes (clusters) $\mathcal{S}$, where each box is summarized by a polynomial approximation $f$ and an error bound $\varepsilon$. It is distinguished from general coverings by its rigid tuple structure and its reliance on the Indexed Vietoris-Rips Complex to define feasible clusters [S1, unofficial].

## How it works
**Inputs:**
- $X$: A set of $n$ vectors in a $(k, D)$-dimensional voxelized vector field.
- $f$: A piecewise multivariate polynomial object.
- $\varepsilon$: A lossy error bound ($\varepsilon > 0$).

**Outputs:**
- A single bit string representing the `box cover with box summaries` data structure.

**Data Structures:**
The core data structure is a tuple defined as:
$$ ((\dots, (Encode(R), \bullet, \dots)_{R \in \mathcal{S}(X, f, \varepsilon)}, (\varepsilon^*, \varepsilon, f)) $$
Where:
- $\mathcal{S}(X, f, \varepsilon)$ is the set of boxes (clusters) covering $X$.
- $R$ is a box (cluster) in $\mathcal{S}$.
- $Encode(R)$ is the encoded representation of the box.
- $\bullet$ represents delimiters.
- $\varepsilon^*$ is a derived error bound defined as $\varepsilon^* := \frac{D(X, f, 2\varepsilon)}{2}$.
- $D(X, f, 2\varepsilon)$ is the largest pairwise distance with respect to distortion $d_f$ on $X$ that is less than $2\varepsilon$.

**Encoding Steps:**
1. **Compute $\varepsilon^*$:** Calculate $D(X, f, 2\varepsilon)$ by sorting all pairwise distances on $X$ with respect to $d_f$. Set $\varepsilon^* = D(X, f, 2\varepsilon) / 2$.
2. **Encode Boxes:** For each box $R \in \mathcal{S}(X, f, \varepsilon)$, compute $Encode(R)$. The specific encoding of $R$ depends on the underlying box compression algorithm, but the output is concatenated.
3. **Encode Delimiters:** Use length-2 bit strings for delimiters:
   - Comma `,`: `00`
   - Opening parenthesis `(`: `01`
   - Closing parenthesis `)`: `10`
4. **Encode Components:** Each component of the tuple (including $Encode(R)$, $\varepsilon^*$, $\varepsilon$, and $f$) is encoded using a function $Bit(\cdot)$. The notation $Bit(\cdot) \parallel 11$ indicates that the bit string for the component is concatenated with the suffix `11`.
5. **Concatenate:** All encoded components and delimiter bit strings are concatenated to form the final single bit string.

**Formulas:**
The size of the resulting bit string is given by:
$$ \text{size}(\mathcal{C}(X, f, \varepsilon)) = |\mathcal{S}(X, f, \varepsilon)|(10r + 4\text{size}(n) + 2v(\text{size}(f), n, \text{size}(X)) + 5r + \text{size}(\varepsilon^*) + \text{size}(\varepsilon) + \text{size}(f)) $$
Where:
- $r$ is the constant number of bits to represent any delimiter.
- $\text{size}(n)$ is the bit size of $n$.
- $v(\text{size}(f), n, \text{size}(X))$ is the maximum number of bits to represent any vector entry in $X$ using $f$ for distortion $d_f$ at lossy bound $\varepsilon$.
- $\text{size}(\varepsilon^*)$ is the bit size of $\varepsilon^*$.
- $\text{size}(\varepsilon)$ is the bit size of $\varepsilon$.
- $\text{size}(f)$ is the bit size of $f$.

**Parameters:**
- $\varepsilon^*$ requires $\log(1/\varepsilon^*) = O(\text{size}(X)/n)$ bits for finite representation [S1, unofficial].

**Pitfalls:**
- The box cover with box summaries data structure does not have a post-encoding $\Lambda$ of the box clusters. Consequently, it is not fully expressive of all possible encoders, and the globally optimal solution may not be a box cover with box summaries [S1, unofficial].

## Subtopics

### [Polynomial Time Decompression](22144-polynomial-time-decompression.md)
Reconstructs a voxelized vector field from the compressed data. It uses a strict inequality constraint to ensure polynomial time complexity, distinguishing it from exact root-finding approaches.

## Operating characteristics
- **Time Complexity:** Computing $D(X, f, 2\varepsilon)$ takes $O(n^2 \log(n) + \text{size}(f)n^2)$ time, determined by sorting all pairwise distances on $X$ with respect to $d_f$ [S1, unofficial].
- **Memory/Size:** The size is linear in the number of boxes $|\mathcal{S}(X, f, \varepsilon)|$ and depends on the bit sizes of the polynomial $f$, the error bounds, and the vector entries.
- **Strengths:** Provides a closed-form size estimate and ensures feasibility within polynomial time for finding $\varepsilon^*$.
- **Limitations:** Not globally optimal because it lacks post-encoding of clusters.

## Background
The concept originates from the study of lossy compression for voxelized vector fields, specifically addressing the hardness of box covering. It is based on the Indexed Vietoris-Rips Complex, which is monotonic: for $\varepsilon^* < \varepsilon < 1$, $iVR_{2\varepsilon^*}^{df}(X) \subseteq iVR_{2\varepsilon}^{df}(X)$. Furthermore, for $r$ in the range $[D(X, f, \varepsilon), 2\varepsilon]$, the complex is constant: $iVR_r^{df}(X) = iVR_{2\varepsilon}^{df}(X)$ [S1, unofficial].

## Sources
[S1] Compression of Voxelized Vector Field Data by Boxes is Hard, https://doi.org/10.48550/arxiv.2510.20801, fetched 2026-10-26 (unofficial)
