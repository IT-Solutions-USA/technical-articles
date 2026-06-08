# Galois Fields in AES: GF(2^8), Irreducible Polynomials, and the Extended Euclidean Algorithm

**Date:** May 2, 2026  
**Author:** IT Solutions USA  
**Category:** Cryptography / Mathematics  
**Tags:** AES, Galois Field, GF(2^8), Cryptography, Finite Field, Extended Euclidean Algorithm, Irreducible Polynomial, SubBytes, S-Box

---

AES — the symmetric cipher protecting the majority of encrypted data on the internet — is built on a branch of mathematics that most engineers never encounter outside a cryptography course: finite field arithmetic, specifically the Galois Field GF(2⁸). Understanding this foundation is not merely academic. It explains why AES is structured the way it is, why its operations are so efficient in hardware and software, and what the SubBytes S-box actually computes.

---

## What Is a Galois Field?

A Galois Field GF(pⁿ) is a finite set of elements with two operations — addition and multiplication — that obey all the standard rules of arithmetic: associativity, commutativity, distributivity, identity elements, and (for every nonzero element) a multiplicative inverse. The distinctive property is that the set is finite: there are exactly pⁿ elements, and every result of an operation lands back in the same set.

For AES, p = 2 and n = 8, giving GF(2⁸) with 256 elements. The choice of base 2 is deliberate: it aligns directly with binary computing. Each of the 256 elements corresponds to a unique 8-bit byte.

---

## Representing Elements as Polynomials

Each element of GF(2⁸) is represented as a polynomial of degree at most 7 with coefficients drawn from GF(2) — that is, each coefficient is either 0 or 1. A general element looks like:

```
a₇x⁷ + a₆x⁶ + a₅x⁵ + a₄x⁴ + a₃x³ + a₂x² + a₁x + a₀
```

where each aᵢ ∈ {0, 1}. This maps directly to an 8-bit binary representation: the byte `0x57` = `0101 0111` represents the polynomial x⁶ + x⁴ + x² + x + 1.

### GF(2³) Elements (illustrative — same structure as GF(2⁸), smaller)

| Binary | Polynomial |
|---|---|
| 000 | 0 |
| 001 | 1 |
| 010 | x |
| 011 | x + 1 |
| 100 | x² |
| 101 | x² + 1 |
| 110 | x² + x |
| 111 | x² + x + 1 |

---

## Addition in GF(2⁸): Bitwise XOR

Adding two elements in GF(2⁸) is performed coefficient-by-coefficient modulo 2 — which is exactly XOR. There are no carries, no overflow, no reduction needed.

**Example:** `0x57 ⊕ 0x83 = 0xD4`

```
0101 0111  (x⁶ + x⁴ + x² + x + 1)
1000 0011  (x⁷ + x + 1)
─────────
1101 0100  (x⁷ + x⁶ + x⁴ + x²)  = 0xD4
```

Subtraction in GF(2) is identical to addition — there are no negative numbers, and a + a = 0 for every element.

---

## Multiplication and the Irreducible Polynomial

Multiplying two polynomials can produce a result of degree 14 or higher — well outside the 0–7 range required for GF(2⁸). To bring the result back into the field, it is reduced modulo an irreducible polynomial: a polynomial that cannot be factored into lower-degree polynomials over GF(2), analogous to a prime number.

AES uses the irreducible polynomial:

```
P(x) = x⁸ + x⁴ + x³ + x + 1   (hex: 0x11B)
```

Any product that exceeds degree 7 is reduced by XOR with `0x11B`, bringing it back into the 0–255 range.

---

## Why GF(2⁸)? Four Design Reasons

| Reason | Detail |
|---|---|
| Operations stay within the field | GF(2⁸) is closed: any addition, subtraction, multiplication, or division of two field elements produces another field element. There is no overflow — the irreducible polynomial wraps the result back in range. This guarantees correctness without bitmask side conditions. |
| Every nonzero element has an inverse | In GF(2⁸), every nonzero element has a unique multiplicative inverse — also in GF(2⁸). This is the property exploited by the SubBytes step of AES to build the S-box. |
| Hardware efficiency | Addition in GF(2⁸) is bitwise XOR — a single CPU instruction. Multiplication can be implemented with shift-and-XOR trees. The entire AES round function can be built from XOR and table lookups, enabling extremely fast software and compact hardware implementations. |
| Byte alignment | 2⁸ = 256 elements maps exactly to the 256 possible values of a single byte. This makes GF(2⁸) arithmetic a natural fit for block cipher design operating on byte-level state matrices. |

---

## The Extended Euclidean Algorithm for Inverses

Every nonzero element of GF(2⁸) has a multiplicative inverse — another element that, when multiplied together, gives 1. Finding this inverse for an arbitrary element requires the Extended Euclidean Algorithm (EEA), which computes both the GCD of two polynomials and the Bézout coefficients that express that GCD as a linear combination of the inputs.

Given A(x) and P(x), the EEA finds U(x) and V(x) such that:

```
A(x) · U(x) + P(x) · V(x) = GCD(A(x), P(x))
```

When GCD = 1 (which is guaranteed for any nonzero A(x) and an irreducible P(x)), U(x) is the multiplicative inverse of A(x) modulo P(x).

---

## Worked Example: Inverse of x² + 1 in GF(2³)

We use GF(2³) with irreducible polynomial P(x) = x³ + x + 1 for clarity — the structure is identical to GF(2⁸), just smaller. We will find the inverse of A(x) = x² + 1.

**Goal:** find U(x) such that (x² + 1) · U(x) ≡ 1 (mod x³ + x + 1)

**Division step:** Divide P(x) by A(x):

| Step | Polynomial | Binary |
|---|---|---|
| Dividend | x³ + x + 1 | 1  0  1  1 |
| q · (x²+1) = x·(x²+1) | x³ + x | 1  0  1  0 |
| XOR (subtract in GF(2)) | 1 | 0  0  0  1 |

The remainder is 1. Since GCD(x² + 1, x³ + x + 1) = 1, the inverse exists. Tracing the Bézout coefficient:

```
1 = (x³ + x + 1) − x · (x² + 1)
  = 1 · P(x) + (−x) · A(x)
  ≡ x · A(x)   (mod P(x), since −1 = 1 in GF(2))
```

Therefore: **(x² + 1)⁻¹ = x** in GF(2³).

**Verification:** (x² + 1) · x = x³ + x ≡ (x + 1) + x = 1 (mod x³ + x + 1) ✓

### Inverse Table for GF(2³) with P(x) = x³ + x + 1

| Element | Binary | Inverse | Binary | Note |
|---|---|---|---|---|
| 1 | 001 | 1 | 001 | Self-inverse |
| x | 010 | x² + 1 | 101 | |
| x + 1 | 011 | x² + x + 1 | 111 | |
| x² | 100 | x + 1 | 011 | |
| x² + 1 | 101 | x | 010 | EEA example above |
| x² + x | 110 | x² + x | 110 | Self-inverse |
| x² + x + 1 | 111 | x + 1 | 011 | |

---

## Where Galois Field Arithmetic Appears in AES

| AES Operation | Uses | How |
|---|---|---|
| SubBytes | Multiplicative inverse in GF(2⁸) | For each byte b in the state matrix, compute b⁻¹ in GF(2⁸) (the EEA gives this), then apply an affine transformation over GF(2). The 0x00 byte maps to 0x00 by convention. The result is the 256-entry AES S-box. |
| AddRoundKey | Bitwise XOR = addition in GF(2⁸) | Each byte of the state is XORed with the corresponding byte of the round key. XOR is addition in GF(2⁸). No field structure beyond addition is required here. |
| ShiftRows | No field arithmetic | Cyclic left-rotation of rows in the state matrix. A purely structural permutation step — no GF arithmetic involved. |
| MixColumns | Polynomial multiplication in GF(2⁸) | Each column of the state is treated as a polynomial over GF(2⁸) and multiplied by a fixed polynomial modulo x⁴ + 1. This is where the full GF(2⁸) multiplication (with the irreducible polynomial 0x11B) is used. |

---

## Extended Euclidean Algorithm — General Reference

The following is the complete general form of the Extended Euclidean Algorithm for computing multiplicative inverses in any GF(2ᵐ). The worked example above applies this procedure with m = 3; AES applies it with m = 8 using P(x) = x⁸ + x⁴ + x³ + x + 1.

**Inputs:** A(x) (the element to invert), P(x) (the irreducible polynomial)  
**Output:** A(x)⁻¹ mod P(x)

```
Initialise:
  r₀ = P(x),  r₁ = A(x)
  t₀ = 0,     t₁ = 1

While r₁ ≠ 0:
  q  = floor(r₀ / r₁)       (polynomial division)
  r₂ = r₀ − q · r₁          (all arithmetic mod 2 on coefficients)
  t₂ = t₀ − q · t₁          (track Bézout coefficient)

  r₀ = r₁,  r₁ = r₂
  t₀ = t₁,  t₁ = t₂

Result: t₀ is A(x)⁻¹ mod P(x)
```

All polynomial coefficient arithmetic uses mod 2 (XOR). Polynomial division uses the same XOR-based long division shown in the worked example.
