%%%
title = "Elliptic Curve Point Encoding"
abbrev = "Elliptic Curve Point Encoding"
ipr= "none"
area = "General"
workgroup = "none"

[seriesInfo]
name = "Individual-Draft"
value = "draft-point-encoding-latest"
status = "informational"

[[author]]
initials = "V."
surname = "Kalos"
fullname = "Vasilis Kalos"
#role = "editor"
organization = "MATTR"
  [author.address]
  email = "vasilis.kalos@mattr.global"

%%%

{mainmatter}

# Introduction

This document specifies point serialization and deserialization procedures for Weierstrass curves. At a high level, the serialization operation is the following,

1. Each serialized point uses 3 extra metadata bits.
2. If compression is used, the point's serialization will consist of the octet representation of it's `x` coordinate. In that case, during de-serialization, the `y` coordinate will be derived from `x`, using the metadata bits and the curve's equation.
3. If compression is not used, the point's serialization will be comprised from the concatenation of the `x` and `y` coordinates octet representation.
4. The metadata bits are appended at the start (i.e., in the most significant bits positions) of the point's serialization.


## Notation

- shift(array) -> result: remove the first element (i.e., the element at index 0) from the array. Returns the array without its first element or INVALID, if the input array is empty.

# Preliminaries

## Curves and Base Fields

For pairing-based applications, there are often 2 distinct curves that are defined by the system's public parameters. Serialization and de-serialization of a point, should convey which of the 2 curves the point belongs to.

- Fp: Field of prime odd order `p`, comprising of scalar values between 0 and p - 1

- Fq: Extension field of order q = p^m. Elements are irreducible polynomials of order `m-1`, i.e., `s_(m-1) * x^(m-1) +  ...  s_2 * x^2 + s_1 * x + s_0`, where each `s_i` is an element of `Fp`. An element `s` of `Fq` will be represented by a the vector `s = (s_(m-1), ..., s_1, s_0)`.

- E: curve defined over `Fp`. A point of `E`, has its `x` and `y` coordinates on `Fp`

- E': curve defined over `Fq`. A point of `E'`, has its `x` and `y` coordinates on `Fq`

## Coordinates Encoding

### Coordinate Serialization

Depending on the base field upon which a curve is defined, each coordinate can be represented either as a scalar `s`, or as a vector of scalars `s = (s_(m-1), ..., s_1, s_0)`. In the first case, the serialization of the coordinate will be the serialization of, i.e., `I2OSP(s)`. In the second case, we use the serialization defined in [Section 2.5] in [@!I-D.irtf-cfrg-pairing-friendly-curves]. More specifically, the serialization of that coordinate will be the concatenation `I2OSP(s_0) || ... || I2OSP(s_(m-1))`.

We define the `to_octets(s)` operation, that will return a coordinate's `s` octet representation as defined above.

### Coordinate Deserialization

Let `octs` be a coordinate's octet representation. To get the coordinate `s` from its encoding `octs`, the field that this coordinate belongs to needs to be determined. If the length of the `octs` representation is `ceil(log2(p)/8)`, and hence its a coordinate on `Fp`, then `s = OS2IP(octs)`. If the length of `octs` is `ceil(log2(q)/8)`, and hence its a coordinate on `Fq`, divide `octs` to `m` parts `s_(m-1), ..., s_0` of length `ceil(log2(p)/8)` each and set `s = (s_0, s_1, ..., s_(m-1))`. If the division is not exact, return an error.

More specifically, we define the `from_octets` operation as follows,

```
res = from_octets(octs)

Inputs:

- octs: octet string

Outputs:

- res: an element of either Fp or Fq, or INVALID

Procedure:

1.  if length(octs) = ceil(log2(p)/8):
2.      return OS2IP(octs)
3.  if length(octs) = ceil(log2(q)/8):
4.      bf = ceil(log2(p)/8)
5.      m = floor(length(octs) / bf)
6.      if m * bf != length(octs): return INVALID
7.      for i in (0, ..., m-1):
8.          r_i = s[(i * bf)..((i+1) * bf - 1)]
9.      return (OS2IP(r_0), OS2IP(r_1), ..., OS2IP(r_(m-1)))
10. else: return INVALID
```

## Point Sign

For any given, non-zero `x` coordinate, there are 2 points on the elliptic curve with that value as their `x` coordinate; `P = (x, y)` and `-P = (x, -y)` (where `-y = p - y mod p`). To define which of those points is the expected one during deserialization, the following functions are used, that will return a bit corresponding to the sign of the point's `y` coordinate.

If the point `P` is on `E` (and hence it has coordinates in `Fp`):

```
sign_P_Fp(y) = 1 if y > (p - 1)/2 else 0
```

If the point `P` is on `E'`, its coordinates will be in `Fq`. Note that if `y != 0`, the first non zero element of `y` and `-y` will be at the same position. Assume that position is `i`. Set the sign of `y` to the output of `sign_P_Fp(y[i])`. More specifically,

```
bit = sign_P_Fq(y)

Inputs:

- y: an element of Fq

Outputs:

- bit: either a 0 or 1 bit

Procedure:

1. for y_i in y:
2.     if y_i != 0:
3.         return sign_P_Fp(y_i)
4. return 0
```

# Metadata Bits

The 3 metadata bits are the following,

1. `C_bit`: 1 if point compression is used, 0 otherwise
2. `I_bit`: 1 if the point is the indemnity point of the curve (point on Infinity), 0 otherwise.
3. `S_bit`: For point `P = (x, y)`, the `S_bit` will be the output of `sign_P_Fp(y)` if `P` is a point of `E`, or the output of `sign_P_Fq(y)` if `P` is a point of `E'`.

# Serialization

Let point `P = (x, y)`. Define `rb` to be the number of most significant bits that are always free (i.e., always set to 0) in the encoding of the `x` coordinate. To calculate `rb`, define `l` to be the length of the octets representation of `x`, i.e., let `l = ceil(log2(p)/8)` and,

```
rb = l * 8 - ceil(log2(p))
```

If there are at least 3 free bits in the encoding of `x` (i.e., if `rb >= 3`), then all the metadata bits will be encoded using those bits. Otherwise, if there are not enough free bits (i.e., `rb < 3`), one extra octet will be appended at the start of the point's octet string representation, to encode the metadata bits that did not "fit".

Set the `C_bit`, `I_bit` and `S_bit` as in (#metadata-bits). Also set,

```
idx = 7 - rb
```

Let `metadata` to be 2 octets representing the number

```
C_bit * 2 ^ (idx + 3) + I_bit * 2 ^ (idx + 2) + S_bit * 2 ^ (idx + 1)
```

Note that if `rb >= 3` the first octet of the `metadata` will be 0, and only the second octet will be used.

Set `x_string = to_octets(x)` and `L = length(x_string)`, where the `to_octets` operation is defined in (#coordinate-serialization). Then do,

- If `C_bit` is 0, meaning that compression will not be used, set `y_string = to_octets(y)`. Let `serialized` be an octet string of all 0s and length `2 * L + 1` (i.e., 1 octet longer than `x_string || y_string`). Set

    ```
    serialized[1..] = x_string || y_string
    ```

- If `C_bit` is 1, meaning that compression is used, set `serialized` to be an octet string of all 0s and length `L + 1` (i.e., 1 octet longer than `x_string`). Set
    ```
    serialized[1..] = x_string
    ```

To encode the metadata, do,

```
1. serialized[0] = metadata[0]
2. serialized[1] = serialized[1] XOR metadata[1]
```

If `serialized[0] == 0` it means that the extra octet was not needed to encode the metadata bits. In that case truncate the extra octet, i.e., set `serialized = shift(serialized)`, where `shift()` as defined in (#notation).

return `serialized` as the point serialization result.


# Deserialization

Let `serialized` to be the serialization of a point `P`. Let `l = ceil(log2(p)/8)` and `rb` to be as in [Serialization](#serialization).

If `length(compressed) = l + 1` it means that an extra octet was used to encode (some) of the metadata bits (i.e., 3 bits did not remain free in the `x` coordinate encoding, meaning that `rb < 3`). Otherwise, all the metadata bits are encoded in the last octet of the `x` coordinate (i.e., `rb > 3`).

Set `metadata` to be 2 octets set to all 0s. Do the following

```
1. if rb < 3:
2.     metadata[0] = serialized[0]
3.     metadata[1] = serialized[1]

       // remove the first octet that was used only for metadata
4.     serialized = shift(serialized)
5. else:
6.     metadata[1] = serialized[0]
```

Note that at this point, the second `metadata` octet, will "contain" bits that are part of the encoding of `x`. More specifically, only the `rb` (which is possibly 0) of the `serialized[0]` octet are used for the metadata bits, and the rest are part of `x`'s octets representation. To separate the 2 do the following,

```
// Mask the rb most significant bits of the metadata
1. mask = 2 ^ (i + 3) + 2 ^ (i + 2) + 2 ^ (i + 1)

// keep the metadata bits
2. metadata[1] = metadata[1] AND mask

// remove the metadata bits from the encoding of x
3. serialized[0] = serialized[0] AND !mask
```

To get the actual metadata bits, set `idx = 7 - rb` and find `C_bit`, `I_bit` and  `S_bit` such that

```
metadata =
 = C_bit * 2 ^ (idx + 3) + I_bit * 2 ^ (idx + 2) + S_bit * 2 ^ (idx + 1)
```

**Note** this can be done by moving those bits on the "start" of the metadata octets by doing `metadata >> 8 - rb`, and then setting  `C_bit`, `I_bit` and  `S_bit` to be the 3 least significant bits of `metadata[1]`. If any other bit in metadata is set, ABORT decoding.

To deserialize the point do,

1. Determine the curve of the point based on the length of the `serialized` octet string:
    - If `length(serialized) = ceil(log2(p)/8)` or `length(serialized) = 2 * ceil(log2(p)/8)`, then the curve of the point was `E`.
    - If `length(serialized) = ceil(log2(q)/8)` or `length(serialized) = 2 * ceil(log2(q)/8)`, hen the curve of the point was `E'`.
    - In any other case, ABORT decoding.
2. If `I_bit` is set, then if `serialized` is comprised from all 0s, return the identity point of the curve determined in step 1. If `serialized` is not all 0s, ABORT.
2. If `C_bit` is 0, meaning that compression is not used, divide `serialized` into 2 octet string `s_0` and `s_1` of equal length so that `serialized = s_0 || s_1`. Set `x = from_octets(s_0)` and `y = from_octets(s_1)`.
    - Set `P = (x, y)`. If `P` is a point of the curve determined in step 1, return `P`. Otherwise, ABORT decoding.
4. If `C_bit` is 1, do the following,
    - Set `x = from_octets(serialized)`, where the `from_octets()` as defined in (#coordinate-deserialization).
    - Solve the equation of the curve defined in step 1 to determine the `y` coordinate. If `y = 0` and `S_bit = 1` ABORT. Otherwise do the following,
        - If the curve determined in step 1 was `E`, then if `S_bit = sign_P_Fp(y)` return `P = (x, y)`. Otherwise, return `P = (x, -y)`.
        - If the curve determined in step 1 was `E'`, then if `S_bit = sign_P_Fq(y)` return `P = (x, y)`. Otherwise, return `P = (x, -y)`.

{backmatter}
