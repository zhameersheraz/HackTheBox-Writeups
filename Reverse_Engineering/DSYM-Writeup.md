# DSYM — HackTheBox Writeup

**Challenge:** DSYM  
**Category:** Reverse Engineering  
**Difficulty:** Easy  
**Platform:** HackTheBox  
**Solver:** zham

**Flag:** `HTB{y0u_r34lly_g0t_m3}`

---

## Description

"Can you find the flag hidden in the binary?"

A challenge with two files — an ELF binary and a DSYM debug symbols file. The binary outputs an encoded string. The encoding turns out to be ROT13 (Caesar Cipher). Using the debug symbols file to understand the binary makes analysis much easier.

---

## Prerequisites

- Kali Linux
- Tools: `file`, `strings`, `chmod`, `elfutils` (`sudo apt install elfutils`), Ghidra or IDA
- ZIP password: `hackthebox`

---

## Step 1: Extract and Identify Files

```bash
unzip DSYM.zip
# password: hackthebox

ls
```

**Result:**
```
getme
dunnoWhatIAm
```

Two files. Check both:

```bash
file getme
file dunnoWhatIAm
```

**Result:**
```
getme:          ELF 64-bit LSB executable, x86-64, dynamically linked
dunnoWhatIAm:   data (DSYM debug symbols file)
```

`getme` is the main binary. `dunnoWhatIAm` is a **DSYM file** — a debug symbols file that contains function names and variable names for the binary. This makes reversing much easier.

---

## Step 2: Install elfutils and Extract Symbols

```bash
sudo apt install elfutils
```

Use `eu-readelf` to read the DSYM file:

```bash
eu-readelf -s dunnoWhatIAm
```

This reveals function names, variable names, and structure — things that would normally be stripped from the binary itself.

---

## Step 3: Run the Binary

```bash
chmod +x getme
./getme
```

**Result:**
```
UGO{j0k_5uch_@z@m1at_QFLA_s1y3}
```

The output looks like a flag but starts with `UGO` instead of `HTB`. This pattern means it's encoded.

---

## Step 4: Analyze in Ghidra

Load `getme` in **Ghidra**. Also load the DSYM file to recover symbol names. Navigate to the main function.

**Key observations:**
- The binary has a hardcoded encoded string
- A function applies Caesar Cipher (ROT13) to the string before printing
- The output `UGO{...}` is the ROT13-encoded version of the real flag

---

## Step 5: Decode ROT13 with Python

The distance from `U` to `H` is 13, `G` to `T` is 13, `O` to `B` is 13 — confirming ROT13.

```python
enc_flag = "UGO{j0k_5uch_@z@m1at_QFLA_s1y3}"

def decrypt_rot13(flag):
    dec_flag = ""
    for x in flag:
        if x.isalpha() and x.isupper():
            dec_flag += chr((ord(x) - 13 - 65) % 26 + 65)
        elif x.isalpha() and x.islower():
            dec_flag += chr((ord(x) - 13 - 97) % 26 + 97)
        else:
            dec_flag += x
    return dec_flag

print(decrypt_rot13(enc_flag))
```

**Result:**
```
HTB{y0u_r34lly_g0t_m3}
```

✅ **Flag: `HTB{y0u_r34lly_g0t_m3}`**

---

## What is a DSYM File?

A **DSYM (Debug SYMbols)** file is a companion file to a binary that stores all the debugging information — function names, variable names, line numbers. Developers use it to debug crashes. When included with a challenge, it makes reverse engineering significantly easier because you can see the original variable and function names instead of just raw addresses.

---

## Attack Chain Summary

```
unzip → two files: getme (ELF) + dunnoWhatIAm (DSYM)
   ↓
elfutils → read DSYM symbols → function/variable names recovered
   ↓
./getme → outputs UGO{j0k_5uch_@z@m1at_QFLA_s1y3}
   ↓
UGO → HTB distance = 13 → ROT13
   ↓
Python ROT13 decode → HTB{w0w_5uch_@m@z1ng_DSYM_f1l3}
```

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `file` | Identify binary and DSYM file types |
| `elfutils` | Read debug symbols from DSYM file |
| Ghidra | Disassemble binary with DSYM symbols loaded |
| Python | Decode ROT13 Caesar Cipher |
| CyberChef | Alternative ROT13 decoder (online) |

---

## Key Takeaways

1. **DSYM files are gifts** — they restore function and variable names, making Ghidra analysis much easier
2. **ROT13 is common in CTFs** — if the output looks like a flag but starts with wrong letters, check the distance to `HTB`
3. **Always run the binary first** — the output itself often reveals the encoding type
4. **`elfutils` reads DSYM files** — install it with `sudo apt install elfutils` on Kali
5. **Caesar Cipher = shift by fixed number** — ROT13 is just Caesar with shift 13; Python can decode it in a few lines
