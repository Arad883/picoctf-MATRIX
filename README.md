Solving the picoCTF "MATRIX" Challenge

A comprehensive guide to solving the picoCTF "MATRIX" reverse engineering challenge, including related "Matrix" challenges in the picoCTF series.

📌 Introduction

The MATRIX challenge from picoCTF (picoMini by redpwn, 2021) is one of the hardest Reverse Engineering challenges on the platform, worth 500 points. It involves reverse-engineering a binary that runs code through a custom virtual machine (VM) with its own instruction set. This guide covers the full solution process, plus related "Matrix" challenges in the picoCTF series.

🎯 Challenge Overview

MATRIX (picoCTF 2021 Mini-Competition by redpwn)

· Category: Reverse Engineering
· Points: 500
· Description: "Escape the matrix."
· Binary: Stripped ELF 64-bit executable
· Core mechanic: A custom stack-based virtual machine that interprets embedded bytecode encoding a navigable maze

When run, the binary prompts for directional input (u, d, l, r) and navigates through a maze structure. The goal is to find the correct sequence of movements to escape and receive the flag.

🔍 Step 1: Initial Analysis

Running the Binary

First, make the binary executable and run it to observe its behavior:

```bash
chmod +x matrix
./matrix
```

The binary prompts for directional input and reports whether you escaped the maze. This runtime behavior indicates it is not a simple flag-comparison validator but a full interactive interpreter.

What Doesn't Work

· Running strings: Prints noise and never the flag, which is assembled character by character at runtime
· Brute-forcing random sequences: The VM checks both position and a running health counter at each barrier. Reaching the exit with the wrong health causes it to jump past the flag routine

🧠 Step 2: Reverse-Engineering the VM

Finding the Dispatch Loop

Load the binary into Ghidra and trace into the main function. The core function (often named step() in clean decompilations) reads one byte from the bytecode array and branches on its value.

### VM Opcodes

The VM implements the following instruction set (operating on 2-byte words with a main and extra stack):

*   **`0x00`** (1 byte) — `nop` — No operation
*   **`0x01`** (1 byte) — `pop r1; ret` — Return from subroutine
*   **`0x10`** (1 byte) — `push [sp-2]` — Duplicate top of stack
*   **`0x11`** (1 byte) — `pop r1` — Pop from stack
*   **`0x12`** (1 byte) — `pop r1; add [sp-2], r1` — Addition
*   **`0x13`** (1 byte) — `pop r1; sub [sp-2], r1` — Subtraction
*   **`0x14`** (1 byte) — `xchg [sp-4], [sp-2]` — Swap top two stack values
*   **`0x20`** (1 byte) — `pop r1; push_e r1` — Push to extra stack
*   **`0x21`** (1 byte) — `pop_e r1; push r1` — Pop from extra stack
*   **`0x30`** (1 byte) — `pop r1; jmp r1` — Jump
*   **`0x31`** (1 byte) — `pop r1; pop r2; test r2, r2; jnz NEXT_INSTRUCTION; jmp r2` — Conditional jump
*   **`0x80`** (2 bytes) — `push NUMBER` — Push 1-byte signed number
*   **`0x81`** (3 bytes) — `push NUMBER` — Push 2-byte number
*   **`0xC0`** (1 byte) — `call getc; push r1` — Read character
*   **`0xC1`** (1 byte) — `pop r1; call putc` — Write character

The VM operates on words (2-byte values) and uses two stacks: a main stack and an "extra" stack for temporary storage.

🐍 Step 3: Building a Python Emulator

To visualize the maze and understand the bytecode, write a Python script that parses the bytecode and simulates VM execution:

```python
def disassemble_bytecode(bytecode):
    """Convert VM bytecode into readable instructions."""
    # Opcode mapping based on reverse-engineered VM
    OPCODES = {
        0x00: ("nop", 1),
        0x01: ("ret", 1),
        0x10: ("dup", 1),
        0x11: ("pop", 1),
        0x12: ("add", 1),
        0x13: ("sub", 1),
        0x14: ("swap", 1),
        0x20: ("push_e", 1),
        0x21: ("pop_e", 1),
        0x30: ("jmp", 1),
        0x31: ("jnz", 1),
        0x80: ("push8", 2),
        0x81: ("push16", 3),
        0xC0: ("getc", 1),
        0xC1: ("putc", 1),
    }
    
    instructions = []
    i = 0
    while i < len(bytecode):
        op = bytecode[i]
        if op in OPCODES:
            name, length = OPCODES[op]
            if length == 2:
                arg = bytecode[i+1]
                instructions.append(f"{name} {arg}")
            elif length == 3:
                arg = (bytecode[i+2] << 8) | bytecode[i+1]
                instructions.append(f"{name} {arg}")
            else:
                instructions.append(name)
            i += length
        else:
            instructions.append(f"UNKNOWN({op:02x})")
            i += 1
    return instructions
```

🗺️ Step 4: Solving the Maze

Understanding the Maze Structure

The bytecode encodes a maze where:

· Movement characters: u (up), d (down), l (left), r (right)
· Health counter: A running value that must be sufficient to survive barriers
· Exit condition: Reach the exit with the correct health to trigger the flag routine

Tracing the Path

By emulating the VM and logging state changes (position, health, and input consumed), you can reconstruct the maze layout and determine the correct path. The flag is assembled character by character at runtime, so you must feed the correct movement sequence to the binary to receive it.

Submitting the Solution

Once you have the correct movement sequence, pipe it into the binary:

```bash
echo "udllurrdd..." | ./matrix
```

If the path is correct, the binary will output the flag.

📚 Related picoCTF Matrix Challenges

Enter The Matrix (picoCTF 2017)

· Category: Binary Exploitation
· Points: 150
· Description: "The Matrix awaits you. Take the red pill and begin your journey."
· Vulnerability: An indexing bug in matrix operations: m->data[r * m->nrows + c] should be m->data[r * m->ncols + c]. This allows writing outside the allocated heap space.
· Exploitation: Overflow into another matrix struct to overwrite a GOT address with system, then trigger the exploit to get a shell.

Hints from the challenge:

· Try playing with different non-symmetrical matrices
· You can SSH into the machine running the challenge
· gdb gives incorrect libc offsets outside of gdb
· Use readelf -s /lib32/libc.so.6 to get function offsets
· Free is passed a pointer to data you control

Deeper Into The Matrix (picoCTF 2017)

· Category: Binary Exploitation
· Points: 200
· Solution: A follow-up to "Enter The Matrix" with a similar structure but different offsets and exploitation approach. Solution scripts are available on GitHub.

⚠️ Common Pitfalls

1. Assuming it's a simple password check: The binary is a VM interpreter, not a flag comparator. You must reverse-engineer the VM logic.
2. Ignoring the health counter: Reaching the exit with insufficient health causes the flag routine to be skipped.
3. Not using a disassembler: Manually reading raw bytecode is impractical. Write a Python disassembler as shown above.
4. Forgetting to handle both stacks: The VM uses a main stack and an extra stack. Ignoring the extra stack will lead to incorrect emulation.

📚 Further Reading

· picoCTF Solutions: MATRIX Walkthrough — Step-by-step guided solution
· DEV Community: picoCTF "MATRIX" Walkthrough — Extremely detailed analysis
· Pugachev.io: "MATRIX" by picoCTF — VM opcode reverse-engineering
· Medium: PICOCTF: MATRIX Writeup — Analysis in Thai with code snippets
· CTFtime: Enter The Matrix Writeup — picoCTF 2017 binary exploitation

---

License: MIT — free to use, modify, and distribute.
