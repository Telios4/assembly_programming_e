add1.asm flags result:
1.Carry Flag remains 0 because there is no carry out of bit 7.
2.Zero Flag remains 0 because the result is not zero.
3.Sign Flag is Set to 1 because Bit 7 of the result is 1.
4.Overflow Flag is Set to 1 because 120 + 10 exceeds signed max (127).
5.Parity Flag Set to 1 because the Low byte has an even number of 1 bits (two).
6.Auxiliarycarry Flag is Set because there is a Carry from bit 3 to bit 4.