add1.asm flags result:
1.Carry Flag remains 0 because there is no carry out of bit 7.
2.Zero Flag remains 0 because the result is not zero.
3.Sign Flag is Set to 1 because Bit 7 of the result is 1.
4.Overflow Flag is Set to 1 because 120 + 10 exceeds signed max (127).
5.Parity Flag Set to 1 because the Low byte has an even number of 1 bits (two).
6.Auxiliarycarry Flag is Set because there is a Carry from bit 3 to bit 4.

| Flag | Status | Why |
|------|--------|-----|
| **CF** | Cleared (0) | No carry out of bit 7, since 130 fits in 8 bits (max 255). |
| **ZF** | Cleared (0) | The result is not zero. |
| **SF** | Set (1) | Bit 7 of the result is 1. |
| **OF** | Set (1) | 120 + 10 exceeds the signed 8-bit maximum (127). |
| **PF** | Set (1) | The low byte `10000010b` has two 1 bits, an even count. |
| **AF** | Set (1) | There is a carry from bit 3 to bit 4. |

---

add2.asm flags result:
1.Carry Flag remains 0 because there is no carry out of bit 15 (the sum fits in 16 bits, 65535 or less).
2.Zero Flag remains 0 because the result is not zero.
3.Sign Flag remains 0 because Bit 15 of the result is 0.
4.Overflow Flag remains 0 because the sum does not exceed the signed 16-bit maximum (32767).
5.Parity Flag remains 0 because the low byte of the result has an odd number of 1 bits.
6.Auxiliary carry Flag remains 0 because there is no carry from bit 3 to bit 4.


| Flag | Status | Why |
|------|--------|-----|
| **CF** | Cleared (0) | No carry out of bit 15, since the sum fits in 16 bits (max 65535). |
| **ZF** | Cleared (0) | The result is not zero. |
| **SF** | Cleared (0) | Bit 15 of the result is 0. |
| **OF** | Cleared (0) | The sum does not exceed the signed 16-bit maximum (32767). |
| **PF** | Cleared (0) | The low byte of the result has an odd number of 1 bits. |
| **AF** | Cleared (0) | No carry from bit 3 to bit 4. |

