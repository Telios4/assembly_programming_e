sub1.asm flags result:
1.Carry Flag is Set to 1 because 50 is smaller than 80, so a borrow was needed out of bit 7.
2.Zero Flag is 0 (cleared) because the result is not zero.
3.Sign Flag is Set to 1 because Bit 7 of the result is 1.
4.Overflow Flag is 0 (cleared) because the signed result (-30) fits within the signed 8-bit range (-128 to 127).
5.Parity Flag is Set to 1 because the low byte 11100010b has an even number of 1 bits (four).
6.Auxiliary carry Flag is 0 (cleared) because the low nibble of 50 (0010b = 2) is not smaller than the low nibble of 80 (0000b = 0), so no borrow from bit 4 was needed.