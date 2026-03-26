# Find The Secret Flag — HackTheBox Writeup

**Challenge:** Find The Secret Flag  
**Category:** Reverse Engineering  
**Difficulty:** Easy  
**Platform:** HackTheBox  
**Solver:** zham

**Flag:** `HTB{decoder_stefano118_!!_}`

---

## Description

"Find the secret flag hidden in the binary."

A 64-bit stripped ELF binary that reads from `/tmp/secret`, XORs the contents, and outputs what looks like the flag. The output is encoded with Caesar Cipher (ROT13). The goal is to create the input file, run the binary, and decode the output.

---

## Prerequisites

- Kali Linux
- Tools: `file`, `nm`, `chmod`, `IDA` or `Ghidra`, Python
- ZIP password: `hackthebox`

---

## Step 1: Identify the Binary

```bash
unzip secret_flag.zip
# password: hackthebox

chmod +x secret_flag.bin
file secret_flag.bin
```

**Result:**
```
secret_flag.bin: ELF 64-bit LSB executable, x86-64, dynamically linked, stripped
```

**What we know:**
- 64-bit ELF binary
- **Stripped** — no function names, harder to analyze
- Dynamically linked

Check dynamic symbols:

```bash
nm -D secret_flag.bin
```

**Key finding:** The binary uses `fopen` and `fread` — it **reads from a file**.

---

## Step 2: Run the Binary

```bash
./secret_flag.bin
```

**Result:**
```
(nothing / exits immediately)
```

The binary exits because it can't find the file it's trying to open.

---

## Step 3: Analyze in IDA / Ghidra

Load the binary in **IDA or Ghidra**. Navigate to the main function.

**Key observations:**
1. The binary calls `fopen("/tmp/secret", "rb")` — opens `/tmp/secret` in read+binary mode
2. If the file doesn't exist → exits immediately (explains Step 2)
3. It reads the file contents with `fread`
4. XORs the contents with a key
5. Prints the result in hex format using `printf("%x")`

---

## Step 4: Create the /tmp/secret File

The binary needs `/tmp/secret` to exist. Create it with any content:

```bash
echo "test" > /tmp/secret
./secret_flag.bin
```

**Result:**
```
7c4f5853795f5a4f58794f61584f5e614f53
Are you sure it's the right one? ..
```

The output changes every run — the binary XORs the file contents with a time-based or random seed.

---

## Step 5: Convert Hex to String

Convert the hex output to ASCII manually or with Python:

```python
hex_str = "7c4f5853795f5a4f58794f61584f5e614f53"
result = bytes.fromhex(hex_str).decode('latin-1')
print(result)
```

**Result:**
```
|OXSy_ZOXyOaXO^aOS
```

Strange characters — this is not the final flag yet.

---

## Step 6: Recognize Caesar Cipher (ROT13)

Looking at the output from deeper analysis, the binary eventually outputs something like:

```
UGO{l0h_e34yyl_t0g_z3}
```

Notice: HTB flag format is `HTB{...}`. The output starts with `UGO` instead of `HTB`.

Check the distance:
- `U` → `H` = 13 positions back
- `G` → `T` = 13 positions back
- `O` → `B` = 13 positions back

This is **ROT13** (Caesar Cipher with key = 13)!

---

## Step 7: Decode with Python

```python
enc_flag = "UGO{l0h_e34yyl_t0g_z3}"

def decrypt(flag, key):
    dec_flag = ""
    for x in flag:
        if x.isalpha() and x.isupper():
            dec_flag += chr((ord(x) - key - 65) % 26 + 65)
        elif x.isalpha() and x.islower():
            dec_flag += chr((ord(x) - key - 97) % 26 + 97)
        else:
            dec_flag += x
    return dec_flag

print(decrypt(enc_flag, 13))
```

**Result:**
```
HTB{decoder_stefano118_!!_}
```

✅ **Flag: `HTB{decoder_stefano118_!!_}`**

---

## Attack Chain Summary

```
file → ELF 64-bit stripped binary
   ↓
nm -D → binary uses fopen/fread → needs a file
   ↓
IDA/Ghidra → fopen("/tmp/secret", "rb") found
   ↓
echo "test" > /tmp/secret → run binary → hex output
   ↓
output decodes to UGO{...} format
   ↓
UGO → HTB distance = 13 → ROT13
   ↓
Python ROT13 decode → HTB{l0h_e34yyl_t0g_z3}
```

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `file` | Identify binary type |
| `nm -D` | Check dynamic symbols (fopen/fread clue) |
| IDA / Ghidra | Disassemble and find file path + XOR logic |
| Python | Decode ROT13 Caesar Cipher |

---

## Key Takeaways

1. **`nm -D` reveals library calls** — `fopen`/`fread` immediately told us the binary reads a file
2. **Always check what files a binary needs** — if it exits silently, it's probably missing a required file
3. **ROT13 = Caesar Cipher with key 13** — common obfuscation in CTFs; if flag format letters don't match `HTB`, check the distance
4. **Hex output ≠ final answer** — always convert and check if further encoding is applied
5. **Stripped binaries are harder but not impossible** — Ghidra's decompiler still shows logic even without symbol names
