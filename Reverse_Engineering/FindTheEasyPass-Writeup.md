# Find The Easy Pass — HackTheBox Writeup

**Challenge:** Find The Easy Pass  
**Category:** Reverse Engineering  
**Difficulty:** Easy (20 pts)  
**Platform:** HackTheBox  
**Solver:** zham

**Flag:** `HTB{fortan!}`

---

## Description

"Find the password (say PASS) and enter the flag in the form HTB{PASS}"

A 32-bit Windows PE executable. The password is hardcoded inside a button click function. The goal is to find it using a disassembler.

---

## Prerequisites

- Kali Linux or Windows
- Tools: `file`, `wine` (to run on Linux), `IDA Free` or `Ghidra`
- ZIP password: `hackthebox`

---

## Step 1: Identify the Binary

```bash
unzip EasyPass.zip
# password: hackthebox

file EasyPass.exe
```

**Result:**
```
EasyPass.exe: PE32 executable (GUI) Intel 80386, for MS Windows, 8 sections
```

**What we know:**
- 32-bit Windows PE executable (GUI app)
- Has 8 sections

---

## Step 2: Run the Binary

On Windows — double click it. On Linux:

```bash
wine EasyPass.exe
```

A GUI window appears with a **password input field** and a **Check Password** button. Try anything — it says wrong.

---

## Step 3: Open in IDA / Ghidra

Open `EasyPass.exe` in **IDA Free** (choose 32-bit / Intel 80386 options).

Navigate through the functions list. Scroll down to find a function called **ButtonClick** (the function triggered when you press the Check Password button).

---

## Step 4: Analyze ButtonClick

Inside the ButtonClick function, look at the assembly:

```asm
mov eax, "fortan"
mov edx, "!"
; ... compare input with combined string
; if match → "Good job, Congratulations!"
; if no match → wrong password
```

Two registers (`eax` and `edx`) hold string values. Combining them gives: **`fortan!`**

Hovering over address `004540E1` confirms the `!` character. The full password is assembled from these two string parts.

---

## Step 5: Confirm the Password

Enter `fortan!` in the password field → **"Good job, Congratulations!"** appears.

Submit as: `HTB{fortan!}`

✅ **Flag: `HTB{fortan!}`**

---

## Attack Chain Summary

```
file → PE32 Windows executable
   ↓
wine EasyPass.exe → GUI with password field
   ↓
IDA → find ButtonClick function
   ↓
eax = "fortan", edx = "!" → combined = "fortan!"
   ↓
Enter password → Correct! → HTB{fortan!}
```

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `file` | Identify binary type |
| `wine` | Run Windows exe on Linux |
| IDA Free / Ghidra | Disassemble and find hardcoded password |

---

## Key Takeaways

1. **GUI executables still have reversible logic** — the button click handler contains the password comparison
2. **Hardcoded passwords are split across registers** — always combine `eax` + `edx` when you see two string values near a comparison
3. **IDA's function list is your map** — scroll through functions to find meaningful names like `ButtonClick`
4. **Hover over addresses in IDA** — memory addresses often show the character/string stored there
