# Bombs Landed — HackTheBox Writeup

**Challenge:** Bombs Landed  
**Category:** Reverse Engineering  
**Difficulty:** Medium (50 pts)  
**Platform:** HackTheBox  
**Solver:** zham

**Flag:** `HTB{younevergoingtofindme}`

---

## Description

"Can you find the password?"

A 32-bit ELF binary with anti-debug protection and a custom strcmp function. The goal is to bypass the conditions, step through the binary, and recover the hidden password from the stack.

---

## Prerequisites

- Kali Linux
- Tools: `file`, `checksec`, `strings`, `r2` (radare2), `gdb` + pwndbg or `edb`

---

## Step 1: Identify the Binary

```bash
unzip BombsLanded.zip
file BombsLanded
```

**Result:**
```
BombsLanded: ELF 32-bit LSB executable, Intel 80386, dynamically linked, no section headers
```

**What we know:**
- 32-bit ELF binary
- No section headers — makes analysis harder
- Dynamically linked

Check security:

```bash
checksec BombsLanded
```

**Result:**
```
Arch:     i386-32-little
RELRO:    No RELRO
Stack:    No canary found
NX:       NX disabled
PIE:      No PIE (0x8048000)
RWX:      Has RWX segments
```

**Good news:** No PIE means fixed addresses — breakpoints won't move. No NX means the stack is executable. This makes debugging easier.

---

## Step 2: Run the Binary

```bash
./BombsLanded
```

**Result:**
```
Bad luck dude.
```

Try with arguments:

```bash
./BombsLanded 1
./BombsLanded 1 2
./BombsLanded 1 2 3
```

**Result with 3+ arguments:**
```
input password:
```

The binary needs **more than 3 arguments** (argc > 3) to even show the password prompt. This is the first hidden condition.

---

## Step 3: Look for Strings

```bash
strings BombsLanded
```

**Interesting output:**
```
Bad luck dude.
Input Password:
Correct!
```

Strings gives us hints but not the password itself.

---

## Step 4: Load in Radare2

```bash
r2 -A BombsLanded
```

Inside radare2:

```
pdf @ main
```

**Key observations from disassembly:**

1. **First condition** — checks if `argc > 3` using a `JA` (Jump if Above) instruction
2. **Second condition** — checks something else using a `JG` (Jump if Greater) instruction
3. **Password prompt** — only shown after bypassing both conditions
4. **Custom strcmp** — the password comparison uses XOR with key `0xA` before comparing

---

## Step 5: Bypass Conditions with GDB + pwndbg

Load the binary in GDB:

```bash
gdb ./BombsLanded
```

Run with enough arguments to trigger condition 1:

```
pwndbg> run arg1 arg2 arg3 arg4
```

**Bypass Condition 1 (JA — argc check):**
- Set a breakpoint at the `JA` instruction address
- When hit, flip the **Zero Flag (ZF)** from 0 to 1:

```
pwndbg> set $eflags |= (1 << 6)
```

**Bypass Condition 2 (JG — second check):**
- Set a breakpoint at the `JG` instruction
- Flip the **Sign Flag (SF)** to 0:

```
pwndbg> set $eflags &= ~(1 << 7)
```

After bypassing both — the `Input Password:` prompt appears.

---

## Step 6: Find the Password on the Stack

Type any input when prompted (e.g., `test`). Continue stepping through the binary.

Set a breakpoint just before the `strncmp` call — this is where the real password is compared against your input.

```
pwndbg> break *<strncmp_address>
pwndbg> continue
```

Step through the loop repeatedly. Watch the stack — the **correct password is being built character by character** in memory as the loop runs.

Keep pressing `continue` until the loop ends. Read the string that appeared on the stack — that is the password.

---

## Step 7: Confirm the Password

Run the binary properly with 4+ arguments and enter the recovered password:

```bash
./BombsLanded arg1 arg2 arg3 arg4
```

```
Input Password: <recovered_password>
Correct!
```

Submit as: `HTB{younevergoingtofindme}`

---

## Attack Chain Summary

```
file → 32-bit ELF, no section headers, no PIE
   ↓
./BombsLanded → "Bad luck dude"
   ↓
./BombsLanded 1 2 3 4 → triggers password prompt (argc > 3)
   ↓
r2 / gdb → find two conditions: JA (argc) and JG (second check)
   ↓
Flip ZF → bypass condition 1
Flip SF → bypass condition 2
   ↓
Enter dummy input → step through loop → password built on stack
   ↓
Read password from stack → submit as HTB{password}
```

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `file` | Identify binary type |
| `checksec` | Check binary security features |
| `strings` | Extract readable strings |
| `radare2` | Disassemble and analyze binary |
| `gdb` + pwndbg | Debug, set breakpoints, flip CPU flags |
| `edb` | Alternative debugger (GUI-based) |

---

## Key Takeaways

1. **Always check argc** — many binaries hide functionality behind argument count checks
2. **No PIE = fixed addresses** — breakpoints don't change between runs, making debugging easier
3. **Flip CPU flags to bypass conditions** — ZF and SF control conditional jumps; flipping them redirects execution flow
4. **Watch the stack during loops** — passwords and keys are often built incrementally in memory
5. **Custom strcmp = XOR comparison** — when a binary uses XOR before comparing, the key (here `0xA`) can be used to recover the original value
6. **`strings` is a hint, not the answer** — always follow up with a debugger to see runtime behavior
