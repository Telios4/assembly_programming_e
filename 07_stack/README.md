# 07_stack
# 01.asm: Basic push and pop (64-bit)

Pushes two registers onto the stack and pops them back in reverse order (LIFO).

1. `mov rax, 100` and `mov rbx, 200` load two values.
2. `push rax`, `push rbx` push 100 then 200. Each push decrements `rsp` by 8.
3. `nop` is a debugger marker to inspect the stack.
4. `pop rcx` gets 200 (top of stack) and `pop rdx` gets 100. Each pop increments `rsp` by 8.
5. `mov rax, 60`, `xor rdi, rdi`, `syscall` exit with code 0.

Final values: rax = 60, rbx = 200, rcx = 200, rdx = 100, rdi = 0.

Flags at exit (set by `xor rdi, rdi`; `mov`, `push`, `pop` and `nop` do not change flags):

1. Carry Flag remains 0 because `xor` always clears CF.
2. Overflow Flag remains 0 because `xor` always clears OF.
3. Zero Flag is set (1) because `rdi XOR rdi` gives 0.
4. Sign Flag is 0 because the result 0 has its top bit clear.
5. Parity Flag is set (1) because the low byte 0 has zero 1-bits, which is an even count.

# 02.asm: Stack frame with local variables (64-bit)

Shows how a function creates a stack frame and stores local variables below `rbp`.

1. `call work` pushes the return address onto the stack and jumps to `work`.
2. `push rbp` saves the caller's base pointer, then `mov rbp, rsp` makes `rbp` the frame base.
3. `sub rsp, 16` reserves 16 bytes of space for local variables.
4. `mov qword [rbp - 8], 100` and `mov qword [rbp - 16], 200` store two locals.
5. `mov rax, [rbp - 8]` and `mov rbx, [rbp - 16]` read them back (rax = 100, rbx = 200).
6. `mov rsp, rbp` frees the locals and `pop rbp` restores the caller's base pointer.
7. `ret` pops the return address and goes back to `_start`, which exits.

Flags at exit (set by the final `xor rdi, rdi`):

1. Carry Flag remains 0 because `xor` always clears CF.
2. Overflow Flag remains 0 because `xor` always clears OF.
3. Zero Flag is set (1) because the result is 0.
4. Sign Flag is 0 because the result 0 has its top bit clear.
5. Parity Flag is set (1) because 0 has an even number of 1-bits.

# 03.asm: call and ret (64-bit)

Shows how `call` and `ret` use the stack to return to the right place, with no stack frame.

1. `mov rax, 10` loads the starting value.
2. `call calculate` pushes the address of `after_call` onto the stack and jumps to `calculate`.
3. `add rax, 5` makes rax = 15.
4. `ret` pops that saved address into `rip`, so execution continues at `after_call`.
5. `nop` marks where rax should be 15, then the program exits with code 0.

Flags at the `nop` (set by `add rax, 5`, result 15):

1. Carry Flag remains 0 because 10 + 5 does not carry out of 64 bits.
2. Overflow Flag remains 0 because there is no signed overflow.
3. Zero Flag is 0 because the result 15 is not zero.
4. Sign Flag is 0 because 15 is positive (top bit clear).
5. Parity Flag is set (1) because the low byte 15 = 00001111 has four 1-bits, an even count.

# 04.asm:

The 32-bit version of 02.asm. It uses `BITS 32`, the `e` registers, and `int 0x80` for the exit syscall.

1. `call work` pushes a 4-byte return address.
2. `push ebp` and `mov ebp, esp` build the frame.
3. `sub esp, 8` reserves 8 bytes for two 4-byte locals.
4. `mov dword [ebp - 4], 100` and `mov dword [ebp - 8], 200` store the locals.
5. `eax` and `ebx` load them back (eax = 100, ebx = 200).
6. `mov esp, ebp`, `pop ebp`, `ret` tear down the frame and return.
7. `mov eax, 1`, `xor ebx, ebx`, `int 0x80` call `sys_exit` with code 0.

Flags at exit (set by `xor ebx, ebx`):

1. Carry Flag remains 0 because `xor` always clears CF.
2. Overflow Flag remains 0 because `xor` always clears OF.
3. Zero Flag is set (1) because the result is 0.
4. Sign Flag is 0 because the result 0 has its top bit clear.
5. Parity Flag is set (1) because 0 has an even number of 1-bits.

# 05.asm:

Shows that every 64-bit `push` uses 8 bytes, and how values are laid out in memory (little-endian).

1. `mov rax, 0x1122334455667788` and `mov rbx, 0x1234` load a large and a small value.
2. `push rax` stores 8 bytes. In memory the lowest address holds `88`, then `77 66 55 44 33 22 11`.
3. `push rbx` also uses 8 bytes, even though 0x1234 is small. It is padded with zeros: `34 12 00 00 00 00 00 00`.
4. `nop` is the point to inspect with `x/4gx $rsp` in GDB.
5. `pop rbx` then `pop rax` restore the original values in reverse order.
6. The program exits with code 0.

Flags at exit (set by `xor rdi, rdi`)

1. Carry Flag remains 0 because `xor` always clears CF.
2. Overflow Flag remains 0 because `xor` always clears OF.
3. Zero Flag is set (1) because the result is 0.
4. Sign Flag is 0 because the result 0 has its top bit clear.
5. Parity Flag is set (1) because 0 has an even number of 1-bits.

# 06.asm:

The 32-bit version of 05.asm. Each push uses 4 bytes and moves `esp` down by 4.

1. `mov eax, 0x11223344` and `mov ebx, 0x1234` load the values.
2. `push eax` stores `44 33 22 11` in memory (little-endian).
3. `push ebx` stores `34 12 00 00`, padded to 4 bytes.
4. `nop` is the inspection point (`x/2wx $esp` in GDB).
5. `pop ebx` then `pop eax` restore the values.
6. `mov eax, 1`, `xor ebx, ebx`, `int 0x80` exit with code 0.

Flags at exit (set by `xor ebx, ebx`):

1. Carry Flag remains 0 because `xor` always clears CF.
2. Overflow Flag remains 0 because `xor` always clears OF.
3. Zero Flag is set (1) because the result is 0.
4. Sign Flag is 0 because the result 0 has its top bit clear.
5. Parity Flag is set (1) because 0 has an even number of 1-bits.

