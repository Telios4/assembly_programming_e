sub1.asm flags result:
1.Carry Flag is Set to 1 because 50 is smaller than 80, so a borrow was needed out of bit 7.
2.Zero Flag is 0 (cleared) because the result is not zero.
3.Sign Flag is Set to 1 because Bit 7 of the result is 1.
4.Overflow Flag is 0 (cleared) because the signed result (-30) fits within the signed 8-bit range (-128 to 127).
5.Parity Flag is Set to 1 because the low byte 11100010b has an even number of 1 bits (four).
6.Auxiliary carry Flag is 0 (cleared) because the low nibble of 50 (0010b = 2) is not smaller than the low nibble of 80 (0000b = 0), so no borrow from bit 4 was needed.



| Flag | Status | Why |
|------|--------|-----|
| **CF** | Set (1) | 50 is smaller than 80, so a borrow was needed out of bit 7. |
| **ZF** | Cleared (0) | The result is not zero. |
| **SF** | Set (1) | Bit 7 of the result is 1. |
| **OF** | Cleared (0) | The signed result (-30) fits within the signed 8-bit range (-128 to 127). |
| **PF** | Set (1) | The low byte `11100010b` has four 1 bits, an even count. |
| **AF** | Cleared (0) | The low nibble of 50 (`0010b` = 2) is not smaller than the low nibble of 80 (`0000b` = 0), so no borrow from bit 4 was needed. |

---
sub2.asm flags result:
1.Carry Flag is Set to 1 because 1000 is smaller than 2000, so a borrow was needed out of bit 15.
2.Zero Flag is 0 (cleared) because the result is not zero.
3.Sign Flag is Set to 1 because Bit 15 of the result is 1.
4.Overflow Flag is 0 (cleared) because the signed result (-1000) fits within the signed 16-bit range (-32768 to 32767).
5.Parity Flag is Set to 1 because the low byte 00011000b has an even number of 1 bits (two).
6.Auxiliary carry Flag is 0 (cleared) because the low nibble of 1000 (1000b = 8) is not smaller than the low nibble of 2000 (0000b = 0), so no borrow from bit 4 was needed.



| Flag | Status | Why |
|------|--------|-----|
| **CF** | Set (1) | 1000 is smaller than 2000, so a borrow was needed out of bit 15. |
| **ZF** | Cleared (0) | The result is not zero. |
| **SF** | Set (1) | Bit 15 of the result is 1. |
| **OF** | Cleared (0) | The signed result (-1000) fits within the signed 16-bit range (-32768 to 32767). |
| **PF** | Set (1) | The low byte `00011000b` has two 1 bits, an even count. |
| **AF** | Cleared (0) | The low nibble of 1000 (`1000b` = 8) is not smaller than the low nibble of 2000 (`0000b` = 0), so no borrow from bit 4 was needed. |

