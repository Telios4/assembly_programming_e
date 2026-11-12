# 07_stack — How the Stack Works in x86 Assembly

Six NASM programs (Linux) that show how the stack behaves: PUSH/POP, CALL/RET, stack frames, and 32-bit vs 64-bit differences. Each program was assembled, run, and stepped through in GDB, and the RSP/ESP values and memory dumps below come from those runs.

| File | Topic | Mode |
|------|-------|------|
| 01.asm | Basic PUSH and POP (LIFO) | 64-bit |
| 02.asm | Stack frame with local variables | 64-bit |
| 03.asm | CALL and RET without a frame | 64-bit |
| 04.asm | Stack frame with local variables | 32-bit |
| 05.asm | Push sizes and little-endian layout | 64-bit |
| 06.asm | Push sizes and little-endian layout | 32-bit |

## Notes that apply to every program

- The stack grows **downwards**: `push` decreases RSP/ESP, `pop` increases it.
- At `_start` the kernel has already built a stack. RSP/ESP points at `argc` (value `1` in my runs). Each table below calls this starting value **S**.
- Addresses such as `0x7fffffffea20` (64-bit) and `0xffffdb30` (32-bit) are from my GDB runs. Yours will differ because of ASLR, but the differences between addresses are identical.
- To watch it yourself: `gdb ./NN`, then `break _start`, `run`, step with `si`, and inspect with `x/4gx $rsp` (64-bit) or `x/4wx $esp` (32-bit).
- 64-bit files build with `nasm -f elf64 NN.asm -o NN.o && ld NN.o -o NN`. 32-bit files (04, 06) build with `nasm -f elf32 NN.asm -o NN.o && ld -m elf_i386 NN.o -o NN`.

---

## 01.asm — Basic PUSH and POP (64-bit)

### Goal
Show that the stack is **LIFO** (last in, first out): values pushed in one order come back in the reverse order, and `RSP` moves by 8 bytes on every `push`/`pop`.

### Build and run
```bash
nasm -f elf64 01.asm -o 01.o
ld 01.o -o 01
./01
echo $?        # 0
```

### How the stack is used, from start to end

When execution begins at `_start`, the kernel has already built a stack for the process. `RSP` points at the first item on it (`argc`, which was `1` in my run). I treat that starting value as **S** (`0x7fffffffea20` under GDB; yours will differ because of ASLR, but the *differences* are identical). The stack grows **downwards**, so a push gives a *lower* address.

| # | Instruction | RSP after | What happens on the stack |
|---|-------------|-----------|---------------------------|
| 1 | `mov rax, 100` | S | Registers only. Stack untouched. |
| 2 | `mov rbx, 200` | S | Registers only. Stack untouched. |
| 3 | `push rax` | S − 8 (`...ea18`) | RSP is decremented by 8, then `100` (0x64) is written at `[RSP]`. |
| 4 | `push rbx` | S − 16 (`...ea10`) | RSP decremented by 8 again, `200` (0xc8) written at `[RSP]`. It is now the **top**. |
| 5 | `nop` | S − 16 | Breakpoint marker; stack holds two values. |
| 6 | `pop rcx` | S − 8 (`...ea18`) | The top value (`200`) is read into `rcx`, then RSP is incremented by 8. |
| 7 | `pop rdx` | S (`...ea20`) | The next value (`100`) is read into `rdx`, RSP incremented by 8. RSP is back to where it started. |
| 8 | `nop` | S | Marker; stack is balanced. |
| 9 | `mov rax, 60` / `xor rdi, rdi` / `syscall` | S | `sys_exit(0)`. No stack use. |

Stack picture at the first `nop` (after both pushes), highest address at the top:

```
address          contents
...ea20  (S)     argc  (belongs to the kernel, untouched)
...ea18          0x64  (100, pushed first)
...ea10  <- RSP  0xc8  (200, pushed last = top)
```

### Observations from running it
- `push` = "RSP -= 8, then store"; `pop` = "load, then RSP += 8". Confirmed: RSP went `ea20 -> ea18 -> ea10` on the pushes and `ea10 -> ea18 -> ea20` on the pops.
- GDB at the first `nop` (`x/2gx $rsp`) showed `0x00000000000000c8  0x0000000000000064`: the lower address (top of stack) holds 200, the one above it holds 100.
- Final registers: `rcx = 200`, `rdx = 100`. So `rcx` got `rbx`'s value and `rdx` got `rax`'s value: **the order is reversed** because of LIFO.
- `rax` and `rbx` were not changed by the push (push copies, it does not move). `rax` only changed later when it was reused for the syscall number (60).
- RSP ends exactly equal to its starting value: every push was matched by a pop, so the stack is **balanced**.
- Exit code is `0`.

---

## 02.asm — Function stack frame with local variables (64-bit)

### Goal
Show how a function builds and tears down a **stack frame** using `RBP` as a fixed anchor and `RSP` as the moving top, and how `call`/`ret` save and restore the return address.

### Build and run
```bash
nasm -f elf64 02.asm -o 02.o
ld 02.o -o 02
./02
echo $?        # 0
```

### How the stack is used, from start to end

Let **S** be `RSP` when `_start` begins (`0x7fffffffea20` in my GDB run; it points at `argc`). `RBP` is `0` at program start.

| # | Instruction | RSP after | RBP after | What happens on the stack |
|---|-------------|-----------|-----------|---------------------------|
| 1 | `call work` | S − 8 (`ea18`) | 0 | Pushes the **return address** `0x401005` (the instruction after the call) and jumps to `work`. |
| 2 | `push rbp` | S − 16 (`ea10`) | 0 | Saves the caller's `RBP` (here `0`) on the stack. |
| 3 | `mov rbp, rsp` | S − 16 (`ea10`) | `ea10` | No stack change. `RBP` now marks the base of this frame. |
| 4 | `sub rsp, 16` | S − 32 (`ea00`) | `ea10` | **Reserves** 16 bytes for locals just by moving RSP down. Nothing is written yet. |
| 5 | `mov qword [rbp-8], 100` | `ea00` | `ea10` | Writes 100 at `ea08`. RSP does **not** change (this is a plain store, not a push). |
| 6 | `mov qword [rbp-16], 200` | `ea00` | `ea10` | Writes 200 at `ea00`. |
| 7 | `mov rax, [rbp-8]` | `ea00` | `ea10` | Reads local #1 back: `rax = 100`. |
| 8 | `mov rbx, [rbp-16]` | `ea00` | `ea10` | Reads local #2 back: `rbx = 200`. |
| 9 | `nop` | `ea00` | `ea10` | Marker: frame is fully built. |
| 10 | `mov rsp, rbp` | `ea10` | `ea10` | **Frees the locals** by moving RSP back up to the frame base. |
| 11 | `pop rbp` | `ea18` | `0` | Restores the caller's `RBP` from the stack. |
| 12 | `ret` | `ea20` (S) | 0 | Pops `0x401005` into `RIP`; execution continues in `_start`. |
| 13 | `mov rax, 60` / `xor rdi, rdi` / `syscall` | S | 0 | `sys_exit(0)`. |

Stack picture at the `nop` (step 9):

```
address          contents
...ea20  (S)     argc (kernel's, untouched)
...ea18          0x401005   <- return address pushed by CALL
...ea10  <- RBP  0x0        <- saved caller RBP
...ea08          100        <- [rbp-8]  local #1
...ea00  <- RSP  200        <- [rbp-16] local #2
```

### Observations from running it
- GDB `x/4gx $rsp` at the `nop` printed (low to high address): `0xc8`, `0x64`, `0x0`, `0x401005`. That is exactly the picture above: 200, 100, saved RBP, return address.
- Locals live **below** `RBP` (negative offsets) and the saved RBP and return address live **at and above** it. That is why `[rbp-8]` and `[rbp-16]` are locals.
- Only `call`, `push`, `pop` and `ret` change RSP by themselves. The locals were stored with `mov` and did not move RSP; the space was claimed earlier with `sub rsp, 16`.
- After `pop rbp`, `RBP` became `0` again (the value the caller had). After `ret`, `RSP` equals S, so the stack is balanced.
- Total stack depth used: 32 bytes (return address 8 + saved RBP 8 + locals 16).
- Final registers: `rax = 100` and `rbx = 200` carried out of the function (then `rax` was reused for 60). Exit code `0`.

---

## 03.asm — CALL and RET with no stack frame (64-bit)

### Goal
Show the minimum the stack does for a function call: `call` saves the return address, `ret` takes it back. This function has no frame, no locals and no `push`/`pop` of its own.

### Build and run
```bash
nasm -f elf64 03.asm -o 03.o
ld 03.o -o 03
./03
echo $?        # 0
```

### How the stack is used, from start to end

Let **S** be `RSP` at `_start` (`0x7fffffffea20` in my GDB run).

| # | Instruction | RSP after | What happens on the stack |
|---|-------------|-----------|---------------------------|
| 1 | `mov rax, 10` | S | Registers only. |
| 2 | `call calculate` | S − 8 (`ea18`) | Pushes the **return address** `0x40100a` (the address of `after_call`, the next instruction) and jumps to `calculate`. |
| 3 | `add rax, 5` | S − 8 | `rax = 15`. Stack unchanged. |
| 4 | `ret` | S (`ea20`) | Pops the 8 bytes at `[RSP]` into `RIP` (so it jumps to `0x40100a`) and adds 8 to RSP. |
| 5 | `nop` (at `after_call`) | S | Execution is back in `_start`; `rax = 15`. |
| 6 | `mov rax, 60` / `xor rdi, rdi` / `syscall` | S | `sys_exit(0)`. |

Stack picture while inside `calculate` (before `ret`):

```
address          contents
...ea20  (S)     argc (kernel's, untouched)
...ea18  <- RSP  0x000000000040100a   <- return address placed by CALL
```

### Observations from running it
- In GDB, stopped at `ret`, `x/1gx $rsp` showed `0x000000000040100a`, and `0x40100a` is exactly the address of the `after_call` label. So `ret` is effectively `pop rip`.
- RSP went `ea20 -> ea18` on `call` and `ea18 -> ea20` on `ret`: one 8-byte push and one 8-byte pop, balanced.
- `ret` does not know where to go; it blindly trusts the top of the stack. That is why a function must leave RSP exactly where it found it before executing `ret` (here it does, because it never touches the stack).
- `rax` went `10 -> 15`, confirming the call ran and returned to the instruction right after `call`.
- Exit code `0`.

---

## 04.asm — Function stack frame with local variables (32-bit)

### Goal
The 32-bit version of the stack-frame program. Same idea, but with `ESP`/`EBP`, **4-byte** stack slots, and the `int 0x80` system call.

### Build and run
```bash
nasm -f elf32 04.asm -o 04.o
ld -m elf_i386 04.o -o 04
./04
echo $?        # 0
```
(`-f elf32` and `-m elf_i386` are required because the code is `BITS 32`.)

### How the stack is used, from start to end

Let **S** be `ESP` at `_start` (`0xffffdb30` in my GDB run; it points at `argc`). `EBP` is `0` at program start.

| # | Instruction | ESP after | EBP after | What happens on the stack |
|---|-------------|-----------|-----------|---------------------------|
| 1 | `call work` | S − 4 (`db2c`) | 0 | Pushes the 4-byte **return address** `0x08049005` and jumps to `work`. |
| 2 | `push ebp` | S − 8 (`db28`) | 0 | Saves the caller's `EBP` (`0`). |
| 3 | `mov ebp, esp` | `db28` | `db28` | No stack change. EBP is now the frame base. |
| 4 | `sub esp, 8` | S − 16 (`db20`) | `db28` | Reserves 8 bytes for two 4-byte locals. |
| 5 | `mov dword [ebp-4], 100` | `db20` | `db28` | Writes 100 at `db24`. ESP unchanged. |
| 6 | `mov dword [ebp-8], 200` | `db20` | `db28` | Writes 200 at `db20`. |
| 7 | `mov eax, [ebp-4]` | `db20` | `db28` | `eax = 100`. |
| 8 | `mov ebx, [ebp-8]` | `db20` | `db28` | `ebx = 200`. |
| 9 | `nop` | `db20` | `db28` | Marker: frame fully built. |
| 10 | `mov esp, ebp` | `db28` | `db28` | Frees the locals. |
| 11 | `pop ebp` | `db2c` | 0 | Restores the caller's EBP. |
| 12 | `ret` | `db30` (S) | 0 | Pops `0x08049005` into `EIP`; back in `_start`. |
| 13 | `mov eax, 1` / `xor ebx, ebx` / `int 0x80` | S | 0 | `sys_exit(0)` (syscall number 1 in `eax`, status in `ebx`). |

Stack picture at the `nop`:

```
address          contents
...db30  (S)     argc (kernel's, untouched)
...db2c          0x08049005  <- return address pushed by CALL
...db28  <- EBP  0x00000000  <- saved caller EBP
...db24          100         <- [ebp-4]  local #1
...db20  <- ESP  200         <- [ebp-8]  local #2
```

### Observations from running it
- GDB `x/4wx $esp` at the `nop` printed (low to high): `0xc8`, `0x64`, `0x0`, `0x08049005`, matching the picture.
- Everything is **half the size** of 02.asm: ESP moves in steps of 4 (not 8), the return address is 4 bytes, and the frame uses 16 bytes in total (4 + 4 + 8) instead of 32.
- The frame layout and the offsets (`[ebp-4]`, `[ebp-8]`) follow the same rule: locals below EBP, saved EBP and return address at/above it.
- ESP returns to S after `ret`: balanced stack.
- The exit uses `int 0x80` with `eax = 1`, not `syscall` with `rax = 60`.
- Exit code `0`.

---

## 05.asm — Pushing values of different sizes, little-endian layout (64-bit)

### Goal
Show that in 64-bit mode every `push` uses a full **8-byte** slot, even for a small value, and show how the bytes are laid out in memory (**little-endian**).

### Build and run
```bash
nasm -f elf64 05.asm -o 05.o
ld 05.o -o 05
./05
echo $?        # 0
```

### How the stack is used, from start to end

Let **S** be `RSP` at `_start` (`0x7fffffffea20` in my GDB run).

| # | Instruction | RSP after | What happens on the stack |
|---|-------------|-----------|---------------------------|
| 1 | `mov rax, 0x1122334455667788` | S | Registers only. |
| 2 | `mov rbx, 0x1234` | S | Registers only. |
| 3 | `push rax` | S − 8 (`ea18`) | RSP -= 8, then all 8 bytes of `rax` are stored at `[RSP]`. |
| 4 | `push rbx` | S − 16 (`ea10`) | RSP -= 8, then all 8 bytes of `rbx` are stored. `0x1234` is small but still takes 8 bytes. |
| 5 | `nop` | S − 16 | Marker: inspect the stack here. |
| 6 | `pop rbx` | S − 8 (`ea18`) | Loads the top 8 bytes into `rbx` (`0x1234`), RSP += 8. |
| 7 | `pop rax` | S (`ea20`) | Loads the next 8 bytes into `rax` (`0x1122334455667788`), RSP += 8. |
| 8 | `mov rax, 60` / `xor rdi, rdi` / `syscall` | S | `sys_exit(0)`. |

Raw bytes at the `nop` (`x/16xb $rsp`, lowest address first):

```
...ea10 (RSP)  34 12 00 00 00 00 00 00      <- rbx = 0x1234, padded to 8 bytes
...ea18        88 77 66 55 44 33 22 11      <- rax = 0x1122334455667788
```

### Observations from running it
- **Little-endian:** the *least* significant byte is stored at the *lowest* address. So `0x1122334455667788` appears in memory as `88 77 66 55 44 33 22 11`, and `0x1234` as `34 12 ...`. GDB's `x/2gx $rsp` shows the 64-bit values re-assembled: `0x0000000000001234  0x1122334455667788`.
- `push rbx` still moved RSP by exactly 8 (`ea18 -> ea10`) even though the value fits in 2 bytes. The remaining 6 bytes are zero.
- Pops come back in reverse: `rbx` first (top of stack), then `rax`. Since each register was popped into itself, both end with their original values, and the round trip through memory lost nothing.
- RSP ends equal to S: balanced stack.
- Exit code `0`.

---

## 06.asm — Pushing values in 32-bit mode, little-endian layout

### Goal
The 32-bit counterpart of 05.asm: every `push` uses a **4-byte** slot and moves `ESP` by 4, with the same little-endian byte layout.

### Build and run
```bash
nasm -f elf32 06.asm -o 06.o
ld -m elf_i386 06.o -o 06
./06
echo $?        # 0
```

### How the stack is used, from start to end

Let **S** be `ESP` at `_start` (`0xffffdb30` in my GDB run).

| # | Instruction | ESP after | What happens on the stack |
|---|-------------|-----------|---------------------------|
| 1 | `mov eax, 0x11223344` | S | Registers only. |
| 2 | `mov ebx, 0x1234` | S | Registers only. |
| 3 | `push eax` | S − 4 (`db2c`) | ESP -= 4, then the 4 bytes of `eax` are stored at `[ESP]`. |
| 4 | `push ebx` | S − 8 (`db28`) | ESP -= 4, then the 4 bytes of `ebx` are stored. `0x1234` is padded to 4 bytes. |
| 5 | `nop` | S − 8 | Marker: inspect the stack here. |
| 6 | `pop ebx` | S − 4 (`db2c`) | Loads the top 4 bytes into `ebx` (`0x1234`), ESP += 4. |
| 7 | `pop eax` | S (`db30`) | Loads the next 4 bytes into `eax` (`0x11223344`), ESP += 4. |
| 8 | `mov eax, 1` / `xor ebx, ebx` / `int 0x80` | S | `sys_exit(0)` through the 32-bit system call interface. |

Raw bytes at the `nop` (`x/8xb $esp`, lowest address first):

```
...db28 (ESP)  34 12 00 00      <- ebx = 0x1234, padded to 4 bytes
...db2c        44 33 22 11      <- eax = 0x11223344
```

### Observations from running it
- **Little-endian again:** `0x11223344` is stored as `44 33 22 11`. GDB's `x/2wx $esp` printed `0x00001234  0x11223344`, i.e. the top of the stack (lowest address) is `ebx`'s value.
- ESP moved by **4** per push/pop (`db30 -> db2c -> db28 -> db2c -> db30`), versus 8 in 05.asm. That is the only real difference between 32-bit and 64-bit stack behaviour here.
- LIFO order again: `ebx` was pushed last, so it is popped first. Because the pops mirror the pushes, both registers get back their original values.
- ESP ends equal to S: balanced stack.
- `eax` becomes `1` and `ebx` becomes `0` just before `int 0x80`, because the program reuses them as the syscall number and exit status.
- Exit code `0`.

---
