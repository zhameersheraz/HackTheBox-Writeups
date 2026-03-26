# Impossible Password — HackTheBox Writeup

**Challenge:** Impossible Password  
**Category:** Reverse Engineering  
**Difficulty:** Easy  
**Platform:** HackTheBox  
**Solver:** zham

**Flag:** `HTB{40b949f92b86b18}`

---

## Description

"Are you able to cheat me and get the flag?"

A 64-bit ELF binary with two password checks. The first password is hardcoded. The second is generated using XOR with key `9`. The goal is to either patch the binary to bypass the second check or reverse the XOR algorithm to decode the flag directly.

---

## Prerequisites

- Kali Linux
- Tools: `file`, `strings`, `r2` (radare2)

---

## Step 1: Identify the Binary

```bash
unzip impossible_password.zip
file impossible_password.bin
```

**Result:**
```
impossible_password.bin: ELF 64-bit LSB executable, x86-64, dynamically linked
```

---

## Step 2: Run the Binary

```bash
./impossible_password.bin
```

**Result:**
```
* 
```

It asks for input. Try anything:

```bash
* test
```

```
test
```

It exits. Now try the correct first password — found by checking strings:

```bash
strings impossible_password.bin
```

**Key string found:** `SuperSeKretKey`

```bash
* SuperSeKretKey
** 
```

A second prompt appears (`**`) — this is the second password check.

---

## Step 3: Analyze with Radare2

Open in radare2 with write mode enabled:

```bash
r2 -Aw impossible_password.bin
```

Analyze and disassemble main:

```
[0x00400750]> aaa
[0x00400750]> pdf @ main
```

**Key observations:**

1. First `strcmp` compares input with `SuperSeKretKey` — if wrong, exits
2. Second check calls a `mysterious()` function that generates a string using **XOR with key `9`**
3. A `jne` at address `0x00400968` jumps away if the second password is wrong — this is what we need to bypass

**The XOR obfuscated string in memory:**
```
A]Kr=9k0=0o0;k1?k81t
```
Hex: `41 5d 4b 72 3d 39 6b 30 3d 30 6f 30 3b 6b 31 3f 6b 38 31 74`

---

## Method 1: Patch the Binary (Easy Way)

Seek to the `jne` instruction and replace it with `nop` (no operation):

```
[0x00400750]> s 0x00400968
[0x00400968]> wa nop
```

Verify the patch:

```
[0x00400968]> pdf @ main
```

The `jne` is now `nop` — the jump no longer happens, so the flag function always runs.

Quit radare2:

```
[0x00400968]> q
```

Run the patched binary:

```bash
./impossible_password.bin
* SuperSeKretKey
** anything
HTB{40b949f92b86b18}
```

✅ **Flag: `HTB{40b949f92b86b18}`**

---

## Method 2: Reverse the XOR (Manual Decode)

The `mysterious()` function XORs each byte of the string with key `9`. We can reverse this ourselves:

```python
encoded = "A]Kr=9k0=0o0;k1?k81t"
key = 9
flag = ""
for c in encoded:
    flag += chr(ord(c) ^ key)
print(flag)
```

**Result:**
```
HTB{40b949f92b86b18}
```

Same flag without even running the binary!

---

## Attack Chain Summary

```
file → ELF 64-bit binary
   ↓
strings → finds "SuperSeKretKey" (first password)
   ↓
./binary → enter SuperSeKretKey → second prompt appears
   ↓
r2 disassembly → finds jne at 0x00400968 blocking flag output
   ↓
METHOD 1: patch jne → nop → run binary → enter any second input → flag
METHOD 2: XOR encoded string with key 9 → decode flag directly
```

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `file` | Identify binary type |
| `strings` | Find hardcoded first password |
| `radare2` | Disassemble binary and patch jne instruction |
| Python | Manually reverse XOR encoding |

---

## Key Takeaways

1. **`strings` reveals hardcoded passwords** — always run it first on any binary
2. **Patching binaries is valid RE technique** — replacing `jne` with `nop` bypasses conditions entirely
3. **XOR is reversible** — XOR with the same key twice returns the original value; knowing the key (`9`) lets you decode anything
4. **Two approaches to RE** — dynamic (patch and run) vs static (reverse the algorithm in Python); both are valid
5. **`r2 -Aw`** — always open radare2 with write mode (`-w`) if you plan to patch the binary
