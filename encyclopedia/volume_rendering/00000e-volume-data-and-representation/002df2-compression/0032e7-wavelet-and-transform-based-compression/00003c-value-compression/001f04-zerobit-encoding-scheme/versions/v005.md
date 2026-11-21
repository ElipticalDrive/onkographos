# Zerobit Encoding Scheme

Also called: Enhanced Encoding Scheme

## Overview
The Zerobit Encoding Scheme is a compression method for wavelet-transformed volume data that improves compression ratio and random access speed by exploiting spatial coherence in coefficient clusters [S1, unofficial]. It modifies the cell tag table to indicate whether the coefficient vector $c_0$ is zero and uses a variable-length significance map based on "zerobits" to avoid storing positional information for nodes where all detail coefficients are zero [S1, unofficial]. Unlike previous schemes that used fixed-size significance maps, this method reduces storage waste for zero-valued coefficients while maintaining fast, direct access to individual coefficients [S1, unofficial].

## How it works
The method operates on a 16x16x16 unit block, which is decomposed into a set of 4x4x4 subblocks called **cells** [S1, unofficial]. Each cell is represented as a level-two multi-resolution octree structure [S1, unofficial].

### Data Structures and Layout

**1. Cell Tag Table**
A table storing one byte per cell. Cells are enumerated front-to-back, top-to-bottom, left-to-right [S1, unofficial].
*   **Tag Value 0:** Indicates the cell contains all zero coefficients.
*   **Tag Value > 0:** Indicates the cell contains non-zero coefficients. The value represents the index of the cell's information block in the Cell Information array [S1, unofficial].
*   **Bit Layout (per entry):**
    *   **Bit 0 (Left-most):** Zerobit for the root node (Level 0). If 0, all 7 detail coefficients of the root are zero.
    *   **Bit 1:** Adjacent to Bit 0.
        *   *For CT Data:* Bytetag for coefficient $c_0$ (indicates if $c_0$ is stored in two-byte stream).
        *   *For RGB Data:* Zero/non-zero flag for the $c_0$ vector $(r,g,b)$ [S1, unofficial].

**2. Cell Information**
Stored in one-byte blocks, indexed by the positive tag value from the Cell Tag Table.
*   **Zerobits (Level 1):** The first byte contains 8 bits representing the zerobits for the 8 nodes at Level 1 of the octree. If a zerobit is 0, the corresponding Level 1 node has all zero detail coefficients.
*   **Significance Map:** Variable-length array of bytes following the zerobits byte.
    *   Length: Equal to the number of zerobits with value 1 in the entire cell (Level 0 zerobit + Level 1 zerobits). Maximum 9 bytes.
    *   Each byte corresponds to a node with a zerobit of 1.
    *   **Bit Layout (per significance map byte):**
        *   **Bits 1-7:** Significance flags for the 7 detail coefficients of that node. If 1, the coefficient is non-zero and stored in a data stream.
        *   **Bit 0 (Left-most):** Two-byte flag. If 0, at least one coefficient in this node is stored in the two-byte stream. If 1, all significant coefficients in this node are stored in the one-byte stream [S1, unofficial].

**3. Bytetag**
A single byte used to track which specific detail coefficients (among those with significance flag 1) are stored in the two-byte stream. A value of 1 in the bytetag means the corresponding detail coefficient is stored in the two-byte stream [S1, unofficial].

**4. Byte Streams**
Two streams per cell:
*   **One-byte stream:** Stores coefficients that fit in one byte.
*   **Two-byte stream:** Stores coefficients requiring two bytes (or signed/unsigned char ranges for RGB).
*   Each stream maintains an **offset value** within the cell information to locate coefficients [S1, unofficial].

**5. Indexing Table T(*)**
A precomputed table with $2^8 = 256$ entries. Given a one-byte word (8 bit flags), it returns the number of bit 1s in the byte. This is used to quickly count the displacement of non-zero coefficients [S1, unofficial].

### Encoding Steps

1.  **Enumerate Cells:** Process cells in front-to-back, top-to-bottom, left-to-right order.
2.  **Tag Cells:**
    *   If all coefficients in the cell are zero, tag with 0.
    *   Otherwise, tag with a positive integer (index into Cell Information).
3.  **Determine Zerobits:**
    *   For the root node (Level 0), check if all 7 detail coefficients are zero. Set Level 0 zerobit accordingly.
    *   For each of the 8 Level 1 nodes, check if all 7 detail coefficients are zero. Set Level 1 zerobits accordingly.
4.  **Construct Significance Map:**
    *   For each node with a zerobit of 1, create a significance map byte.
    *   Set bits 1-7 based on whether each of the 7 detail coefficients is non-zero.
    *   Set bit 0 (Two-byte flag) to 0 if any coefficient in this node requires two bytes; otherwise 1.
5.  **Handle $c_0$:**
    *   **CT Data:** Store $c_0$ in the one-byte stream even if zero. Use Bit 1 of the Cell Tag Table entry as the bytetag for $c_0$ [S1, unofficial].
    *   **RGB Data:** $c_0$ is a 3-component vector $(r,g,b)$. Store in a three-byte stream. Use Bit 1 of the Cell Tag Table entry to indicate if $c_0 = (0,0,0)$. $c_0$ does not use the bytetag mechanism [S1, unofficial].
6.  **Store Detail Coefficients:**
    *   Coefficients with significance flag 1 are stored in the appropriate stream (one-byte or two-byte) based on the Two-byte flag and the Bytetag.
    *   Maintain offset values for both streams within the cell information [S1, unofficial].

### Decoding Algorithm

To decode the coefficient at index $(i, j, k)$:

1.  **Find Cell:** Identify the cell $C$ containing $(i, j, k)$.
2.  **Check Cell Tag:**
    *   If `CellTag[C] == 0`, return 0. [Case 1]
3.  **Compute Local Index:** Calculate the local index $(i', j', k')$ within cell $C$.
4.  **Check Level 0 Zerobit:**
    *   If the zerobit for the root node (Level 0) is 0, return 0. [Case 2]
5.  **Check Significance Map:**
    *   Identify the significance map byte corresponding to the node containing $(i', j', k')$.
    *   If the bit-flag for $(i', j', k')$ in that byte is 0, return 0. [Case 3]
6.  **Retrieve Value:**
    *   Calculate the displacement: the number of preceding non-zero coefficients (with flag 1) in the enumeration order. Use the Indexing Table $T(*)$ to count set bits in the significance map bytes efficiently [S1, unofficial].
    *   Add the displacement to the stream's offset value to find the address.
    *   Access the appropriate byte stream (one-byte or two-byte) and return the value. [Case 4]

### Parameters and Characteristics

*   **Zerobit Ratio:** The ratio of nodes where all 7 detail coefficients are zero.
    *   **CT Data (3.0% desired ratio):** Level 0 zerobit ratio = 0.230; Level 1 zerobit ratio = 0.949 [S1, unofficial].
    *   **RGB Data (1.0% desired ratio):** Level 0 zerobit ratio = 0.754; Level 1 zerobit ratio = 0.993 [S1, unofficial].
*   **Data Types:**
    *   **CT Data:** Coefficients typically fit in one-byte or two-byte streams.
    *   **RGB Data:** $c_0$ components range [0, 255] (unsigned char); detail components range [-128, 127] (signed char), requiring three-byte streams [S1, unofficial].

### Edge Cases and Pitfalls

*   **CT Data $c_0$ Storage:** In CT data, $c_0$ is stored in the one-byte stream even if its value is zero. This is considered unnecessary overhead but is accepted because $c_0=0$ often implies the cell is null, making the tag 0 case more frequent [S1, unofficial].
*   **RGB Data $c_0$:** The bit adjacent to the Level 0 zerobit in the Cell Tag Table is repurposed to indicate if $c_0=(0,0,0)$, reducing waste compared to the CT bytetag approach [S1, unofficial].
*   **Two-Byte Stream Flag:** The left-most bit of the significance map byte is repurposed from unused space to indicate the existence of two-byte coefficients. A value of 0 indicates existence [S1, unofficial].

## Operating characteristics

*   **Compression Ratio:** Improves over previous schemes (which used fixed 8-byte significance maps) by using variable-length significance maps. The average number of bytes for the significance map in non-null cells is less than 8 bytes [S1, unofficial].
*   **Random Access Speed:** Supports fast random access. Decoding a single coefficient requires only a few table lookups and bit checks, avoiding full reconstruction of the cell [S1, unofficial].
*   **Memory:** Requires storage for Cell Tag Table, Cell Information (Zerobits, Significance Maps, Offsets), Bytetag, and Byte Streams. The variable-length significance map reduces memory usage for sparse data.
*   **Strengths:** High compression ratio for sparse wavelet coefficients; fast random access; no image quality loss compared to previous methods [S1, unofficial].
*   **Limitations:** Slightly more complex decoding logic due to variable-length maps and two-byte stream flags. The overhead of storing $c_0$ in CT data even when zero is a minor inefficiency [S1, unofficial].

## Background

The Zerobit Encoding Scheme (also called Enhanced Encoding Scheme) was proposed to address the trade-off between compression ratio, reconstruction speed, and image quality in interactive volume visualization systems [S1, unofficial]. It builds on previous wavelet-based compression methods [9, 10] by exploiting the spatial coherence of wavelet coefficients, where null coefficients often form thick clusters [S1, unofficial]. The method was presented in the context of the Visible Human Project (VHP) conference in 1998 [S1, unofficial].

## Sources

[S1] Compression Scheme, https://www.nlm.nih.gov/research/visible/vhpconf98/AUTHORS/PARK/SECTION3.HTM, fetched 2026-10-28 (unofficial)
