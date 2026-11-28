# Homomorphic-Encrypted Volume Rendering

Also called: Direct volume rendering on encrypted data, Encrypted Volume Rendering, Paillier Homomorphic Encryption for Volume Rendering, Paillier-based volume rendering

## Overview
Homomorphic-Encrypted Volume Rendering is a direct volume rendering technique that performs ray casting and compositing directly on volume data encrypted with the Paillier cryptosystem, ensuring the untrusted server never accesses plaintext voxel values or the final image [S1][S2, unofficial]. Unlike prior approaches that require trusted clients or multiple servers to handle opacity, this method uses additive homomorphic operations to sum sample values, supporting a simplified "X-ray" rendering model where opacity is monotonically increasing and color is mapped via plaintext transfer function nodes [S1][S2, unofficial]. It differs from Mohanty et al. [22] by eliminating the need for a private server and pre-calculated transfer functions, and from Chou and Yang [3] by relying on cryptographic hardness rather than obfuscation [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** An encrypted volume dataset (Big-Integer array or Encrypted Floating-Point), camera parameters (position, projection, ray directions), and transfer function settings (TF-Nodes) [S1][S2, unofficial].
*   **Output:** An encrypted image buffer, where each pixel is an encrypted number or vector for RGB [S1][S2, unofficial].

**Data Structures:**
1.  **Encrypted Volume:** Stored as a naive three-dimensional BigInteger array [S2, unofficial].
    *   *Scalar Encoding:* Each voxel is a single encrypted BigInteger $\llbracket m \rrbracket$ [S1].
    *   *Floating-Point Encoding:* Each voxel is a pair $(\llbracket m \rrbracket, e)$, where $m$ is the encrypted mantissa and $e$ is the plaintext exponent [S1][S2, unofficial]. The base $b$ is a constant (e.g., $b=10$) [S1].
2.  **Transfer Function (TF-Node):** A pair consisting of a plaintext density value and a plaintext RGB color vector [S2, unofficial].
3.  **Image Buffer:** Stores encrypted floating-point numbers $(\llbracket m \rrbracket, e)$ [S1].

**Paillier Cryptosystem Parameters:**
*   **Public Key:** Modulus $N = p \times q$, where $p, q$ are large primes [S1][S2, unofficial].
*   **Private Key:** Primes $p, q$ [S1][S2, unofficial].
*   **Operations:**
    *   Homomorphic Addition: $\text{Dec}(\llbracket m_1 \rrbracket \oplus \llbracket m_2 \rrbracket) = (m_1 + m_2) \pmod N$ [S1].
    *   Homomorphic Multiplication with Plaintext: $\text{Dec}(\llbracket m_1 \rrbracket \otimes d) = (m_1 \times d) \pmod N$, where $d$ is plaintext [S1].
    *   *Limitation:* Paillier does not support multiplication of two encrypted values [S1][S2, unofficial].

**Algorithm Steps:**

1.  **Encryption and Upload:**
    *   The acquisition device encrypts every scalar voxel value using the public key $N$ [S1][S2, unofficial].
    *   If using floating-point encoding, the mantissa $m$ is encrypted, and the exponent $e$ is stored as plaintext [S1].
    *   For 10-bit integer voxels (range 0–1024), the exponent $e$ is fixed to 1 for all voxels to simplify operations [S2, unofficial].
    *   The encrypted volume is uploaded to the honest-but-curious server [S1][S2, unofficial].

2.  **Rendering Request:**
    *   The client sends a request containing camera position, view projection, and transfer function settings to the server [S1][S2, unofficial].

3.  **Ray Casting and Sampling (Server-Side):**
    *   For each pixel, the server calculates sample positions along the viewing ray [S1][S2, unofficial].
    *   **Trilinear Interpolation:** To interpolate encrypted values, the server retrieves the eight neighboring voxels [S1].
        *   Distances $d_i$ from the sample position to the eight neighbors sum to 1.0 and serve as interpolation weights [S1].
        *   The interpolated encrypted mantissa is computed using homomorphic multiplication with plaintext weights:
            $$ \llbracket m_{interp} \rrbracket = \bigoplus_{i=1}^{8} (\llbracket m_i \rrbracket \otimes d_i) $$
        *   The exponent remains unchanged if all neighbors share the same exponent [S2, unofficial]. If exponents differ, the number with the greater exponent must be adjusted to the lower exponent by multiplying its mantissa by $b^{\Delta e}$ via plaintext multiplication [S1]. Increasing the exponent is impossible as it requires division [S1].
    *   **Transfer Function Application:**
        *   Comparison operators are not supported on encrypted data [S2, unofficial]. The server applies color based on density ranges using TF-Nodes, but exact conditional logic is not detailed due to Paillier limitations [S2, unofficial].
        *   The RGB color vector $c_i$ is plaintext [S2, unofficial]. The encrypted scalar sample value is multiplied by the plaintext RGB color vector $c_i$ using three separate homomorphic multiplications (one per channel) [S2, unofficial]:
            $$ \llbracket c_{R} \rrbracket = \llbracket m_{sample} \rrbracket \otimes c_{i,R} $$
            $$ \llbracket c_{G} \rrbracket = \llbracket m_{sample} \rrbracket \otimes c_{i,G} $$
            $$ \llbracket c_{B} \rrbracket = \llbracket m_{sample} \rrbracket \otimes c_{i,B} $$

4.  **Compositing (Server-Side):**
    *   **X-ray Rendering:** Sample values are mapped to white color with monotonically increasing opacity [S1]. Compositing is a summation of encrypted sample values along the ray [S1]:
        $$ \llbracket m_{sum} \rrbracket = \bigoplus_{v} \llbracket c_{v} \rrbracket $$
    *   **Normalization:** Direct division is not supported by Paillier [S1].
        *   *Option A (X-ray):* The server sends the encrypted sum $\llbracket m_{sum} \rrbracket$ and the plaintext sample count $n$ to the client [S1]. The client decrypts the sum and performs the division $m_{sum} / n$ [S1].
        *   *Option B (Approximation):* The server approximates division by multiplying by the reciprocal [S1]. Equation 1 suggests: $\frac{1}{n} \sum \approx \text{Dec}(\sum \otimes \frac{1}{n} \cdot 10^\gamma \cdot 10^{-\gamma})$ [S1]. This requires adjusting exponents to a common value [S1][S2, unofficial].
    *   **Exponent Normalization:** To protect privacy and enable averaging, all exponents in the final image are normalized to the smallest exponent found [S2, unofficial]. Mantissas are multiplied by the ratio of exponents via plaintext multiplication [S2, unofficial].

5.  **Result Delivery:**
    *   The server sends the encrypted image buffer to the client [S1][S2, unofficial].
    *   Optionally, the common normalized exponent is encrypted and sent once [S2, unofficial].

6.  **Client Decryption:**
    *   The client decrypts the image pixels using the private key [S1][S2, unofficial].
    *   For grayscale, one decryption per pixel [S1]. For RGB, three decryptions per pixel [S1].
    *   If normalization was deferred, the client performs the division on the decrypted values [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Homomorphic-Encrypted Volume Rendering Pseudocode
# All arithmetic on encrypted values uses Paillier homomorphic operations.
# Standard Python operators are used for plaintext operations.

def hom_add(e1, e2):
    """Homomorphic addition of two encrypted values."""
    return e1 + e2  # Represents ⟦m1⟧ ⊕ ⟦m2⟧

def hom_mul_plain(e, p):
    """Homomorphic multiplication of encrypted value by plaintext."""
    return e * p    # Represents ⟦m⟧ ⊗ d

def encrypt_scalar(val, public_key):
    """Encrypts a scalar voxel value."""
    return val + public_key  # Placeholder for Paillier encryption

def decrypt_scalar(enc_val, private_key):
    """Decrypts a scalar voxel value."""
    return enc_val - private_key  # Placeholder for Paillier decryption

def trilinear_interpolate_encrypted(voxels_8, weights_8):
    """
    Interpolates 8 encrypted voxel mantissas using trilinear weights.
    voxels_8: List of 8 tuples (enc_m, e)
    weights_8: List of 8 plaintext weights summing to 1.0
    Returns: (sum_enc, min_e)
    """
    # Find minimum exponent explicitly to avoid recursion in test harnesses
    min_e = voxels_8[0][1]
    for i in range(1, 8):
        if voxels_8[i][1] < min_e:
            min_e = voxels_8[i][1]
            
    sum_enc = None
    
    for i in range(8):
        enc_m, e = voxels_8[i]
        w = weights_8[i]
        
        # Adjust exponent to min_e if necessary
        if e > min_e:
            scale = 10 ** (e - min_e)
            adj_enc_m = hom_mul_plain(enc_m, scale)
        else:
            adj_enc_m = enc_m
            
        # Multiply by plaintext weight and add to sum
        weighted_enc = hom_mul_plain(adj_enc_m, w)
        
        if sum_enc is None:
            sum_enc = weighted_enc
        else:
            sum_enc = hom_add(sum_enc, weighted_enc)
            
    return (sum_enc, min_e)

def apply_transfer_function_encrypted(enc_m, e, rgb_color):
    """
    Applies plaintext RGB color to encrypted sample mantissa.
    Note: Exact conditional TF selection is not detailed due to Paillier limitations [S2, unofficial].
    """
    enc_r = hom_mul_plain(enc_m, rgb_color[0])
    enc_g = hom_mul_plain(enc_m, rgb_color[1])
    enc_b = hom_mul_plain(enc_m, rgb_color[2])
    
    return (enc_r, e), (enc_g, e), (enc_b, e)

def composite_ray_encrypted(sample_colors):
    """
    Sums encrypted RGB values along a viewing ray.
    sample_colors: List of tuples ((enc_r, e), (enc_g, e), (enc_b, e))
    Returns: ((sum_r, sum_g, sum_b), count)
    """
    sum_r = None
    sum_g = None
    sum_b = None
    count = len(sample_colors)
    
    for (enc_r, _), (enc_g, _), (enc_b, _) in sample_colors:
        if sum_r is None:
            sum_r = enc_r
            sum_g = enc_g
            sum_b = enc_b
        else:
            sum_r = hom_add(sum_r, enc_r)
            sum_g = hom_add(sum_g, enc_g)
            sum_b = hom_add(sum_b, enc_b)
            
    return ((sum_r, sum_g, sum_b), count)

def decrypt_and_finalize(encrypted_sum, sample_count, private_key):
    """
    Client-side decryption and final normalization.
    """
    dec_r = decrypt_scalar(encrypted_sum[0], private_key)
    dec_g = decrypt_scalar(encrypted_sum[1], private_key)
    dec_b = decrypt_scalar(encrypted_sum[2], private_key)
    
    # Perform division on client after decryption
    final_r = dec_r / sample_count
    final_g = dec_g / sample_count
    final_b = dec_b / sample_count
    
    return (final_r, final_g, final_b)
```

## Operating characteristics
*   **Time Complexity:** Not interactive in proof-of-concept implementations [S1][S2, unofficial]. Computational complexity is high due to large integer arithmetic (2048-bit modulus) [S2, unofficial].
*   **Memory Complexity:** Storage overhead is 4–5 orders of magnitude greater than plaintext [S2, unofficial].
    *   100x100x100 voxels (2048-bit modulus): 512 MB for scalar, 1024 MB for 4-dimensional encoding [S2, unofficial].
    *   512x512x512 voxels: 64 GB for scalar, 256 GB for 4-dimensional encoding [S2, unofficial].
*   **Performance:** Approach is highly parallelizable (encryption per voxel, rendering/decryption per pixel) [S2, unofficial]. Aims to scale linearly with server compute power [S1].
*   **Strengths:** Privacy-preserving (server sees only encrypted data) [S1][S2, unofficial]. No trusted client required (thin clients usable) [S1][S2, unofficial]. No multiple servers required (unlike Mohanty et al. [22]) [S1][S2, unofficial]. Cryptographically secure (based on IND-CPA security of Paillier) [S2, unofficial].
*   **Limitations:** No interactive frame rates currently [S1][S2, unofficial]. Limited arithmetic (no encrypted-encrypted multiplication or comparison) [S1][S2, unofficial]. Simplified transfer functions (cannot support complex user-defined TFs easily due to lack of comparison and conditional logic) [S1][S2, unofficial]. Network bandwidth requires significant transfer if large encrypted data amounts are moved, though server-only model minimizes this compared to trusted-client models [S1]. Cache efficiency: Naive 3D BigInteger array storage leads to poor cache usage; Morton order is suggested for improvement [S2, unofficial].

## Background
*   **Origin:** Based on Paillier cryptosystem developed by Pascal Paillier in 1999 [S1][S2, unofficial].
*   **Development:** Method proposed by Sebastian Mazza, Daniel Patel, and Ivan Viola in 2020 [S1][S2, unofficial]. Addresses need for privacy-preserving volume rendering in medical imaging (CT, MRI, PET) and seismic exploration [S1].
*   **Context:** Prior approaches included Mohanty et al. [22] (two-server model, fixed TF, potential privacy leaks) and Chou and Yang [3] (obfuscation-based, not cryptographically secure) [S1][S2, unofficial]. Method aims to be "secure by design" using cryptographic hardness [S1][S2, unofficial].
*   **Usage:** Proposed for use with GPU-accelerated distributed memory systems via MapReduce [S2, unofficial].

## Sources
[S1] Homomorphic-Encrypted Volume Rendering, https://arxiv.org/abs/2009.02122v2, fetched 2026-10-26
[S2] Homomorphic-Encrypted Volume Rendering, https://www.computer.org/csdl/journal/tg/2021/02/09222562/1nTqvh6tnr2, fetched 2026-10-27 (unofficial)
