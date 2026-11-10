# ELSCINT1/PMSCT_RLE1 Delta Decoding

Also called: PMSCT_RLE1 decompression

## Overview
Decodes a custom RLE + Delta compressed byte stream used by ELSCINT1/PMSCT medical imaging devices into a sequence of 16-bit unsigned integers [S1, unofficial]. It differs from general-purpose codecs by using two specific escape markers (`0xa5` for RLE, `0x5a` for delta) that cannot appear as literal data, and by operating on a two-pass process (RLE expansion followed by delta reconstruction) [S1, unofficial].

## How it works
**Inputs and Outputs:**
- `inbuffer`: Pointer to the compressed input byte stream.
- `length`: Number of bytes in `inbuffer`.
- `output`: Reference to a container (e.g., `std::vector<unsigned short>`) storing the decoded 16-bit pixel values.

**Data Structures:**
- `temp`: An intermediate buffer (`std::vector<char>`) storing the RLE-decoded byte stream before delta decoding.

**Steps:**

1.  **RLE Pass:** Iterate through `inbuffer` from index `i = 0` to `length - 1`.
    -   If `inbuffer[i] == 0xa5`:
        -   Set `repeat = inbuffer[i+1] + 1` [S1, unofficial].
        -   Set `value = inbuffer[i+2]` [S1, unofficial].
        -   Append `value` to `temp` buffer `repeat` times [S1, unofficial].
        -   Increment `i` by 2 (the loop increment will handle the next byte, effectively skipping the count and value bytes) [S1, unofficial].
    -   Otherwise:
        -   Append `inbuffer[i]` to `temp` [S1, unofficial].

2.  **Delta Pass:** Initialize `delta = 0` [S1, unofficial]. Iterate through `temp` from index `i = 0` to `temp.size() - 1`.
    -   If `temp[i] == 0x5a`:
        -   Read `v1 = temp[i+1]` and `v2 = temp[i+2]` [S1, unofficial].
        -   Compute `value = (v2 * 256) + v1` [S1, unofficial].
        -   Append `value` to `output` [S1, unofficial].
        -   Set `delta = value` [S1, unofficial].
        -   Increment `i` by 2 (the loop increment handles the next byte) [S1, unofficial].
    -   Otherwise:
        -   Compute `value = temp[i] + delta` [S1, unofficial].
        -   Append `value` to `output` [S1, unofficial].
        -   Set `delta = value` [S1, unofficial].

3.  **Post-processing:** If `output.size() % 2 != 0`, resize `output` to `output.size() - 1` to remove trailing padding byte [S1, unofficial].

**Formulas:**
- RLE repeat count: $repeat = (unsigned char)inbuffer[i+1] + 1$ [S1, unofficial]
- Delta-encoded 16-bit value from marker: $value = (unsigned short)(v2 \times 256 + v1)$ [S1, unofficial]
- Delta-encoded 16-bit value from literal: $value = (unsigned short)(temp[i] + delta)$ [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def decode_elscint1(inbuffer: bytes, length: int) -> list[int]:
    temp = []
    i = 0
    while i < length:
        if inbuffer[i] == 0xa5:
            repeat = inbuffer[i + 1] + 1
            value = inbuffer[i + 2]
            temp.extend([value] * repeat)
            i += 2  # Skip count and value; loop increment handles next
        else:
            temp.append(inbuffer[i])
        i += 1

    output = []
    delta = 0
    i = 0
    while i < len(temp):
        if temp[i] == 0x5a:
            v1 = temp[i + 1]
            v2 = temp[i + 2]
            value = (v2 * 256) + v1
            output.append(value)
            delta = value
            i += 2  # Skip v1 and v2; loop increment handles next
        else:
            value = temp[i] + delta
            output.append(value)
            delta = value
        i += 1

    if len(output) % 2 != 0:
        output.pop()

    return output
```

**Parameters:**
- `inbuffer`: Pointer to the compressed input byte stream [S1, unofficial].
- `length`: Number of bytes in `inbuffer` [S1, unofficial].
- `output`: Reference to `std::vector<unsigned short>` storing the decoded 16-bit pixel values [S1, unofficial].

**Edge Cases and Pitfalls:**
- The decoded output size may be odd due to padding in the compressed stream. This is handled by truncating the last byte if `output.size() % 2 != 0` [S1, unofficial].
- The escape markers `0xa5` (RLE) and `0x5a` (Delta) must not appear as literal data values in the compressed stream, as they trigger escape sequences [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] oprogramadorreal/vize, https://github.com/oprogramadorreal/vize, fetched 2026-10-26 (unofficial)
