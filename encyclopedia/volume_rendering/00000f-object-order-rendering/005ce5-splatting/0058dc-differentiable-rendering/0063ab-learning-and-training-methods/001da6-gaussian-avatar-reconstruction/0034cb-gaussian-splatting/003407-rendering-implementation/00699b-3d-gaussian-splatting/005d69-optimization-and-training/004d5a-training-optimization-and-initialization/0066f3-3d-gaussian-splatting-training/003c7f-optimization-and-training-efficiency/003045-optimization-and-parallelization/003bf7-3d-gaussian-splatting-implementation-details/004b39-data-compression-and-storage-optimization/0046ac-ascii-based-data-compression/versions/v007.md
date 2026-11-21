# ASCII-Based Data Compression
Also called: Anywhere Compression, Base 94 Floating-Point Compression, Base 95 Integer Compression

## Overview
This method reduces the storage size and transmission time of 3D Gaussian Splatting attributes by encoding numerical values into ASCII strings using base-95 (integers) and base-94 (floating-point) systems [S1, unofficial]. It distinguishes itself from sibling methods like [Sensitivity-Aware Vector Clustering](12167-sensitivity-aware-vector-clustering.md) and [Compact 3D Gaussian Representation (C3DGS)](12927-compact-3d-gaussian-representation-c3dgs.md) by focusing on data transmission efficiency during training rather than spatial redundancy or codebook quantization [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Continuous numerical values (integers or floating-point numbers) representing Gaussian attributes (position, scale, rotation, opacity, color).
*   **Output:** A string of ASCII characters representing the compressed value.

**Data Structures:**
*   **Integer Encoding:** Maps values in the range $[0, 95]$ to a single ASCII character.
*   **Floating-Point Encoding:** Maps the integer component to a single ASCII character from the range $[0, 94]$, followed by a separator character, followed by the decimal places.

**Steps:**

1.  **Integer Compression:**
    *   Select ASCII characters with codes 32 to 126 (inclusive). There are 95 such characters.
    *   Map each integer value $v \in [0, 95]$ to the corresponding ASCII character $c$.
    *   The encoded string is simply $c$.
    *   **Decoding:** Convert the ASCII character $c$ back to its decimal integer value $v$.

2.  **Floating-Point Compression:**
    *   Select ASCII characters with codes 32 to 125 (inclusive) for the integer part. There are 94 such characters.
    *   Use ASCII character 126 (`~`) as the separator between the integer and fractional parts.
    *   Let the floating-point number be $f$.
    *   Extract the integer part $I$ and the fractional part. The source specifies that the fractional part is represented by the number of decimal places $d$.
    *   **Encoding:**
        *   Encode $I$ using the base-94 mapping to a single ASCII character $c_I$.
        *   The separator is the character $c_{sep} = \text{chr}(126)$.
        *   The fractional part is encoded as the integer $d$ (number of decimal places). Note: The source implies $d$ is a single digit or small integer, but does not specify a base for $d$ other than it being part of the string. However, the decoding formula suggests $d$ is the exponent.
        *   The encoded string is $c_I + c_{sep} + \text{str}(d)$.
    *   **Decoding:**
        *   Split the string at the separator `~`.
        *   Let the first part be $s_I$ and the second part be $s_d$.
        *   Decode $s_I$ to get integer $I$.
        *   Decode $s_d$ to get integer $d$.
        *   Calculate the final value $f = I \times 10^{-d}$.

3.  **Color Matrix Optimization:**
    *   The color matrix values are constrained to the range $[0.0, 2.0]$.
    *   Each value is rounded to one decimal place.
    *   This results in 21 possible values: $0.0, 0.1, \dots, 2.0$.
    *   These values are then compressed using the ASCII string compression algorithm described above.
    *   **Note:** The source states that compressing the color matrix into ASCII strings is difficult due to the processing time required for traversing large matrices. Internal functions like `torch.where` and `torch.index_select` are used to replace `for` loops, but this is still not fast enough to support full replacement in all contexts [S1, unofficial].

**Parameters:**
*   **Integer Base:** 95.
*   **Floating-Point Integer Base:** 94.
*   **Separator:** ASCII 126 (`~`).
*   **Color Precision:** 1 decimal place.

**Edge Cases and Pitfalls:**
*   **Matrix Traversal:** Compressing large color matrices is computationally expensive. Using vectorized operations (`torch.where`, `torch.index_select`) helps but may not be sufficient for full replacement [S1, unofficial].
*   **Line Breaks:** Using line break characters in ASCII strings adds an extra operator in writing and decoding, which should be avoided [S1, unofficial].
*   **Range Limitations:** The integer compression is limited to $[0, 95]$. Values outside this range require multiple characters or a different encoding scheme, which is not detailed in the source [S1, unofficial].
*   **Floating-Point Precision:** The decoding formula $f = I \times 10^{-d}$ assumes that the fractional part is represented solely by the number of decimal places $d$. This implies that the integer part $I$ contains the significant digits, and $d$ scales them. This is a non-standard floating-point representation and may lose precision for numbers with many significant digits [S1, unofficial].

## Operating characteristics
*   **Training Speed:** Gains approximately 6% or less improvement in training speed [S1, unofficial].
*   **Storage:** Reduces data size significantly for color matrices by limiting precision to one decimal place and using ASCII compression [S1, unofficial].
*   **Limitations:** Not suitable for compressing the entire color matrix efficiently due to traversal overhead [S1, unofficial]. The method is primarily effective for data transmission during training rather than final storage or rendering [S1, unofficial].

## Background
This method was proposed in the paper "I3DGS: Improve 3D Gaussian Splatting from Multiple Dimensions" [S1, unofficial]. The research focuses on improving the training speed of 3D Gaussian Splatting models through various optimizations, including compression, learning, structure design, and color adjustments [S1, unofficial].

## Sources
[S1] I3DGS: Improve 3D Gaussian Splatting from Multiple Dimensions, https://arxiv.org/abs/2405.06408v1, fetched 2026-10-26 (unofficial)
